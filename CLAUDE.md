# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This is **IINA** (fork `KlassyCoder/iina-mpv-mac`), a macOS media player that is a **frontend for mpv**. UI is Swift/Cocoa; media playback is delegated to **libmpv**. IINA does not decode or render video itself in the general case; it hosts mpv and draws mpv's output.

The reason this fork exists: getting IINA to play video correctly and smoothly **inside a Parallels macOS VM**, where the default accelerated render path produces a black or corrupted image. See "Parallels VM: the problem and fixes" below.

## Sibling project: mpv-mac

A companion mpv source checkout lives at:

```
/Users/coreyklass/Documents/Work-Projects/hacking/mpv-mac
```

It is configured as an allowed additional directory for this project (`.claude/settings.local.json`), so Claude Code can read/build it from within this repo.

That tree carries a new mpv video output, **`vo_avfoundation`** (branch `vo-avfoundation`), which enqueues VideoToolbox-decoded frames directly onto an `AVSampleBufferDisplayLayer` (Core Video -> the OS Core Animation compositor), bypassing OpenGL/Metal. It is the mpv-native version of the display path QuickTime and Quick Look use, and the reason it exists is the same Parallels VM problem below. See that project's `CLAUDE.md` and `DOCS/man/vo.rst` (`avfoundation` entry) for details. Relevant files there:

- `video/out/vo_avfoundation.m` - the `vo_driver` (hwdec-ctx registration, CVPixelBuffer -> CMSampleBuffer wrapping, layer enqueue).
- `video/out/mac/av_layer.swift` - the `AVSampleBufferDisplayLayer` / window host.

Note the architectural mismatch (detailed under "Native AVFoundation display" below): `vo_avfoundation` makes **mpv own its own NSWindow**, whereas IINA embeds mpv and drives rendering itself. So `--vo=avfoundation` is not a drop-in for IINA; using it inside IINA requires the integration work described later, not just a flag.

## How IINA renders (why VM rendering breaks)

Key facts, verified in this fork:

- IINA forces mpv's video output to `libmpv` (`iina/MPVController.swift:664`, `setOptionString(vo, "libmpv")`). mpv's own `vo=gpu`/`gpu-next` (and their Metal/OpenGL backends) are therefore **not** what draws the image in IINA.
- IINA drives rendering through mpv's **libmpv render API** with `MPV_RENDER_API_TYPE_OPENGL` (`iina/MPVController.swift:679-691`, `mpv_render_context_create`).
- The GL output goes into IINA's own `CAOpenGLLayer` backed by a Core OpenGL (CGL) context (`iina/ViewLayer.swift`).
- The CGL pixel format is either hardware-accelerated (`kCGLPFAAccelerated`) or Apple's CPU software renderer (`kCGLRendererGenericFloatID`), selected from the mpv option **`cocoa-cb-sw-renderer`** (`iina/ViewLayer.swift:405,439-441`; option defined `iina/MPVOption.swift:1302-1303`).

Consequence: mpv's `gpu-api`, `vo=...`, and Metal settings do nothing in IINA, because IINA replaces mpv's video output with its own OpenGL layer. The lever that matters is `cocoa-cb-sw-renderer`.

## Parallels VM: the problem and fixes

### The problem
In a Parallels macOS guest, IINA shows a **black image**, or occasionally a choppy frame with a **grid of black lines** overlaid. Cause: the guest's virtual GPU botches **accelerated OpenGL** rendering. IINA's default CGL path is hardware-accelerated (`kCGLPFAAccelerated`), so it hits the broken path. QuickTime and Quick Look are smooth in the same VM because they display through **AVFoundation / Core Video / Core Animation**, which the VM handles cleanly.

### Decode vs display (the core insight)
"Hardware accelerated video" splits into two independent concerns in a VM:

1. **Decode.** `VideoToolbox` is an API, not necessarily silicon. When the GPU decode block is not exposed to the guest, it transparently falls back to Apple's optimized **CPU** decoder behind the same API (visible as CPU spiking in `VTDecoderXPCService`, not `duetexpertd`). QuickTime, Quick Look, and IINA with `hwdec=videotoolbox*` all use this and it works in the VM.
2. **Display.** QuickTime/Quick Look present via `AVSampleBufferDisplayLayer` -> Core Video -> the OS compositor (works in the VM). IINA presents via **accelerated OpenGL** (renders black/grid in the VM).

So the fix is to change the **display** path, not the decoder.

### The working config fix (no code changes; works today)
In IINA -> Settings -> Advanced -> enable "Set additional options", add these as **key = value** rows (no leading `--`), then **fully quit and relaunch IINA** (options are read when the layer is built, not on file reopen):

```
cocoa-cb-sw-renderer = yes      # force Apple's CPU software GL renderer (avoids the broken accelerated GL path)
hwdec = videotoolbox-copy       # VideoToolbox decode with copy-back to system memory
profile = fast                  # cheaper software compositing (biggest smoothness win on the CPU renderer)
```

