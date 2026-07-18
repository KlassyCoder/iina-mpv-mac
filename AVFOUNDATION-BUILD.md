# Building IINA with the `vo_avfoundation` custom libmpv

This fork can render video through mpv's **`vo_avfoundation`** output
(`AVSampleBufferDisplayLayer` → Core Video → the OS compositor) embedded
directly into IINA's window, instead of the default OpenGL render path. It
exists for environments where accelerated OpenGL/Metal renders incorrectly -
notably inside a **Parallels macOS VM**, where the default path shows a black or
grid-corrupted image but the AVFoundation path (what QuickTime uses) works.

The video output itself lives in the sibling **mpv** fork
(`KlassyCoder/mpv-mac`, branch `vo-avfoundation`). This document is the recipe
for building that custom `libmpv` and bundling it into an IINA `.app`.

> Status: this is a working **test build** - Debug configuration, arm64-only,
> ad-hoc signed, built against a reduced mpv feature set. It is not a
> distribution build. See "Known limitations" at the end.

## Prerequisites

- Xcode (command-line `xcodebuild`), Homebrew, Meson/Ninja.
- The mpv fork checked out next to this repo, e.g.
  `../mpv-mac` on branch `vo-avfoundation`.
- `brew install dylibbundler ffmpeg@7`

Paths below assume:
- mpv fork:  `/path/to/mpv-mac`
- IINA fork: `/path/to/iina-mpv-mac`

## 1. Build the custom libmpv (against ffmpeg 7, no libavdevice)

IINA pins an ffmpeg-7-era dylib set. Build mpv against `ffmpeg@7` so the
soname versions match (`libavcodec.61`, etc.). Disable `libavdevice`: the
ffmpeg-7 `libavdevice` pulls in Homebrew's `sdl2-compat` shim (which aborts at
launch trying to load SDL3), and mpv hard-`exit()`s if any linked ffmpeg
library is older than what it was built against - dropping libavdevice avoids
both. It is not needed for playback.

```sh
cd /path/to/mpv-mac
export PKG_CONFIG_PATH="$(brew --prefix ffmpeg@7)/lib/pkgconfig:$(brew --prefix)/lib/pkgconfig"

meson setup build-iina \
  -D{cocoa,coreaudio,avfoundation,gl-cocoa,videotoolbox-gl,videotoolbox-pl}=enabled \
  -D{swift-build,macos-cocoa-cb,macos-media-player,macos-touchbar,vulkan}=enabled \
  -Davfoundation-vo=enabled -Dlibmpv=true -Dlibavdevice=disabled \
  -Dobjc_args="-Wno-error=deprecated -Wno-error=deprecated-declarations"

meson compile -C build-iina
# -> build-iina/libmpv.2.dylib  (links libavcodec.61 etc., no avdevice, no SDL)
```

## 2. Fetch IINA's stock dylib set

The Xcode project links ~71 dylibs from `deps/lib`. Fetch them (the bundled
`other/download_libs.sh` only works when the repo dir is named exactly `iina`,
so fetch directly):

```sh
cd /path/to/iina-mpv-mac
mkdir -p deps/lib
curl -s "https://iina.io/dylibs/universal/filelist.txt" -o /tmp/iina-filelist.txt
cat /tmp/iina-filelist.txt | xargs -P 8 -I {} sh -c \
  'curl -s "https://iina.io/dylibs/universal/{}" -o "deps/lib/{}"'
# yt-dlp is optional for local playback; a stub keeps the Copy Files phase happy:
mkdir -p deps/executable && printf '#!/bin/sh\nexit 0\n' > deps/executable/youtube-dl && chmod +x deps/executable/youtube-dl
```

Keep one SDL-free `libavdevice.61` around for IINA's own link (mpv no longer
uses it, but the Xcode project still references it):

```sh
curl -s "https://iina.io/dylibs/universal/libavdevice.61.dylib" -o /tmp/stock-avdevice.dylib
```

## 3. Build IINA (arm64, ad-hoc signed)

```sh
cd /path/to/iina-mpv-mac
xcodebuild -project iina.xcodeproj -scheme iina -configuration Debug \
  -destination 'generic/platform=macOS' ARCHS=arm64 ONLY_ACTIVE_ARCH=YES \
  CODE_SIGN_IDENTITY="-" CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=YES \
  LD_RUNPATH_SEARCH_PATHS="@executable_path/../Frameworks $PWD/deps/lib" \
  CONFIGURATION_BUILD_DIR="$PWD/build-out" build
# -> build-out/IINA.app
```

