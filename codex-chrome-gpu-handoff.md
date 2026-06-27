# Codex / Chrome GPU Handoff

Date: 2026-06-28
Machine context: MacPro6,1 on OCLP 2.4.1, macOS 15.7.7 (24G720), dual AMD FirePro D300.

## Summary

Chrome and Codex Desktop were hanging when using the default GPU path. OCLP root/GPU patches were checked and appear installed correctly, so the issue is likely Chromium/Electron compatibility with the OCLP legacy AMD/Metal stack rather than missing OCLP patches.

## OCLP Check Results

OCLP CLI path:

```bash
"/Library/Application Support/Dortania/OpenCore-Patcher.app/Contents/MacOS/OpenCore-Patcher"
```

Version:

```text
OpenCore Legacy Patcher 2.4.1
```

Important results:

```text
--auto_patch:
- Detected Snapshot seal not intact, skipping
- Versions match
```

Root patch record:

```text
/System/Library/CoreServices/OpenCore-Legacy-Patcher.plist
Time Patched: June 28, 2026 @ 03:29:46
```

Patchsets present:

```text
AMD Legacy GCN
AMD OpenCL
Monterey OpenCL
Revert Monterey GVA
Modern Wireless
```

Loaded/visible GPU signals:

```text
system_profiler SPDisplaysDataType:
- AMD FirePro D300 x2
- Metal: Supported

kextstat/kmutil showed:
- AMDSupport
- AMD7000Controller
- AMDRadeonX4000
- AMDRadeonX4000HWServices
- AMDFramebuffer
- Lilu
- AMFIPass
```

## Chrome Workaround

Chrome works if GPU is disabled.

Important: the reliable way to get Chrome open was the user's launch command:

```bash
open -a "Google Chrome" --args --disable-gpu
```

The settings below were added as supporting defaults/profile state, but they were not enough by themselves at first. If Chrome hangs again, start with the command above.

Settings written:

```bash
defaults write com.google.Chrome HardwareAccelerationModeEnabled -bool false
defaults write com.google.Chrome DisableGpu -bool true
```

Chrome profile setting:

```json
"hardware_acceleration_mode_enabled": false
```

File:

```text
~/Library/Application Support/Google/Chrome/Local State
```

Backup made:

```text
~/Library/Application Support/Google/Chrome/Local State.codex-backup
```

Emergency launch command:

```bash
open -a "Google Chrome" --args --disable-gpu
```

## Codex Desktop Findings

Codex Desktop is Electron/Chromium based:

```text
/Applications/Codex.app
Bundle ID: com.openai.codex
Chromium profile: ~/Library/Application Support/Codex
```

Important: full `--disable-gpu` is not the right fix for Codex Desktop.

When launched with full GPU disable, Codex reported:

```text
Error: GPU access not allowed. Reason: GPU access is disabled through commandline switch --disable-gpu and --disable-software-rasterizer.
```

The successful workaround is to keep GPU access available, but force software rendering through SwiftShader/ANGLE and disable the problematic accelerated paths.

Successful launch command:

```bash
open -a Codex --args --use-angle=swiftshader --enable-unsafe-swiftshader --disable-gpu-compositing --disable-accelerated-2d-canvas --disable-accelerated-video-decode --disable-features=Vulkan,WebGPU,CanvasOopRasterization,UseSkiaRenderer
```

After this, Codex Desktop reached:

```text
window ready-to-show
renderer mounted
```

and no longer stayed stuck as only the main process at 100% CPU.

## Codex Settings Already Changed

Codex hardware acceleration was disabled in its Chromium profile:

```text
~/Library/Application Support/Codex/Local State
~/Library/Application Support/Codex/Default/Preferences
```

Values:

```json
"hardware_acceleration_mode_enabled": false,
"hardware_acceleration_mode_previous": false
```

Do not set `disable_gpu: true` for Codex Desktop; that can trigger the internal GPU access error.

macOS defaults:

```bash
defaults write com.openai.codex HardwareAccelerationModeEnabled -bool false
defaults delete com.openai.codex DisableGpu
```

Backups made:

```text
~/Library/Application Support/Codex/Local State.codex-backup
~/Library/Application Support/Codex/Default/Preferences.codex-backup
```

GPU cache folders were moved aside:

```text
~/Library/Application Support/Codex/Default/GPUCache.codex-backup-20260628-0415
~/Library/Application Support/Codex/Default/DawnWebGPUCache.codex-backup-20260628-0415
~/Library/Application Support/Codex/GPUPersistentCache.codex-backup-20260628-0415
~/Library/Application Support/Codex/GraphiteDawnCache.codex-backup-20260628-0415
```

## If Codex Updates And Hangs Again

1. Kill stuck Codex Desktop processes:

```bash
pkill -KILL -f "/Applications/Codex.app"
```

2. Launch with SwiftShader:

```bash
open -a Codex --args --use-angle=swiftshader --enable-unsafe-swiftshader --disable-gpu-compositing --disable-accelerated-2d-canvas --disable-accelerated-video-decode --disable-features=Vulkan,WebGPU,CanvasOopRasterization,UseSkiaRenderer
```

3. Confirm process args:

```bash
ps auxww | rg -i "[A]pplications/Codex.app/Contents/MacOS/Codex|[C]odex \\(Renderer\\)|[C]odex \\(Service\\)"
```

Expected:

```text
Main Codex process includes --use-angle=swiftshader
Renderer processes exist
CPU is not stuck at 100% forever
```

4. If it still hangs, collect a sample:

```bash
sample "$(pgrep -f '/Applications/Codex.app/Contents/MacOS/Codex' | head -1)" 5 -file /tmp/codex-desktop-sample.txt
```

Then show `/tmp/codex-desktop-sample.txt`.

## Persistent Fix (2026-06-28)

Three layers to make the GPU workaround survive reboots and updates:

### 1. Wrapper Script

`~/.local/bin/codex` — one command to fix everything and launch:

```bash
codex
```

This script: kills stuck processes → restores Local State + Preferences → clears GPU caches → launches with SwiftShader flags.

### 2. Login LaunchAgent

`~/Library/LaunchAgents/com.castlen3.codex-gpu-fix.plist` runs at every login:

- Restores `hardware_acceleration_mode_enabled: false` in both `Local State` and `Default/Preferences`
- Removes `disable_gpu` from `Local State` (dangerous for Codex; triggers GPU access error)
- Clears all GPU caches
- Sets `defaults write com.openai.codex HardwareAccelerationModeEnabled -bool false`
- Deletes `defaults com.openai.codex DisableGpu`

Log: `~/.local/share/codex-gpu-fix.log`

### 3. Dock .app Wrapper

`/Applications/Codex Safe.app` — AppleScript wrapper that calls `~/.local/bin/codex`. Has the same icon as Codex. Drag this to the Dock and use it instead of the original Codex icon.

### If Codex Still Won't Open

```bash
# Terminal one-liner
~/.local/bin/codex

# Or from Spotlight: type "Codex Safe" and launch the wrapper app
```

## Notes For Future Debugging

- Chrome can use full `--disable-gpu`.
- Codex Desktop should not use full `--disable-gpu`; use SwiftShader/ANGLE instead.
- The likely root cause is Chromium/Electron GPU backend compatibility with OCLP legacy AMD/Metal patches, not missing OCLP GPU patches.
- OCLP patches were verified as present and active on 2026-06-28.
