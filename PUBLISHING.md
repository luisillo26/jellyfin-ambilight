# Publishing Guide

This plugin is packaged with [JPRM](https://github.com/oddstr13/jellyfin-plugin-repository-manager),
the tool used by the official Jellyfin plugin template. `build.yaml` is the single source of
truth for the plugin's identity, version and target ABI.

## Prerequisites

- Push access to the repository that serves `manifest.json`
  (currently `luisillo26/jellyfin-ambilight`)
- .NET 10 SDK for local builds
- Python 3 with `jprm` for local packaging (`pip install jprm`); CI installs it itself

## Why versions must have four parts

Jellyfin stores the assembly version (`2.5.0.0`) in `PluginInfo.Version` and looks up the
plugin image, the enable/disable switch and uninstall by that exact value. If the catalog
publishes `2.5.0` instead, `/Plugins/{id}/2.5.0.0/Image` and `/Plugins/{id}/2.5.0.0/Disable`
return 404: no icon, dead toggle, nothing in the server log.

JPRM reads `version` from `build.yaml`, rewrites `Version`, `AssemblyVersion` and
`FileVersion` in `Directory.Build.props` and the project, and writes the same four-part value
into the package `meta.json`, so the two can no longer disagree. Do not hand-edit version
numbers in `manifest.json`.

## Release process

1. **Note the changes**

   Add a new section to `CHANGELOG.md` and create `release-notes/RELEASE_NOTES_vX.Y.Z.md`.
   The release workflow uses that file as the release body and as the catalog changelog.

2. **Bump `build.yaml`** (optional but keeps local builds honest)

   ```yaml
   version: "3.1.0.0"
   ```

   The tag always wins at release time: the workflow passes the tag to JPRM as `--version`
   and normalises it to four parts, so `v3.1.0` and `v3.1.0.0` produce the same package.

3. **Commit and tag**

   ```bash
   git add .
   git commit -m "release: v3.1.0"
   git push origin master
   git tag v3.1.0
   git push origin v3.1.0
   ```

4. **What the workflow does** (`.github/workflows/release.yaml`)

   - builds and packages with JPRM (`ambilight_3.1.0.0.zip` + `meta.json`)
   - verifies `meta.json` reports the tag version and `build.yaml`'s `targetAbi`, and fails otherwise
   - creates the GitHub release with the zip and the release notes
   - prepends the new version to `manifest.json` and pushes it to `master`

## Building a package locally

```bash
dotnet restore && dotnet build --no-restore --configuration Release   # compile check
pip install jprm
mkdir -p artifacts
jprm plugin build . -o ./artifacts --version 3.0.0.0
```

The result is `artifacts/ambilight_3.0.0.0.zip` containing the DLL, `thumb.png` and
`meta.json` at the archive root. JPRM edits `Directory.Build.props` in place, so check
`git status` afterwards.

JPRM writes `"image"` into `meta.json`, but Jellyfin's manifest reader only reads
`"imagePath"`, so a locally built zip has no icon until a catalog refresh downloads it. The
release workflow injects `"imagePath": "thumb.png"` into `meta.json` before uploading, which
is why the released asset shows the icon even when installed manually.

## Compatibility rules

| Plugin build | targetAbi | Runs on |
|---|---|---|
| 3.x | `12.0.0.0` | Jellyfin 12.x |
| 2.5.0.0 | `10.10.0.0` | Jellyfin 10.10 / 10.11 |

Both stay in `manifest.json`: Jellyfin installs the newest version whose `targetAbi` is not
greater than the server version, so 10.x servers keep getting 2.5.0.0 while 12.x servers get 3.x.

## Verification

After publishing, check that the plugin is actually reachable:

```bash
curl -o /dev/null -w '%{http_code}\n' \
  http://<server>:8096/Plugins/b3f6b4c7-0a3d-4bd4-a7e3-c8d5a0a1e3f0/<version>/Image
```

`200` means the installed manifest version and the assembly version agree. `404` means they
do not — the icon and the enable/disable switch will not work until they do.
