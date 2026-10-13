# Handoff: lan-mouse stuck/repeating keys on Hyprland

## Setup
- **Linux:** Hyprland 0.56.2, libei portal capture, wlroots emulation. **Mac:** client `mini.local`, positioned to the left of Linux.
- **Repo:** `/home/sushil/PERSONAL/UTILS/lan-mouse`, branch `fix/forward-keys-while-waiting-for-ack`, pushed to `fork` (`fullstacksushil/lan-mouse`). No PR open. Upstream `origin/main` has no new commits.
- **Untracked, leave alone:** `de.feschber.LanMouse.autostart.desktop`, `handoff.md`.

## Done
| Commit | What | Status |
|---|---|---|
| `a9251e0` | `src/capture.rs`: forward keyboard events while waiting for `Ack` | Verified earlier |
| `7a83c06` | `input-emulation/src/wlroots.rs`: skip modifier events that don't change the shared virtual keyboard's state | **Dead-keyboard bug fixed, user-tested** |

**Dead-keyboard cause:**
- The Mac sends `modifiers(0,0,0,0)` on every capture release.
- Linux replayed it on the virtual keyboard while capture was active.
- Hyprland then removed and re-added the captured keyboard and never sent `StartEmulating` for it, so the keyboard stayed dead.
- This explained 40/51 removals in `lm-linux4` and 82/106 in `lm-linux5`.

## Repeating keys: in progress, fix is in Hyprland
**Cause (confirmed in Hyprland source):**
- `CInputCaptureResource::activate()` (`src/protocols/InputCapture.cpp:214`) never releases keys that the focused client still holds.
- While capture is active, key releases go only to capture, so a key held during the crossing repeats forever in the Linux app.

**Rejected approaches:**
- **Injecting the release from lan-mouse:** Hyprland routes the injected key back into capture, creating a feedback loop (2,203 events/s). Stashed as `git stash@{0}`; safe to drop.
- **`LM_RESTART_SESSION_ON_DEVICE_CHANGE=1`:** the user rejected it. It churns EIS sessions, and Hyprland leaks file descriptors with each one.
- **Hyprland PR #16435:** it releases keys only on a focus change, and capture activation doesn't change focus.

**fcitx5 (Omarchy `omarchy-fcitx5.service`):**
- It is an IME used only for CapsLock XCompose sequences. Its only input method is `keyboard-us`.
- With an IME active, keys bypass `m_pressed` and the IME repeats them itself.
- The user has stopped it for now (`systemctl --user stop omarchy-fcitx5.service`; restart with `start`).
- The Hyprland patch does **not** cover the IME path.

**Patched Hyprland build:**
- Location: `~/builds/hyprland-patched` (Arch packaging repo, checked out at `b8035f2` = 0.56.2-2, so it matches the installed libraries).
- Patch: `input-capture-release-keys.patch`. It adds `CInputManager::releaseSeatKeys()`, which sends a release for every key in `m_pressed` and then clears it. `activate()` calls it before `startEmulating`.
- Packaging:
  - `pkgrel=2.1`.
  - Removed the `libEGL.so libGLESv2.so` soname deps; makepkg had generated them unresolvable, and `libglvnd` covers them.
  - Build with `PATH=/usr/bin:$PATH` because mise's Python 3.10 can't run `generateLuaStubs.py`.
- Output: `hyprland-0.56.2-2.1-x86_64.pkg.tar.zst`, built and dependencies verified. **Not installed yet.**

## Next steps
1. Install: `sudo pacman -U ~/builds/hyprland-patched/hyprland-0.56.2-2.1-x86_64.pkg.tar.zst`. The user was about to run this.
2. Pin it: `IgnorePkg = hyprland` in `/etc/pacman.conf`. Check for existing IgnorePkg entries first.
3. Log out and back in.
4. Test:
   - Hold `d` in a terminal and cross to the Mac. The repeat should stop at the crossing.
   - Also test Ctrl+C typed from the Mac keyboard.
5. If it works, post the Hyprland Discussion. The draft is at `notes/hyprland-discussion.md`; it is generic and doesn't mention lan-mouse. Users can't open issues there, only Discussions. Consider updating the draft to say the fix was tested.
6. Undo if needed: `sudo pacman -S hyprland` and remove the `IgnorePkg` line.

## Open, not started
- **Possible double repeat on the Mac:** Hyprland's InputCapture sends its own key repeat (release+press pairs at the repeat rate) to EIS, and `input-emulation/src/macos.rs` also runs its own repeat task.
- **Mac modifier repeat:** in `macos.rs`, modifiers go through the repeat task, and any other key cancels it with a spurious key-up. Proposed fix: for modifiers, post a `modifier_event` plus one `key_event` and return without starting a repeat.
- **Mac emulation:** releasing a non-last key never posts that key's key-up (`cancel_repeat_task` only releases the repeating key).
- **Boot-time connect storm:** no backoff when lan-mouse starts before the network is up.
- **fcitx5 decision:** keep it disabled (losing CapsLock compose), or extend the Hyprland patch to release keys through the IME.

## Logs
- `/tmp/lm-linux{,2..7}.log`, `/tmp/lm-mac{,2..6}.log`. Logs 4, 5, and 7 are at trace level.
- Run: `LAN_MOUSE_LOG_LEVEL=trace ./target/release/lan-mouse daemon 2>&1 | tee /tmp/lm-linuxN.log`. Rebuild with `cargo build --release` first.
- Hyprland build log: `/tmp/hyprland-build.log`.
