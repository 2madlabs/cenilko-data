# cenilko-data

Public release-asset host for Cenilko's offline data bundle.

The generated files are staged locally under `release/` and uploaded to the
GitHub Release tagged `data`. The `release/` directory is intentionally ignored:
the artifacts are GitHub Release assets, not Git objects.

Upload artifacts first and `manifest.json` last so clients never receive a
manifest that references an asset that is not available yet.
