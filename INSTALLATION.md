# Installation Guide

## Docker Data Volume (Do This First For Docker)

If running Jellyfin in Docker, add a persistent volume for ambilight data before installing the plugin:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    volumes:
      - /path/to/config:/config
      - /path/to/ambilight-data:/config/data/ambilight  # Add this line
```

Or with docker run:
```bash
docker run -v /path/to/ambilight-data:/config/data/ambilight jellyfin/jellyfin
```

## Preferred: Plugin Repository Install

1. Go to **Jellyfin Dashboard** -> **Administration** -> **Plugins** -> **Repositories**
2. Click **Add Repository**
3. Enter:
   - **Repository Name**: `Ambilight`
   - **Repository URL**: `https://raw.githubusercontent.com/luisillo26/jellyfin-ambilight/master/manifest.json`
4. Save, open **Catalog**, find **Ambilight**, and click **Install**
5. Restart Jellyfin

> This release targets **Jellyfin 12.0 or newer (.NET 10)**. On Jellyfin 10.10/10.11 pick **v2.5.0.0** from the catalog instead.

## Manual Install (Fallback)

1. Download the latest `ambilight_x.x.x.x.zip` from [Releases](https://github.com/luisillo26/jellyfin-ambilight/releases)
2. Extract the zip into a new `Ambilight` folder in your Jellyfin plugins directory (the archive holds the DLL, `thumb.png` and `meta.json` at its root):
   - Linux: `/var/lib/jellyfin/plugins/Ambilight/`
   - Windows: `C:\ProgramData\Jellyfin\Server\plugins\Ambilight\`
   - Docker: `/config/plugins/Ambilight/`
3. Restart Jellyfin

## Verification

After installation:
1. Go to **Dashboard** → **Plugins**
2. You should see **Ambilight** in the list
3. Click **Settings** to configure

## Next Steps

See the [README](README.md) for configuration instructions.