Notes from working through this:
- `gpu-api = opengl` is **rejected** (`return value -7`) - not a usable lever here.
- `cocoa-cb-sw-renderer = yes` alone renders correctly but is choppy; adding `hwdec = videotoolbox-copy` moves decode off the CPU; `profile = fast` makes the CPU compositing cheap enough to be smooth.
- With pure software decode (`hwdec = no`), also useful: `vd-lavc-threads = 0`, `vd-lavc-fast = yes`, `vd-lavc-skiploopfilter = all`. Not needed once VideoToolbox decode is on.
- Net: decode on VideoToolbox's CPU decoder, compositing on Apple's CPU OpenGL renderer. 1080p is smooth; heavy 4K/10-bit may stay marginal because compositing is still on the CPU.

### Native AVFoundation display (the integration goal)
To match QuickTime (VideoToolbox decode + OS-composited display, no CPU compositing), IINA needs to display through `AVSampleBufferDisplayLayer` instead of its OpenGL layer. The public libmpv render API only exposes `MPV_RENDER_API_TYPE_OPENGL` and `MPV_RENDER_API_TYPE_SW`; it never hands out the VideoToolbox `CVPixelBuffer`. So there are three realistic paths, with different trade-offs:

- **Path A - IINA-side sample-buffer layer (recommended for IINA).** Switch IINA's render context to `MPV_RENDER_API_TYPE_SW`, render each frame into a `CVPixelBuffer`, and display it via an `AVSampleBufferDisplayLayer` that IINA owns and embeds in its existing window (keeps the OSC, title bar, all IINA UI). Touch-points: `MPVController.swift` (render-context type), a new sample-buffer layer class beside `ViewLayer.swift`, and `VideoView.swift` to host it. Caveat: this is still **CPU render** (SW target), so smoothness is similar to the `cocoa-cb-sw-renderer=yes` config above; the win is that the final present goes through the OS compositor rather than software OpenGL. It does **not** use the sibling `vo_avfoundation`.
- **Path B - standalone mpv with `vo_avfoundation`.** Use the mpv-mac build directly: `mpv --vo=avfoundation --hwdec=videotoolbox <file>`. True VideoToolbox-decode-to-OS-composited-display, but mpv owns its own window - **no IINA UI**.
- **Path C - `--wid` embedding for `vo_avfoundation`.** Add foreign-parent-view (`--wid`) embedding to mpv's macOS window stack so `vo_avfoundation` can render into an IINA-provided `NSView`, then let IINA host it. mpv's macOS backend currently only *exposes* its window id outward (`mpv-mac/video/out/mac/common.swift:561`); it has **no** `--wid` embedding. This is the largest effort (upstream-mpv-scale C/Swift work) but the only path that gets true GPU-decode-to-display **and** keeps IINA's UI.

Be honest in any plan about which path is being taken; do not imply `vo_avfoundation` drops into IINA unchanged.

## Build, run

IINA builds with **Xcode** (`iina.xcodeproj`), not a command-line meson/make flow. libmpv + FFmpeg arrive as prebuilt dylibs under `deps/lib`, with headers in `deps/include`.

```sh
./other/download_libs.sh      # fetch prebuilt universal dylibs (libmpv, ffmpeg, ...) into deps/
# then open iina.xcodeproj in the latest public Xcode and Build
```

### Swapping in a custom libmpv (e.g. one built from mpv-mac)
IINA's README section "Building mpv manually" is the reference; the shape is:

1. Build libmpv from the mpv-mac tree (meson; produces `libmpv.*.dylib`). For a VM-focused build enable the macOS feature set (see `mpv-mac/CLAUDE.md`); `vo_avfoundation` is gated by the `avfoundation-vo` meson option.
2. Copy the matching mpv/FFmpeg headers into `deps/include/`, replacing the current ones (**header and dylib versions must match**).
3. Run `other/parse_doc.rb` and copy the regenerated `MPVOption.swift`, `MPVCommand.swift`, `MPVProperty.swift` from `other/` into `iina/` (only needed if the mpv option/command/property API changed; these files are generated - never hand-edit them).
4. Run `other/change_lib_dependencies.rb` to rewrite install names and stage the dylibs into `deps/lib`.
5. In Xcode, replace the `.dylib` references in the Frameworks group with the ones from `deps/lib`, and ensure they are in the "Copy Dylibs" and "Link Binary With Libraries" phases.
6. Build.

Important: swapping libmpv alone gets you a libmpv that *contains* `vo_avfoundation`, but IINA still forces `vo=libmpv` and drives the OpenGL render API (above). Actually using AVFoundation display in IINA requires Path A or Path C, not just the new dylib.

## Conventions

- Swift/Cocoa (ObjC where legacy). 2-space indentation, remove trailing whitespace (per `CONTRIBUTING.md` / `.editorconfig`).
- `MPVOption.swift`, `MPVCommand.swift`, `MPVProperty.swift` are **generated** by `other/parse_doc.rb` - do not edit by hand.
- Main development branch upstream is `develop`; rebase on it before PRs.
- This is a personal fork; commit/push only when asked, and branch off before committing on a mainline branch.
