---
name: build-obs-flatpak-github-action
description: Use when packaging an OBS Studio plugin as a flatpak via GitHub Actions, or slimming/debugging that build — the OBS-extension (not app) model, flatpak-builder auto-bundle bloat, runtime-ref anchoring for `flatpak install`, strip/symbol control, and JUCE/CMake patches in sandboxed CI.
---

# Build an OBS Studio Plugin Flatpak with GitHub Actions

## Overview
OBS plugins ship as **flatpak extensions**, not apps. The whole workflow is built around that: you build the extension with `build-extension: true` against the OBS runtime + SDK Qt, then bundle it from flatpak-builder's *export* repo — never with the action's auto-bundle. Get these four things right and the artifact is a slim `~10–17 MB` `.flatpak` that passes `flatpak install`'s metadata validation out of the box. Get them wrong and you get a 64–117 MB bloated bundle or a broken `flatpak install`.

This skill is derived from a working OBS plugin flatpak (`com.obsproject.Studio.Plugin.atkAudio`). Copy the templates in `references/`, then adapt by reading the gotchas below — most failures come from ignoring them.

## When to Use
- Packaging an OBS *plugin/extension* as a flatpak via GitHub Actions (Flathub-independent, no remote).
- A build/publish workflow that emits a bloated `.flatpak` (~64–117 MB) and needs slimming.
- Debugging why `flatpak install <file>.flatpak` fails or the extension won't load in OBS on Linux.

## When NOT to Use
- Packaging a standalone OBS app or non-OBS extension → the OBS-runtime anchoring assumptions don't hold.
- You just need to *bump* an existing manifest to a new upstream version → use `dependency-updates`/cron patterns, not this build model.

## The Four Non-Negotiables

1. **It is an extension, never an app.**
   ```yaml
   build-extension: true          # builds against org.freedesktop.Sdk, no OBS runtime needed in CI
   runtime: com.obsproject.Studio # host runtime that provides libobs/obs-frontend-api at RUNTIME
   runtime-extensions: []         # do NOT list the OBS runtime here — it stays out of the bundle
   sdk: org.freedesktop.Sdk//25.08
   ```
   `build-extension` lets CI build against SDK Qt **without** the OBS runtime installed (avoids the `"com.obsproject.Studio/... not installed"` error). Build-time Qt == runtime SDK Qt, so JUCE's `find_package(Qt6)` links the same in-process Qt OBS uses → no ABI drift. Don't rebuild obs-deps Qt; skip it.

2. **Never use `build-bundle: true`.** The action's auto-bundle runs `flatpak build-bundle --runtime --runtime-repo=<flathub>`, which emits a *static-delta* ostree (deltas + update metadata) and bloats the artifact to ~100–117 MB. Instead:
   ```yaml
   uses: flatpak/flatpak-github-actions/flatpak-builder@v6.7
   with:
     build-bundle: false          # <-- the key line; also set build-debug-bundle: false
     repo-dir: repo               # flatpak-builder's export repo, not a hand-built one
     manifest-path: <id>.yaml
   ```
   Then in a run step, bundle **manually from that export repo** (see template).

3. **Anchor the bundle on the *extension's* runtime ref, not the OBS platform ref.** flatpak-builder places the extension commit under `runtime/<app-id>/x86_64/stable`, which contains `.Plugin.<name>` and cannot match the OBS platform ref `com.obsproject.Studio/x86_64/stable` (host-verified: ext refs are rejected there).
   ```bash
   REF=$(ostree --repo=repo refs | grep -E "\.Plugin\.<name>/x86_64/stable$" || true)
   [ -n "$REF" ] || { echo "no ${ID} runtime ref in export repo"; exit 1; }
   flatpak build-bundle repo out-<ver>.flatpak "$REF" stable
   ```
   The empty-REF guard fails **before** the bundle step, so a missing ref is caught early.

