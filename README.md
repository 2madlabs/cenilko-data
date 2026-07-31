# cenilko-data

Public distribution host for the offline data bundle used by **Cenilko**, an
offline-first Android/iOS app for the Slovenian real-estate market (GURS ETN
recorded sales and rentals since 2007, price map, area statistics, comparable-sales
estimator, cadastre lookup, mortgage tools, price alerts).

This repository holds almost no code. It used to be the download endpoint itself,
serving the artifacts as **GitHub Release assets**. Distribution now runs from a
**Cloudflare R2 bucket** on the custom domain `https://data.2madlabs.si`, so what
is left here is this documentation plus a local staging directory.

The artifacts themselves are produced elsewhere — by the Python (uv) pipeline in
the `cenilko-android` repository (`pipeline/`), which downloads the yearly ETN
exports from the GURS JGP API, cleans them, and builds the SQLite databases,
overlays and `manifest.json`.

## How distribution works

- One **Cloudflare R2 bucket**, `cenilko-data`, published anonymously on the
  custom domain `https://data.2madlabs.si`. Every object sits flat at the bucket
  root. Artifact names carry a content hash, so a rebuild writes a new object
  instead of replacing one; only `manifest.json` is overwritten in place.
- The apps have the manifest URL compiled in:
  `https://data.2madlabs.si/manifest.json` (`MANIFEST_URL` in
  `app/build.gradle.kts` on Android, the default source in
  `ArtifactManifest.swift` on iOS). Debug builds point at
  `http://127.0.0.1:8765/manifest.json` instead.
- Artifact `url` values in the manifest are **relative to the manifest's own
  location**, so the same bundle can be served from R2, any other object storage,
  or a local `http.server` without regenerating it. The move off GitHub Releases
  changed only the base URL; the manifest itself is untouched.
- `Cache-Control` is set per object at upload time. Content-hashed artifacts are
  `public, max-age=31536000, immutable`; `manifest.json` is
  `public, max-age=300, must-revalidate`, so a new bundle becomes visible within
  five minutes.
- Upload order matters: artifacts first, `manifest.json` last. A client must never
  fetch a manifest that references an object that is not published yet.

### The frozen GitHub release

The old rolling release under the tag `data` still exists and still serves its
last set of assets from
`https://github.com/2madlabs/cenilko-data/releases/download/data/manifest.json`.
It is frozen: CI no longer touches it, so the data behind it goes stale from the
cutover onwards. It is kept only so pre-migration internal-test builds, which have
that URL compiled in, keep syncing something until they are replaced. The
repository stays public for as long as that release has to be reachable. Once
those builds are gone, the release can be deleted and this repository can go
private or be archived.

## Repository layout

```
README.md      # this file
.gitignore
release/       # local staging copy of the generated bundle — NOT committed
```

`release/` is where the pipeline output is staged before being uploaded to the
bucket. It is listed in `.gitignore` and must never be committed: the base
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

This is a **deterrent, not access control**. The bucket stays anonymously
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

Both share a concurrency group so they can never write to the bucket at the same
time. Both run `pipeline/verify_release.py` as a hard gate before any upload: it
checks encrypted asset size and hash, performs an authenticated decryption, checks
the raw sha256 of locally built files, and re-verifies carried-forward entries
against what is actually published.

Uploads go through R2's S3-compatible API with the AWS CLI that is preinstalled on
the runners, against the account's `*.r2.cloudflarestorage.com` endpoint. Each job
uploads the rebuilt artifacts first, then `manifest.json`, then garbage-collects:
any object other than `manifest.json` that the new manifest does not reference and
that is older than 14 days is deleted. The grace period means a client that is
mid-sync against the previous manifest never hits a 404.

Publishing needs three secrets on the app repo: `CENILKO_DATA_KEY_B64` (the
artifact key) and `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` (an R2 API token
scoped to this bucket). The account id, bucket name and endpoint are plain values
in the workflows. `CENILKO_DATA_TOKEN`, the old contents:write token for this
repository, is no longer used.

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

- **Weekly job failed** — fix and re-run. The bucket still holds the last good
  pair; clients are unaffected.
- **Bad artifact published** — rebuild and re-run the workflow. It republishes to
  R2 in the usual order. Versions are content-derived, so clients pick up the
  replacement on their next sync, at most five minutes after the new
  `manifest.json` lands. The superseded objects stay reachable and are removed by
  the cleanup step once they are 14 days old.
- **Corrupt files on a device** — Settings → "Re-download data" force-fetches the
  pair; the existing database keeps serving until the new one opens.

## Data sources and licensing

All artifacts are derived from publicly available Slovenian open data — GURS (ETN,
cadastre, valuation records), DRSV (flood hazard, water-protection zones), the
Ministry of Culture eVRD heritage register, the energy performance certificate
register, and SURS municipality indicators. Attribution requirements, data-state
dates, and the full per-source licence list are documented in the main Cenilko
project and shown in the app itself.
