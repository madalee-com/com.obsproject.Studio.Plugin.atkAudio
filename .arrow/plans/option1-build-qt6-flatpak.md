# Plan: Diagnose + fix atkAudio OBS flatpak centreWithSize segfault

- status: active
- goal: Root-cause the flatpak-only `juce::Component::centreWithSize` crash on filter apply; deliver a manifest/code fix that actually changes runtime behavior; confirm direction with runtime ldd/gdb evidence.
- steps:
  - [x] Gather live GH run log — proved Option-1 fix was a NO-OP (qt6 module skipped "no sources"; QT_DIR ignored, WIN32-gated path).
  - [x] Pin crash chain: obs_module_load -> atk::create() -> ObsJucePluginFormatLifecycle::initialize() -> PluginHost2::ctor(:193 make_unique<MainHostWindow>) -> MainHostWindow::ctor. Patch 0003 defers geometry ops via MessageManager::callAsync.
  - [x] Confirm patch-apply geometry: atkref snapshot is already post-patch (0.34.8 a241988); real build uses tag 0.35.0 in manifest.
  - [x] obs-deps 2026-08-26 release ships ONLY macos + windows qt6 bundles (NO Linux Qt6). Confirmed via release asset list.
  - [ ] Get runtime ldd + fresh gdb bt on installed .so (user runs) — cheap ABI-drift vs real-UB discriminator.
  - [ ] Decide: if ldd shows plugin links only Sdk /usr/lib/qt6 (=runtime Qt) -> Theory A dead; pivot to Theory C (real UB / patch not applied). If it links a distinct bundled Qt -> Option-1 rebuild may help.
  - [ ] Apply chosen fix to manifest or source; commit; user re-tests.
- verify: flatpak rebuilds clean AND crash no longer occurs at apply; ldd confirms expected Qt linkage.
