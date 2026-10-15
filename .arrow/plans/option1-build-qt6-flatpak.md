# Option 1: Build Qt 6 (obs-deps 6.11.1) inside flatpak-builder, point plugin QT_DIR at it

- goal: kill atkAudio flatpak segfault (juce::Component::centreWithSize on filter apply) by making the
  plugin link the SAME obs-deps-built Qt6 ABI that OBS Studio's flatpak bundles at runtime — instead of
  the SDK /usr/lib/qt6 build that drifts.
- status: active (implementing option 1 in the manifest; push to GH; user builds+tests via actions)

## DEFINITIVE TARGET (live evidence, not inference)
- gdb's "Qt 6.11.1" == obsproject/obs-deps release **2026-08-26**, which ships Qt **6.11.1**
  (modules/Core.json, Gui.json, Widgets.json, Network.json all "version":"6.11.1").
- That asset is a WINDOWS prebuilt (.lib/.pdb/.prl, NO .so). There is NO Linux Qt6 bundle anywhere in
  obs-deps (confirmed: only macos + windows-deps-qt6-* assets). So on Linux we MUST build Qt from source
  inside flatpak-builder. SDK 25.08 ships its OWN divergent ~6.11.x build = the drift source, NOT the fix.
- Build target: upstream qtbase **6.11.1** (+ Gui/Widgets/Network/Svg) installed into /app. Plugin's
  find_package(Qt6 Widgets Core REQUIRED) resolves via CMAKE_PREFIX_PATH=/app + QT_DIR=/app/lib/cmake/Qt6,
  so JUCE links the freshly-built obs-equivalent Qt instead of SDK /usr/lib/qt6.

## WHAT CHANGES (com.obsproject.Studio.Plugin.atkAudio.yaml)
1. Add a `modules:` entry building qtbase 6.11.1 from source into the build-extension prefix, with the SDK
   build-deps it needs (fontconfig, xcb-util + xcb-util-wm/util-keysyms/util-image/render, libxkbcommon,
   mesa/GL, icu, alsa, openssl, dbus). flatpak-builder modules get the Sdk, so use build-deps on gtk3/dev.
2. In atkaudio-plugin config-opts:
   - DELETE `-DQt6_VERSION=6.11` (no effect; upstream find_package has no inline VERSION -> can't inject).
   - DELETE `-DQT_DIR=/usr/lib/qt6/lib/cmake/Qt6` (the drift source).
   - ADD `-DCMAKE_PREFIX_PATH=/app -DQT_DIR=/app/lib/cmake/Qt6` so JUCE links the bundled obs-Qt6.
3. KEEP patch 0003 (defer MainHostWindow construction off OBS module thread) as safety net — it stops the
   pre-context crash path and does not conflict with a correct Qt link; verify it still applies after Qt dir change.

## WHY THIS CLOSES THE DRIFT
- Plugin compiled + linked against /app/qt6 6.11.1 == the ABI OBS bundles (same obs-deps source/version).
  One Qt build in the process -> JUCE Component vtable layout matches at runtime -> no corruption.
- RPATH on the plugin .so must include $ORIGIN + /app/plugins/atkAudio/lib so the loader pulls OUR qt6, not
  SDK/usr/lib/qt6. flatpak build-extension prefix is /app/plugins/atkAudio; set CMAKE_INSTALL_RPATH accordingly.

## RISKS (surface to user)
- From-source Qt 6 build in CI is HEAVY/SLOW (~30-45 min) and needs the Sdk to supply all xcb/GL/icu dev
  headers. If a build-dep is missing, flatpak-builder config fails loudly — expect GH iteration on build-deps.
- ABI match depends on matching obs-deps' configure flags (LTO/optimization). Upstream qtbase 6.11.1 with
  default Linux xcb config should be ABI-compatible for JUCE's Core/Widgets usage; minor risk remains. If the
  crash persists at a DIFFERENT call site, the vtable layout still differs -> revisit obs-deps patchset.
- Extension size grows (Qt libs in /app). Acceptable for a filter plugin extension.

## VERIFY (user side)
- Run GH actions build; confirm Qt6 module builds clean and plugin .so links libQt6Core/Gui/Widgets from /app.
  `ldd atkaudio-pluginforobs.so | grep Qt6` should point at /app, not /usr/lib/qt6.
- OBS + apply filter -> no garbage return frames (#1=0x1a etc.) in gdb bt; centreWithSize returns normally.
