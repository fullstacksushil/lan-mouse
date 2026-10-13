**Title:** input-capture: keys held when capture activates keep auto-repeating in the focused client

### Description

If a key is held when an input-capture session activates, the focused client never receives that key's release. It keeps auto-repeating the key until another key is pressed after capture ends.

This affects any client using the InputCapture portal, such as software KVMs that pass the pointer and keyboard to another machine at a screen edge.

### Steps to reproduce

1. Start any InputCapture portal client with a barrier on one screen edge.
2. Focus a terminal and hold a key, for example `d`, until it starts repeating.
3. While still holding it, move the pointer into the barrier so capture activates.
4. Release the key.

**Expected:** the terminal stops repeating `d`.
**Actual:** the terminal keeps printing `ddddd…` for as long as capture is active, and after it ends until another key is pressed.

### Cause

`CInputCaptureResource::activate()` (`src/protocols/InputCapture.cpp`) starts emulating towards the EIS client but leaves the focused surface's keyboard state untouched. While captured, `CInputManager::onKeyboardKey` doesn't pass key events to the seat (`passEvent` is false when `PROTO::inputCapture->isCaptured()`). So the release of a key that was pressed before activation goes only to the capture client. The focused client still has the key down and its client-side repeat keeps running.

Keyboard focus doesn't change on activation, so no `wl_keyboard.leave` is sent to stop the repeat either.

### Proposed fix

Release every key the seat has delivered to the focused client when capture activates:

```cpp
// CInputManager (public)
void CInputManager::releaseSeatKeys() {
    for (auto key : m_pressed)
        g_pSeatManager->sendKeyboardKey(Time::millis(Time::steadyNow()), key, WL_KEYBOARD_KEY_STATE_RELEASED);
    m_pressed.clear();
}

// CInputCaptureResource::activate(), before m_eis->startEmulating(...)
g_pInputManager->releaseSeatKeys();
```

After activation, the key's physical release goes to the capture client as it does today. If the key is still held when capture ends, its eventual release is dropped by the existing `!CONTAINS && !pressed` check in `onKeyboardKey`, so the client never receives a stray release.

This doesn't cover an active input method grab (for example fcitx5). With an IME active, keys go through `IME->sendKey` and are not tracked in `m_pressed`, so the IME's own repeat continues the same way. That path would need the same release sent through the IME.

### Version

Hyprland 0.56.2 (`efb5099`), Arch Linux.
