# atkAudio — Plugin for OBS Studio (Flatpak)

`com.obsproject.Studio.Plugin.atkAudio` is a [JUCE](https://www.juce.com/)-based plugin for [OBS Studio](https://www.obsproject.com/) that lets you interact directly with audio and MIDI hardware, connect to OBS audio sources, and surface output audio as new OBS sources — useful for complex audio-processing setups.

This repo ships that plugin as an **extension Flatpak** for OBS (`com.obsproject.Studio`). It is built and released by GitHub Actions; there is no Flathub listing, so it is installed from the release asset directly (see below).

## What's in this repo

- `com.obsproject.Studio.Plugin.atkAudio.yaml` — flatpak-builder manifest (the build definition for the extension).
- `com.obsproject.Studio.Plugin.atkAudio.metainfo.xml` — appstream metadata (name, summary, release history).
- `0001-*.patch`, `0003-*.patch` — build patches applied on top of the atkAudio source.
- `.github/workflows/build-flatpak.yaml` — CI that builds the extension and publishes a slim `.flatpak` to each release.
- `README.md` — this file.

## Getting the flatpak

Releases are tagged `vX.Y.Z`. Each tag publishes a single, self-contained asset:

```
atkAudioForOBS-<version>.flatpak
```

(There is no `.flatpakref` and no separate pages/website download — the `.flatpak` file is the only thing you need.)

The latest version is **v0.35.0**. Download `atkAudioForOBS-0.35.0.flatpak` from the [releases page](https://github.com/madalee-com/com.obsproject.Studio.Plugin.atkAudio/releases) (or, if you cloned this repo, from the tag's attached asset).

## Installing (post-install steps)

The plugin ships as a **self-contained Flatpak bundle**, so installation is a single command — no Flatpak remote or `*.flatpakref` is required:

```bash
flatpak install atkAudioForOBS-0.35.0.flatpak
```

Then just launch OBS normally and open the **atkAudio** plugin from the plugins menu. That's it — nothing else to configure to get started.

### Running on Wayland (important)

The editor opens fine on X11, but **crashes under a pure Wayland session** (no X11 socket visible to the flatpak) with a segfault inside the JUCE window code.

- **Root cause:** OBS's flatpak does not grant its sandbox X11 access under Wayland, and JUCE's Linux windowing is X11/XCB-only — so `XOpenDisplay()` fails and the first geometry call dereferences a dead window peer. This is a flatpak sandbox/permission issue, not a code bug.
- **Fix (one-time, user-side):** tell the OBS flatpak to expose the X11 socket. This is the standard, supported way to grant a flatpak X11 access under Wayland:

  ```bash
  flatpak override --user com.obsproject.Studio --socket=x11
  ```

  Then relaunch OBS and open the atkAudio editor. To remove it later, drop the override:

  ```bash
  flatpak override --user --remove com.obsproject.Studio
  ```

  This override is a per-client launch setting (`~/.local/share/flatpak/overrides/com.obsproject.Studio`); it cannot be baked into this plugin's build.

## How the build works (optional detail)

CI uses `flatpak-builder` with `build-bundle: false`, then runs `flatpak build-bundle` manually against flatpak-builder's export repository. This deliberately avoids the static-delta bundling that other recipes do — it keeps the release asset slim (~10 MB) instead of ~100+ MB, and `flatpak install <file>.flatpak` validates the commit metadata out of the box.
