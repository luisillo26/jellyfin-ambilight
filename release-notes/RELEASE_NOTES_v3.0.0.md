# Release Notes - v3.0.0

This release moves the plugin onto **Jellyfin 12** and fixes the missing plugin icon and the
enable/disable switch on servers running 12.x.

## Jellyfin 12 support

- Built against `Jellyfin.Controller` / `Jellyfin.Model` **12.0.0** and **.NET 10**, with
  `targetAbi: 12.0.0.0`.
- No behaviour changes: extraction (AMb2/AMb3), playback, UDP streaming to WLED, the REST API
  and the settings page are unchanged. The 12.x plugin interfaces used by the plugin are
  compatible, so this is a retarget rather than a rewrite.

## Fixed: blank plugin icon and dead enable/disable switch

The catalog published plugin versions as `3.0.0` while the assembly reported `3.0.0.0`.
Jellyfin resolves the plugin image and the enable/disable/uninstall actions by exact version
match, so the dashboard asked for `/Plugins/{id}/3.0.0.0/Image` and `/Plugins/{id}/3.0.0.0/Disable`
and both returned 404 — the card showed no icon, the switch did nothing, and nothing was
logged on the server.

Versions are now always published in the four-part form, and the build uses
[JPRM](https://github.com/oddstr13/jellyfin-plugin-repository-manager) (the packaging tool of
the official Jellyfin plugin template), which writes the same version into the project, the
assembly and `meta.json`. The two can no longer disagree. Existing `manifest.json` entries
were normalised as well.

## Packaging

- Plugin zips now follow the standard Jellyfin layout: `Jellyfin.Plugin.Ambilight.dll`,
  `thumb.png` and `meta.json` at the archive root instead of inside a subfolder.
- Release assets are named `ambilight_<version>.zip`.
- CI builds with `jellyfin/jellyfin-meta-plugins` and .NET 10.

## Compatibility

| Plugin version | Jellyfin |
|---|---|
| 3.0.0.0 | 12.0 and newer |
| 2.5.0.0 | 10.10 / 10.11 |

Both remain in the repository manifest, so 10.x servers keep receiving 2.5.0.0.

## Known issue

The plugin's REST API (`/Ambilight/*`) does not declare `[Authorize]`, so its endpoints are
reachable without authentication. A fix is planned for a follow-up release.