4. **Don't re-add bloat after flatpak-builder.** Because we bundle from a fresh export repo that contains *only* the extension commit, no OBS-platform static-delta base is emitted → slim artifact. Two things destroy this if left in:
   - **Stray runtime-style install paths** (e.g. `obs-plugins/<arch>/`, `install.sh`, `share/`) that upstream's `install` rule drops in. flatpak build-bundle re-encodes them as large static deltas → 64 MB bloat. Remove exactly those paths in `post-install`. Use explicit `rm -rf <path>` — **not** `find -delete`, which refuses to delete a still-non-empty parent dir and fails the build.
   - **Debug symbols.** The flatpak-builder action doesn't support a `strip` input (it warns "Unexpected input(s) 'strip'"). Set the build type at configure time instead: `-DCMAKE_BUILD_TYPE:STRING=Release`, `-DATK_DEBUG=OFF`, then `strip --strip-all` the built `.so` in post-install. Release + strip leaves no `.debug_*` sections and trims the symbol table (~20 MB → ~17 MB).

## Manifest Source Tag — Derive the Version Carefully
The release version is the atkAudio source git module's `tag:`. Multiple `sources:` modules each carry a `tag:` (atkAudio *and* JUCE), so **don't** blindly grab the first `tag:` match — target the atkAudio module specifically. In CI:
```bash
VER=$(grep -E '^\s+tag:' <manifest>.yaml | sed -n '<N>p' | awk '{print $2}' | tr -d '[:space:]')
```
where `<N>` is the line number of the atkAudio source module. Hardcoding is acceptable only if module order is stable; otherwise parse the module block directly.

## JUCE + CMake Patches in Sandboxed CI
- JUCE arrives as a `_deps/juce-src` git source and must be patched **inline** (`type: patch`). flatpak-builder extracts every source into one build dir and runs `patch -p1`, so each `.patch` header must be prefixed with the dest path (`a/_deps/juce-src/...`) so `-p1` resolves relative to the build root.
- Force JUCE's FetchContent to reuse that checkout, never hit the network: `-DFETCHCONTENT_FULLY_DISCONNECTED=ON`.
- Guard test subdirectories that upstream CMake assumes exist (`-DBUILD_TESTING=OFF`).

## Runtime Gotcha (post-install, not build)
The plugin editor **crashes under a pure Wayland session**: OBS's flatpak denies X11 access to its sandbox and JUCE's Linux windowing is X11/XCB-only, so `XOpenDisplay()` fails and geometry ops dereference a dead QWidget peer. Fix is per-client, user-side (can't be baked into the build):
```bash
flatpak override --user com.obsproject.Studio --socket=x11
```

## References / Templates
- `references/workflow.yaml` — the full working CI workflow (build job + release job). Copy and rename.
- `references/manifest.yaml` — a trimmed, annotated manifest template (extension flags, strip/purge logic, sources + patches). Adapt module URLs/tags for your plugin.

## Verification Checklist (run before claiming success)
- [ ] Artifact is a single `.flatpak` with **no** `.flatpakref` and no second differently-named bundle.
- [ ] Artifact size is `~10–20 MB`, not 64 MB+. If larger, stray runtime paths or debug symbols slipped through — grep post-install for `obs-plugins/<arch>`, `install.sh`, `share/`, and confirm `strip --strip-all`.
- [ ] No `build-bundle: true` and no `--runtime-repo` anywhere; bundling is manual from the export `repo/`.
- [ ] REF anchors on `.Plugin.<name>/x86_64/stable`; an explicit empty-REF guard fails the job if absent.
- [ ] No separate extra-data/metadata "verify" step — flatpak-builder already sets `xa.ref`/`xc.metadata` for runtime-ref extensions, so `flatpak install` validates out of the box. Remove any hand-rolled `validate_commit_metadata` check; the bundle guard fails first if anything is missing.
- [ ] JUCE patch headers are prefixed with their `_deps/...` dest path; `-DFETCHCONTENT_FULLY_DISCONNECTED=ON` present.
- [ ] CI runs on a pinned Ubuntu + the matching `flatpak-github-actions:freedesktop-<ver>` container (`--privileged`), matching the OBS Studio flatpak build environment.
- [ ] **User end-to-end:** host-level install may not reproduce; hand the user `flatpak install out-<ver>.flatpak` to validate on their machine (they own the install loop).
