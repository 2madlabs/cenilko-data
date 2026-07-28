# cenilko-data

Public distribution host for the offline data bundle used by **Cenilko**, an
offline-first Android/iOS app for the Slovenian real-estate market (GURS ETN
recorded sales and rentals since 2007, price map, area statistics, comparable-sales
estimator, cadastre lookup, mortgage tools, price alerts).

This repository holds almost no code. Its job is to be the anonymous,
unauthenticated download endpoint the app talks to at runtime: the artifacts are
**GitHub Release assets**, not Git objects.

The artifacts themselves are produced elsewhere — by the Python (uv) pipeline in
the `cenilko-android` repository (`pipeline/`), which downloads the yearly ETN
exports from the GURS JGP API, cleans them, and builds the SQLite databases,
overlays and `manifest.json`.

## How distribution works

- One **rolling GitHub Release** under the tag `data`. The weekly and full data
  jobs overwrite its assets in place (`gh release upload --clobber`); the tag and
  the release are never recreated per build.
- The app's release build has the manifest URL compiled in:
  `https://github.com/2madlabs/cenilko-data/releases/download/data/manifest.json`
  (`MANIFEST_URL` in `app/build.gradle.kts`). Debug builds point at
  `http://127.0.0.1:8765/manifest.json` instead.
- Artifact `url` values in the manifest are **relative to the manifest's own
  location**, so the same bundle can be served from GitHub Releases, object
  storage, or a local `http.server` without regenerating it.
- The repository must stay **public**. The app downloads assets anonymously; the
  `data-full` workflow explicitly fails if visibility is anything else.
- Upload order matters: artifacts first, `manifest.json` last. A client must never
  fetch a manifest that references an asset that is not published yet.

## Repository layout

```
README.md      # this file
.gitignore
release/       # local staging copy of the generated bundle — NOT committed
```

`release/` is where the pipeline output is staged before being uploaded as release
assets. It is listed in `.gitignore` and must never be committed: the base
artifact alone is ~100 MB, past GitHub's per-file limit for Git objects.

## Bundle contents

| Asset | What it is |
| --- | --- |
| `manifest.json` | Index of the bundle: versions, sizes, hashes, encoding, minimum app version. |
| `etn-base-<year>-<sha12>.sqlite.zst.enc` | ETN deals from 2007 through the last closed year. Large (~100 MB compressed), rebuilt only by the yearly full job. |
| `etn-current-<year>-<sha12>.sqlite.zst.enc` | Current-year ETN deals, rebuilt weekly. Also carries the "cumulative" tables (energy certificates, seismic hazard, SURS municipality indicators) so they can be replaced without touching base. |
| `flood-<sha12>.pmtiles.enc` | DRSV flood-hazard vector tiles, three warning classes (frequent / rare / very rare). Rendered as a map overlay. |
| `terrain-<sha12>.bin.zst.enc` | Per-cell slope, aspect octant, and a yearly clear-sky insolation index derived from SRTM 1-arcsecond elevation. |
| `constraints-<sha12>.bin.zst.enc` | Coarse (~180 m cell) point lookup for cultural-heritage regimes, water-protection zones, and nearby recorded water/sewerage/electricity networks. Screening aid only. |

The ETN SQLite files share one schema (`posel`, `enota`, `obcina`, `naselje`, `ko`,
`prostori`, `raba`, `agg_obdobje`, `agg_grid`, `energy_cert`, `seismic`,
`demografija`, `obcina_kazalniki`, `meta`). The app opens `base` and `ATTACH`es
`current`, then unifies and deduplicates them through TEMP views.

Overlays (`flood`, `terrain`, `constraints`) are optional in the manifest and are
versioned independently by content hash — refreshing one does not force a schema bump.

## manifest.json

```json
{
  "schemaVersion": 8,
  "minAppVersionCode": 3,
  "current": {
    "coversFrom": "2026-01-01",
    "version": "etn-v8-current-2026-a26b1eca67c1",
    "url": "etn-current-2026-a26b1eca67c1.sqlite.zst.enc",
    "bytes": 5172331,
    "rawBytes": 17231872,
    "sha256": "c87cd848…",
    "assetSha256": "a26b1eca…",
    "encoding": "zstd+aes-256-gcm-chunked-v2"
  },
  "base": { "…": "same shape" },
  "flood": { "…": "same shape, encoding aes-256-gcm-chunked-v2" },
  "terrain": { "…": "same shape" }
}
```

Field meanings:

