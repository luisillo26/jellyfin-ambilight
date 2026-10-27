# AGENTS.md — Jellyfin Ambilight Plugin

C# / .NET 10 plugin for Jellyfin 12 that drives WLED LED strips with edge-detected video colors. Single project, no solution file.

## Build

```bash
dotnet clean --configuration Release 2>&1 && dotnet restore && dotnet build --no-restore --configuration Release
```

No test suite, no linter, no formatter, no typecheck beyond what the compiler provides. CI runs exactly the two commands above.

## Architecture (non-obvious facts)

- **Entry point:** `Plugin.cs` — `BasePlugin<PluginConfiguration>` + `IHasWebPages`.
- **DI wiring:** `Server/AmbilightServiceRegistrator.cs` registers `AmbilightEntryPoint` as a hosted service.
- **Orchestrator:** `Server/AmbilightEntryPoint.cs` — `IHostedService` that subscribes to Jellyfin playback/library events and manages service lifecycles.
- **Extraction pipeline:** `AmbilightInProcessExtractor` spawns `ffmpeg` as a subprocess, reads raw RGB24 frames, computes per-LED-zone colors (Sobel + Gaussian), writes AMb3 binary files (Deflate-compressed chapters with delta encoding and RLE dedup). FPS/duration probed via `ffprobe`.
- **Playback pipeline:** `AmbilightInProcessPlayer` auto-detects AMb2/AMb3 via magic bytes, decompresses/reconstructs frames, applies visual processing (gamma, saturation, smoothing, rotation), streams raw RGB over UDP to WLED.
- **Storage:** File-based — `{itemId}.ambilight.json` (metadata) + `{itemId}.bin` (data). Default: `/data/ambilight`.
- **REST API:** `Api/AmbilightController.cs` at `/Ambilight/*`.
- **Config UI:** `Configuration/configPage.html` — embedded vanilla JS page, no framework.
- **Userscript:** `userscript/jellyfin-ambilight-ui.user.js` — standalone Tampermonkey script, unrelated to plugin build.

## Key conventions

- All C# files include GPL-3.0 license headers.
- Nullable reference types and implicit usings are enabled.
- Jellyfin packages pinned to `12.0.0` with `ExcludeAssets="runtime"` (not shipped). The 12.x packages only ship `lib/net10.0`, so the project must target `net10.0`.
- NuGet restore forced to nuget.org only.
- No lockfile. Bump the Jellyfin package versions only together with `targetAbi` in `build.yaml` — a plugin compiled against one major version will not load on the next (see the Jellyfin 12.0 notes).
- `build.yaml` is the source of truth for version, `targetAbi`, framework and artifacts. JPRM rewrites `Version`/`AssemblyVersion`/`FileVersion` in `Directory.Build.props` at release time, so never hardcode them in the csproj — a mismatch between the assembly version and the catalog version breaks the plugin icon and the enable/disable switch (exact-match lookup in `PluginManager.GetPlugin(id, version)`).
- Config hot-reloaded at runtime — services read `Plugin.Instance?.Configuration` directly, no caching.

## AMb2 binary format

Magic `"AMb2"` + float fps + 4×ushort LED counts + 1 byte format. Per-frame: ulong timestamp (microseconds) + RGB byte payload.

## AMb3 binary format

Magic `"AMb3"` + 96-byte header (version, flags, duration_us, total_frames, fps, LED counts, compression, index_offset, chunk_count, reserved). Followed by Deflate-compressed chunks, each with a 32-byte chunk header (timestamp, chunk_type: keyframe/delta/RLE, compressed_size, uncompressed_size, frame_count, brightness, flags, reserved). Ends with an `IDX3` seeking index (timestamp → file_offset → chunk_index). Format detection is by magic bytes in the player; AMb2 files continue to work unchanged.

## LED ordering

Top (L→R), Right (T→B), Bottom (R→L), Left (B→T) — clockwise from viewer. `InputPosition` rotates starting LED index.

## Gotchas

- `obj/`, `bin/`, `release/`, `publish/` are in `.gitignore` but may be present — do not commit them.
- `EmbeddedBinariesResolver` extracts platform-specific Rust binaries from DLL resources at runtime. Currently only `AmbilightInProcessExtractor` (C#) is actively used.
- `CleanupStuckExtractions()` runs on startup to reset items stuck in "extracting"/"queued" after crashes.
- Extraction concurrency uses a manual semaphore (lock + counter) — items queue when at capacity.
- Current branch is `devel`, default branch is `master`.
- Jellyfin web client device IDs use `base64(UserAgent|timestamp)` which changes per session — plugin strips timestamps for matching.

## Post-change tasks

After any significant code change, always run these to verify nothing is broken:

```bash
dotnet restore && dotnet build --no-restore --configuration Release
```

There is no test suite, linter, or formatter. A clean build with 0 warnings and 0 errors is the verification gate. If the build fails, fix before committing.
