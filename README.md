# MacPro6,1 macOS 15 Chromium GPU Notes

Field notes for a MacPro6,1 "trashcan" on OCLP 2.4.1, macOS 15.7.7, dual AMD FirePro D300.

The OCLP GPU patches looked fine. The real problem appeared to be Chromium / Electron hitting a bad GPU path on this machine.

## What Worked

### Chrome

Chrome was recoverable with full GPU disable:

```bash
open -a "Google Chrome" --args --disable-gpu
```

Supporting settings:

```bash
defaults write com.google.Chrome HardwareAccelerationModeEnabled -bool false
defaults write com.google.Chrome DisableGpu -bool true
```

### Codex Desktop

Do not use full `--disable-gpu` for Codex Desktop. That can trigger:

```text
Error: GPU access not allowed
```

The working launch flags were:

```bash
open -a Codex --args --use-angle=swiftshader --enable-unsafe-swiftshader --disable-gpu-compositing --disable-accelerated-2d-canvas --disable-accelerated-video-decode --disable-features=Vulkan,WebGPU,CanvasOopRasterization,UseSkiaRenderer
```

## Persistent Codex Fix

Keep the desktop workaround separate from the CLI.

- `codex` should remain the real Codex CLI
- `codex-safe` should be the desktop launcher workaround
- `Codex Safe.app` should call `codex-safe`

Example `~/.local/bin/codex-safe`:

```bash
#!/usr/bin/env bash
set -euo pipefail

APP="/Applications/Codex.app"
CODEX_DIR="$HOME/Library/Application Support/Codex"
LOCAL_STATE="$CODEX_DIR/Local State"
PREFS="$CODEX_DIR/Default/Preferences"

set_json_bool() {
    local file="$1"
    local key="$2"
    local value="$3"
    [[ -f "$file" ]] || return 0
    /usr/bin/jq --argjson v "$value" ".${key} = \$v" "$file" > "$file.tmp"
    mv "$file.tmp" "$file"
}

set_json_bool "$LOCAL_STATE" hardware_acceleration_mode_enabled false
set_json_bool "$PREFS" hardware_acceleration_mode_enabled false
set_json_bool "$PREFS" hardware_acceleration_mode_previous false

if [[ -f "$LOCAL_STATE" ]]; then
    /usr/bin/jq 'del(.disable_gpu)' "$LOCAL_STATE" > "$LOCAL_STATE.tmp" 2>/dev/null || true
    [[ -f "$LOCAL_STATE.tmp" ]] && mv "$LOCAL_STATE.tmp" "$LOCAL_STATE"
fi

rm -rf \
    "$CODEX_DIR/Default/GPUCache" \
    "$CODEX_DIR/Default/DawnWebGPUCache" \
    "$CODEX_DIR/GPUPersistentCache" \
    "$CODEX_DIR/GraphiteDawnCache" 2>/dev/null || true

open -a "$APP" --args \
    --use-angle=swiftshader \
    --enable-unsafe-swiftshader \
    --disable-gpu-compositing \
    --disable-accelerated-2d-canvas \
    --disable-accelerated-video-decode \
    --disable-features=Vulkan,WebGPU,CanvasOopRasterization,UseSkiaRenderer
```

AppleScript wrapper app:

```applescript
do shell script "/Users/yourname/.local/bin/codex-safe"
```

Use that wrapper in Dock instead of the original `Codex.app`.

## Mistake To Avoid

Do not replace terminal `codex` with the desktop workaround script. If you do that, you lose the normal CLI and make recovery harder.

## Caveat

This is one-machine workaround data, not a universal OCLP fix.