## 4. Bundle the custom libmpv + its deps into the app, and re-sign

The Xcode Copy-Dylibs phase copies the stock set; overlay our libmpv and its
ffmpeg-7 dependency closure (as `@rpath`), keep the SDL-free avdevice, drop the
SDL2 shim, strip duplicate rpaths, and re-sign:

```sh
cd /path/to/iina-mpv-mac
FW=build-out/IINA.app/Contents/Frameworks

cp -f /path/to/mpv-mac/build-iina/libmpv.2.dylib "$FW/libmpv.2.dylib"
dylibbundler -of -b -x "$FW/libmpv.2.dylib" -d "$FW" -p @rpath/ \
  -s "$FW" -s "$(brew --prefix ffmpeg@7)/lib" -s "$(brew --prefix)/lib"

cp -f /tmp/stock-avdevice.dylib "$FW/libavdevice.61.3.100.dylib"
rm -f "$FW/libavdevice.61.dylib"; ( cd "$FW" && ln -s libavdevice.61.3.100.dylib libavdevice.61.dylib )
rm -f "$FW/libSDL2-2.0.0.dylib"
install_name_tool -id @rpath/libavdevice.61.3.100.dylib "$FW/libavdevice.61.3.100.dylib"

# strip all LC_RPATH (deps are @rpath-resolved by the app), then re-sign
for f in "$FW"/*.dylib; do [ -L "$f" ] && continue; \
  while otool -l "$f" | grep -q LC_RPATH; do \
    rp=$(otool -l "$f" | awk '/LC_RPATH/{getline;getline;print $2;exit}'); \
    [ -z "$rp" ] && break; install_name_tool -delete_rpath "$rp" "$f" 2>/dev/null || break; done; done
for f in "$FW"/*.dylib; do [ -L "$f" ] || codesign --force --sign - "$f"; done
codesign --force --deep --sign - build-out/IINA.app
```

`build-out/IINA.app` is now self-contained (verify with
`otool -L` showing no `/opt/homebrew` paths).

## 5. Enable embedded mode and run

Embedded mode is opt-in (off by default; the OpenGL path is unchanged when
off):

```sh
defaults write com.colliderli.iina useAVFoundationEmbed -bool true
# or, per-launch:  IINA_VO_AVFOUNDATION=1 build-out/IINA.app/Contents/MacOS/IINA
open -a "$PWD/build-out/IINA.app" /path/to/video.mp4
```

When embedded, IINA sets `vo=avfoundation`, `hwdec=videotoolbox`, and
`--wid=<video view pointer>`, and skips its OpenGL render context; mpv installs
its `AVSampleBufferDisplayLayer` as a sublayer of IINA's video view.

## Known limitations

- **No OSD, no subtitles, no on-screen controls over the video** -
  `AVSampleBufferDisplayLayer` shows only the decoded frame; mpv's OSD/subs are
  normally composited by the OpenGL renderer, which is bypassed.
- **Limited color management** (no `--target-trc`/ICC/tone-mapping).
- **Reduced mpv feature set**: this libmpv is built without `libavdevice` and
  without Lua, so `--ytdl` and the built-in `osc`/`stats` scripts don't exist.
  IINA adapts (skips those options; treats mpv option errors as non-fatal).
  Rebuild mpv with `-Dlua=enabled` to restore ytdl/scripts.
- **Test artifact**: Debug, arm64-only, ad-hoc signed.
- On a real Mac (working accelerated GL) the default OpenGL path is fine; this
  build is specifically for VMs / broken-GL environments.

## Gotchas encountered (for future maintainers)

- Homebrew "SDL2" is now `sdl2-compat`, which `abort()`s at load if it can't
  find SDL3 - hence disabling libavdevice (its only consumer here).
- mpv `check_library_versions()` calls `exit(1)` if a runtime ffmpeg lib is
  older than the build's - all ffmpeg dylibs must match the mpv build (ffmpeg@7).
- The VO must not do `DispatchQueue.main.sync` during creation: IINA creates the
  VO on mpv's core thread (which holds a lock the main thread waits on), so
  blocking the main thread deadlocks. The VO's AppKit setup is done with
  `main.async`.
- Parallels shared folders can serve stale copies of rebuilt dylibs; copy test
  builds to the guest's local disk.
