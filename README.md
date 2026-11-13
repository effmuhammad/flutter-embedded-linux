# Embedded Linux (eLinux) embedding for Flutter
![image](https://github.com/sony/flutter-elinux/blob/main/doc/images/overview.png)

[![build-test](https://github.com/sony/flutter-embedded-linux/actions/workflows/build-test.yml/badge.svg)](https://github.com/sony/flutter-embedded-linux/actions/workflows/build-test.yml)

This project was created to develop **non-official** embedded Linux embeddings of [Flutter](https://flutter.dev/). This embedder is focusing on embedded Linux (eLinux) system use cases. It is also implemented based on Flutter desktop for Windows and has some unique features to use it in embedded systems.

If you develop flutter apps for eLinux, use [flutter-elinux](https://github.com/sony/flutter-elinux), which is a non-official extension to the [Flutter SDK](https://github.com/flutter/flutter) to build and debug Flutter apps for embedded Linux devices.

### Repositories

- [flutter-elinux](https://github.com/sony/flutter-elinux): Flutter tools for eLinux
- [flutter-elinux-plugins](https://github.com/sony/flutter-elinux-plugins): Flutter plugins for eLinux
- [flutter-embedded-linux](https://github.com/sony/flutter-embedded-linux): eLinux embedding for Flutter
- [meta-flutter](https://github.com/sony/meta-flutter): Yocto recipes of eLinux embedding for Flutter

## Objective & Goal
Our objective is to use Flutter in embedded systems. We're developing this embedder to use Flutter in embedded products. Ultimately we would like to propose and contribute this software to the mainline of [Flutter Engine](https://github.com/flutter/engine), which means we would like to add an embedded systems version into the Flutter repo for all embedded developers. Please note that this is just our ideal, not the official opinion of the Flutter community.

We would be grateful if you could give us feedback on bugs and new feature requests. We would like to cover the specifications of general-purpose embedded systems.

## Features
- Flutter embedder optimized for Embedded Systems
  - Lightweight than Flutter desktop for Linux (Not using X11 and GTK)
  - Minimal dependent libraries
  - The main target of this embedder is Arm64 devices. We haven't confirmed in Arm 32bit (ARMv7, armhf) devices
- Display backend support
  - [Wayland](https://wayland.freedesktop.org/)
  - Direct rendering module ([DRM](https://en.wikipedia.org/wiki/Direct_Rendering_Manager))
    - Generic Buffer Management ([GBM](https://en.wikipedia.org/wiki/Mesa_(computer_graphics)))
    - [EGLStream](https://docs.nvidia.com/drive/drive_os_5.1.6.1L/nvvib_docs/index.html#page/DRIVE_OS_Linux_SDK_Development_Guide/Graphics/graphics_eglstream_user_guide.html) for NVIDIA devices
- Always single window fullscreen
  - You can choose always-fullscreen or flexible-screen (any size) only when using Wayland/X11 backend
- Keyboard, mouse and touch inputs support
- Equivalent quality to Flutter desktops
- API compatibility with Flutter desktop for Windows and GLFW
  - APIs such as MethodChannel and EventChannel are completely the same with them

## Documentation
Documentation for this software can be found at [Wiki](https://github.com/sony/flutter-embedded-linux/wiki).

## Supported platforms
This embedder supports x64 and Arm64 (aarch64, ARMv8) architectures on Linux which supports either Wayland backend or DRM backend. See [Support status](https://github.com/sony/flutter-elinux/wiki/Support-status) for details.
## DRM-GBM with Flutter 3.50.0 (x64)

Use a `libflutter_engine.so` built from the same Flutter engine revision as the app. The Flutter 3.50.0 SDK used for this setup reports engine revision `371f77202a73e8dff57a8b3bff75bca55c090aee`. Its source is in the Flutter monorepo under `engine/src`; the public `linux-x64-embedder` download is not available for this revision, so build the engine locally.

Install Chromium `depot_tools` and make `gclient` available in `PATH`. For example:

```sh
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git "$HOME/depot_tools"
export PATH="$HOME/depot_tools:$PATH"
"$HOME/depot_tools/ensure_bootstrap"
export FLUTTER_ROOT=/path/to/flutter-3.50.0
```

The Flutter 3.50.0 checkout contains the matching engine source; syncing fetches its third-party dependencies. Preserve an existing `.gclient` configuration:

```sh
if [ ! -f "$FLUTTER_ROOT/.gclient" ]; then
  cp "$FLUTTER_ROOT/engine/scripts/standard.gclient" "$FLUTTER_ROOT/.gclient"
fi
(cd "$FLUTTER_ROOT" && gclient sync --no-history)
```

Build the debug x64 engine and the DRM-GBM runner:

```sh
(
  cd "$FLUTTER_ROOT/engine/src"
  ./flutter/tools/gn --runtime-mode=debug --unoptimized \
    --embedder-for-target --disable-desktop-embeddings \
    --no-build-embedder-examples --enable-fontconfig
ninja -C out/host_debug_unopt -j4 libflutter_engine.so
)

cd /path/to/flutter-embedded-linux
mkdir -p build
cp "$FLUTTER_ROOT/engine/src/out/host_debug_unopt/libflutter_engine.so" \
  build/libflutter_engine.so
cmake -S . -B build \
  -DUSER_PROJECT_PATH=examples/flutter-drm-gbm-backend \
  -DFLUTTER_ENGINE_SOURCE_DIR="$FLUTTER_ROOT/engine/src"
cmake --build build
```

`FLUTTER_ENGINE_SOURCE_DIR` makes CMake compile against the Embedder API header from the same Flutter source tree as the library. Keep it and the engine library on the same revision.

Build the Flutter app with the matching SDK and mode:

```sh
cd /path/to/flutter-app
"$FLUTTER_ROOT/bin/flutter" build linux --debug
```

From an active text virtual terminal, with no X11/Wayland compositor using the DRM device, launch it directly:

```sh
LD_LIBRARY_PATH="/path/to/flutter-embedded-linux/build${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
FLUTTER_DRM_DEVICE=/dev/dri/card0 \
  /path/to/flutter-embedded-linux/build/flutter-drm-gbm-backend \
  --bundle=/absolute/path/to/flutter-app/build/linux/x64/debug/bundle
```

The DRM backend runs fullscreen. The launching user needs access to the DRM card and input event devices. No systemd unit is involved.

References:
- [Flutter engine development setup](https://github.com/flutter/flutter/blob/ab2912035e160d9cad6306df307576da6128ebdb/docs/engine/contributing/Setting-up-the-Engine-development-environment.md)
- [Sony embedded Linux usage guide](https://github.com/sony/flutter-embedded-linux/wiki/How-to-use-Flutter-for-Embedded-Linux)
