# Mac Pro Trashcan macOS 15 GPU Notes

Notes from a MacPro6,1 "trashcan" running OpenCore Legacy Patcher and macOS 15, focused on Chrome / Chromium / Electron GPU hangs with the dual AMD FirePro D300 setup.

This is mostly for the small group of people still keeping these machines alive and occasionally stepping on the same OCLP + legacy AMD + Chromium GPU rake.

## Machine Context

- MacPro6,1
- OpenCore Legacy Patcher 2.4.1
- macOS 15.7.7 (24G720)
- Dual AMD FirePro D300
- Chrome and Codex Desktop affected

## Short Version

On this machine, OCLP root / GPU patches appeared to be installed correctly. The failure looked more like a Chromium / Electron GPU backend compatibility issue with the OCLP legacy AMD / Metal stack.

Chrome can be recovered with full GPU disable:

```bash
open -a "Google Chrome" --args --disable-gpu
```

Codex Desktop should not use full `--disable-gpu`. It worked by keeping GPU access available but forcing SwiftShader / ANGLE and disabling problematic accelerated paths:

```bash
open -a Codex --args --use-angle=swiftshader --enable-unsafe-swiftshader --disable-gpu-compositing --disable-accelerated-2d-canvas --disable-accelerated-video-decode --disable-features=Vulkan,WebGPU,CanvasOopRasterization,UseSkiaRenderer
```

## Notes

- [Chrome GPU disable notes](chrome-gpu-disable-notes.md)
- [Codex / Chrome GPU handoff](codex-chrome-gpu-handoff.md)

## Caveat

This is not a universal OCLP fix. It is a field note from one MacPro6,1 configuration. Treat it as a workaround and debugging reference, not a guarantee.
