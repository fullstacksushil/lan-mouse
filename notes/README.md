# Notes: lan-mouse on Hyprland (Linux ↔ Mac)

Commands used while debugging stuck and repeating keys. Background is in `../handoff.md`; the Hyprland bug report draft is in `hyprland-discussion.md`.

## Build and run lan-mouse
```sh
cargo build --release                       # always rebuild before a test run
LAN_MOUSE_LOG_LEVEL=trace ./target/release/lan-mouse daemon 2>&1 | tee /tmp/lm-linuxN.log
cargo fmt && cargo clippy --workspace --all-targets --all-features
cargo test --workspace
```
On the Mac, check out `fix/forward-keys-while-waiting-for-ack` before building.

## Reading the logs
```sh
# keyboard device churn from Hyprland
grep -oE "(DeviceAdded|DeviceRemoved)\(Device[A-Za-z]* \{ device: Device\(\"[^\"]*\"\)" /tmp/lm-linuxN.log | sort | uniq -c

# key events captured by libei
grep -E "from ei: KeyboardKey" /tmp/lm-linuxN.log

# events Linux received from the Mac
grep -E "<-<-<-<-<-" /tmp/lm-linuxN.log | grep -vE "ping|motion"

# capture transitions
grep -E "activated: |releasing capture|entering client|leaving client" /tmp/lm-linuxN.log
```

## Session restart flag (off on purpose)
```sh
LM_RESTART_SESSION_ON_DEVICE_CHANGE=1   # forces a capture-session restart on every device change
```
Leave it unset. It churns EIS sessions, and Hyprland leaks file descriptors for each one.

## Hyprland inspection
```sh
hyprctl version
hyprctl devices -j | python3 -c "import json,sys; [print(k['name']) for k in json.load(sys.stdin)['keyboards']]"
hyprctl getoption input:virtualkeyboard:share_states
ls /proc/$(pidof Hyprland)/fd | wc -l     # fd count; should not keep growing
```

## Testing whether Hyprland honors a key-up from another virtual keyboard
```sh
wtype -P s; sleep 1; wtype -p s            # press from one virtual keyboard, release from another
```

## fcitx5 (Omarchy input method, used for CapsLock compose)
`pkill fcitx5` doesn't stop it, because the service restarts it after 2 s (`Restart=always`). Use the service:
```sh
systemctl --user stop omarchy-fcitx5.service
systemctl --user start omarchy-fcitx5.service
systemctl --user disable omarchy-fcitx5.service    # permanent; loses CapsLock compose
```
- Unit: `/usr/share/omarchy/default/systemd/user/omarchy-fcitx5.service`
- Environment: `/usr/share/omarchy/default/environment.d/10-omarchy-fcitx.conf`

## Patched Hyprland (input-capture releases held keys)
The build lives in `~/builds/hyprland-patched`, using the Arch packaging at commit `b8035f2` (0.56.2-2).
```sh
# one-time build dependencies
sudo pacman -S --needed --asdeps cmake glaze hyprland-protocols meson ninja

# build; system python is needed, mise's 3.10 breaks generateLuaStubs.py
cd ~/builds/hyprland-patched
PATH=/usr/bin:$PATH MAKEFLAGS="-j8" makepkg -f --noconfirm

# repackage without recompiling (after a PKGBUILD-only change)
PATH=/usr/bin:$PATH makepkg -R -f --noconfirm

# check that every dependency resolves before installing
for d in $(bsdtar -xOf hyprland-0.56.2-2.1-x86_64.pkg.tar.zst .PKGINFO | awk '/^depend = /{print $3}'); do pacman -T "$d" >/dev/null || echo "UNRESOLVED: $d"; done

# install and pin
sudo pacman -U ~/builds/hyprland-patched/hyprland-0.56.2-2.1-x86_64.pkg.tar.zst
sudo sed -i 's/^#\?IgnorePkg *=.*/IgnorePkg = hyprland/' /etc/pacman.conf

# undo: stock package, then remove the IgnorePkg line
sudo pacman -S hyprland

# remove the build dependencies later
sudo pacman -Rns $(pacman -Qdtq)
```
After a Hyprland update upstream, check whether `InputCapture.cpp` `activate()` releases held keys. If it doesn't, re-apply `input-capture-release-keys.patch` to the new version.

## Git
```sh
git push fork fix/forward-keys-while-waiting-for-ack
git fetch origin && git log --oneline HEAD..origin/main   # new upstream commits
git stash list                                            # stash@{0}: rejected local-key-release attempt
```