- `schemaVersion` — ETN SQLite schema generation. Must satisfy the client's
  `MIN_SUPPORTED_SCHEMA_VERSION..SUPPORTED_SCHEMA_VERSION` range
  (`EtnDatabase.kt`), and matches `ETN_SCHEMA_VERSION` in `pipeline/config.py`.
- `minAppVersionCode` — clients below this refuse the bundle. Raised to `3` when
  the encrypted CENK envelope was introduced; older builds can parse the manifest
  but cannot install encrypted artifacts.
- `version` — `etn-v{schema}-{kind}-{year}-{sha12}` for ETN files,
  `{name}-{sha12}` for overlays. Content-derived, so any rebuild changes the
  string and triggers a client re-download.
- `bytes` / `assetSha256` — size and hash of the encrypted asset as downloaded.
- `rawBytes` / `sha256` — size and hash of the plaintext payload, verified on the
  device after decrypt + decompress.
- `encoding` — `zstd+aes-256-gcm-chunked-v2`, or `aes-256-gcm-chunked-v2` for
  already-compressed payloads (PMTiles).
- `coversFrom` — earliest transaction date covered, shown in the app.

The snapshot above reflects the bundle staged in `release/`. The pipeline has since
moved on (schema 11 adds ownership share, descriptive deal/unit context, and SURS
municipality indicators; `constraints` was added as a fourth overlay), so treat the
local copy as a point-in-time staging directory, not the source of truth.

## Encryption (CENK v2)

Every asset is wrapped in a chunked AES-256-GCM envelope before upload
(`pipeline/artifact_crypto.py`):

- Header: magic `CENK`, format version `2`, chunk size, plaintext size, 8-byte
  nonce prefix (big-endian, 25 bytes total).
- Payload: 1 MiB records, each with its own GCM tag. The nonce is
  `prefix || chunkIndex`; the header plus index and size are the associated data.
  This lets the app authenticate and write one record at a time instead of
  buffering a 100 MB asset in memory.

This is a **deterrent, not access control**. The release stays anonymously
downloadable and the client necessarily ships the same 32-byte key, which a
determined person can recover from the APK. Its only purpose is to stop casual
inspection of the raw files in a browser.

The key is a 32-byte value, base64-encoded, read from `CENILKO_DATA_KEY_B64` (CI
secret) or, locally, from the ignored `pipeline/.data/release-key.b64`. The same
value is embedded into the app by Gradle. Never commit it. Rotating it means
rebuilding every artifact **and** shipping an app that contains the new key —
older installs cannot decrypt newly rotated data.

## Publishing

Publishing is driven by GitHub Actions in the `cenilko-android` repository, not
here:

- **`data-weekly`** (Mondays 04:17 UTC) — fetches the published manifest, refreshes
  the constraints overlay best-effort, rebuilds only the current-year ETN artifact,
  carries base/flood/terrain/constraints forward, verifies, republishes.
- **`data-full`** — yearly rebuild of everything, including base.

Both share a concurrency group so they can never publish to the rolling release at
the same time. Both run `pipeline/verify_release.py` as a hard gate before any
upload: it checks encrypted asset size and hash, performs an authenticated
decryption, checks the raw sha256 of locally built files, and re-verifies
carried-forward entries against what is actually published. Publishing needs two
secrets on the app repo: `CENILKO_DATA_TOKEN` (contents:write here) and
`CENILKO_DATA_KEY_B64`.

## Working with the bundle locally

Nothing to build in this repository. To produce and serve a bundle:

```sh
# in the cenilko-android repo
cd pipeline
uv run python main.py build --years 2024     # or: full / weekly
cd .data/out && python -m http.server 8765
adb reverse tcp:8765 tcp:8765                # debug builds read 127.0.0.1:8765
```

To inspect an already-published asset, decrypt it with `iter_decrypted()` from
`pipeline/artifact_crypto.py` (needs the release key), then `zstd -d`.

## Recovery

- **Weekly job failed** — fix and re-run. The release tag still holds the last good
  pair; clients are unaffected.
- **Bad artifact published** — rebuild and re-run the workflow. Versions are
  content-derived, so clients pick up the replacement on their next sync.
- **Corrupt files on a device** — Settings → "Re-download data" force-fetches the
  pair; the existing database keeps serving until the new one opens.

## Data sources and licensing

All artifacts are derived from publicly available Slovenian open data — GURS (ETN,
cadastre, valuation records), DRSV (flood hazard, water-protection zones), the
Ministry of Culture eVRD heritage register, the energy performance certificate
register, and SURS municipality indicators. Attribution requirements, data-state
dates, and the full per-source licence list are documented in the main Cenilko
project and shown in the app itself.
