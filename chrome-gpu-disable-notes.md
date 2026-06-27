# Chrome GPU Disable Notes

Date: 2026-06-28
Machine context: OCLP MacPro6,1, macOS 15.7.7, AMD FirePro D300.

## Problem

Chrome hangs or fails to open when GPU / hardware acceleration is enabled.

This appears to be a Chromium GPU backend compatibility issue with the OCLP legacy AMD/Metal stack. OCLP GPU patches were checked separately and appeared installed correctly.

## Working Fix

Disable Chrome hardware acceleration / GPU acceleration.

Important caveat from the actual fix:

The reliable way to get Chrome open on this machine was to launch it first with:

```bash
open -a "Google Chrome" --args --disable-gpu
```

The defaults / `Local State` settings below were applied afterward as supporting settings, but by themselves they were not enough to get Chrome open reliably.

Settings currently applied:

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

Backup:

```text
~/Library/Application Support/Google/Chrome/Local State.codex-backup
```

## Emergency Launch

If Chrome gets stuck after an update, use this exact command first:

```bash
open -a "Google Chrome" --args --disable-gpu
```

Then go to:

```text
Chrome Settings -> System -> Use graphics acceleration when available
```

Turn it off, then relaunch Chrome.

If normal Dock/Finder launch still hangs after turning the setting off, keep using the command above to start Chrome.

## Manual Terminal Fix

If the UI cannot be opened, do not start with the settings below. First open Chrome with:

```bash
open -a "Google Chrome" --args --disable-gpu
```

Then apply or confirm these supporting settings:

```bash
defaults write com.google.Chrome HardwareAccelerationModeEnabled -bool false
defaults write com.google.Chrome DisableGpu -bool true
```

Then write Chrome's `Local State` setting:

```bash
ruby -rjson -e 'path = File.expand_path("~/Library/Application Support/Google/Chrome/Local State"); data = JSON.parse(File.read(path)); data["hardware_acceleration_mode_enabled"] = false; File.write(path, JSON.generate(data))'
```

## Verify

Check macOS defaults:

```bash
defaults read com.google.Chrome HardwareAccelerationModeEnabled
defaults read com.google.Chrome DisableGpu
```

Expected:

```text
0
1
```

Check Chrome profile:

```bash
/usr/bin/jq -r '.hardware_acceleration_mode_enabled' "$HOME/Library/Application Support/Google/Chrome/Local State"
```

Expected:

```text
false
```

## What Not To Confuse With Codex Desktop

Chrome can be launched with full:

```bash
--disable-gpu
```

Codex Desktop should not use full `--disable-gpu`; it needs SwiftShader instead. See:

```text
~/Desktop/Codex-Chrome-GPU-handoff.md
```

## Restore / Undo

To allow Chrome GPU acceleration again:

```bash
defaults delete com.google.Chrome DisableGpu
defaults write com.google.Chrome HardwareAccelerationModeEnabled -bool true
```

Then update `Local State`:

```bash
ruby -rjson -e 'path = File.expand_path("~/Library/Application Support/Google/Chrome/Local State"); data = JSON.parse(File.read(path)); data["hardware_acceleration_mode_enabled"] = true; File.write(path, JSON.generate(data))'
```

On this machine, re-enabling GPU acceleration is expected to make Chrome hang again.
