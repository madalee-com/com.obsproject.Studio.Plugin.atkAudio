# ATKAudio Plugin for OBS packaged as a Flatpak

This thing is awesome.

## Running on Wayland

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

<img width="481" height="517" alt="image" src="https://github.com/user-attachments/assets/8897a198-ed1f-4b95-af51-84b7f8e90422" />

<img width="1319" height="1038" alt="image" src="https://github.com/user-attachments/assets/879f07a8-743f-42b7-93f5-b076ed20a31a" />
