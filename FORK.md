# Roman's fork (branch `seamless`)

Upstream: https://github.com/feschber/lan-mouse. This branch is upstream `main` (69958c6, 2026-10-25) plus:

1. **Upstream PR #548** (kirebyte), merged as is: open the link to a client as soon as it is activated instead of on the first crossing.
2. **Keep links connected** (`src/service.rs`, `keep_connected`): every 15 s, and whenever a peer connects in, reopen the link to any active client that has none. Without it a link closed by a peer restart or a network blip is only reopened by the next crossing, which is refused once with "not connected".
3. **Windows 10054** (`vendor/webrtc-util/src/conn/conn_udp_listener.rs`, wired in with `[patch.crates-io]`): the UDP listener loop no longer exits on `ConnectionReset`. On Windows an ICMP port-unreachable for an earlier send shows up as os error 10054 on the listening socket; upstream then stops listening while the process keeps running (upstream issue #262).
4. **macOS capture tap at HID level** (`input-capture/src/macos.rs`): the event tap sits at `CGEventTapLocation::HID` instead of `Session`, so it runs before session-level taps of other Mac apps (Wispr Flow, Hammerspoon). With `Session` those apps acted on keys typed while the pointer was on the other machine. Scroll is the exception: it has its own tap at `Session` level, so scroll tools (Mos) still shape the wheel before it is forwarded, as upstream does (build 0ae93068, 2026-11-20). Note: this was written while chasing a scroll outage that turned out to be on the Windows side (a dead AutoHotkey mouse hook), so HID-level scroll capture is NOT proven broken.
5. **macOS media keys are forwarded** (`input-capture/src/macos.rs`, `create_media_tap`): play/pause, next, previous, volume and mute are `NSSystemDefined` events (type 14) that upstream never captures, so they always acted on the Mac. A second raw tap captures them while the pointer is on the other machine and sends them as evdev keys (164, 163, 165, 115, 114, 113). Brightness and other system keys stay local.
6. **Modifiers on posted keystrokes are forwarded** (`input-capture/src/macos.rs`, `synthetic_mods`): apps like Logi Options+ post a keystroke such as Cmd+W with Cmd only in the event flags and no FlagsChanged event, so the other machine got a bare W. The missing modifiers are now pressed around the key. A posted Cmd is sent as **Ctrl** (Windows target: Ctrl is the shortcut key there, and Win+W/Win+T are taken by the Windows shell), so Options+ Cmd+W/Cmd+T arrive as Ctrl+W/Ctrl+T. Cmd typed on the keyboard is unchanged (Meta, via FlagsChanged). Limit: if Ctrl is physically held while Options+ posts a Cmd shortcut, Titan sees Ctrl released early. Test: `cargo test -p input-capture synthetic_mods`, run in CI on macOS.
7. **Posted keystrokes are captured at Session level** (`input-capture/src/macos.rs`): measured 2026-11-23, a keystroke posted by an app (Logi Options+ mouse buttons) carries NO modifier flags at the HID tap, only from Session level on, so patch 6 alone still sent a bare key. The HID tap now lets key events with a source process id pass, and the Session-level tap (the scroll one) forwards them with their flags and drops them. Typed keys are still taken at HID level. `posted_chord` replaces one whole chord: posted Ctrl+Option+Cmd+M (Options+ thumb button; Hammerspoon opens Mission Control on the Mac) goes out as Win+Tab, because Ctrl+Alt on the Windows side is Wispr Flow push-to-talk.

Not patched because upstream `main` already has it: `LAN_MOUSE_HIDDEN=1` starts the macOS app with no window (menu bar icon only). The log level variable is `LAN_MOUSE_LOG_LEVEL`.

## Build

No local toolchain needed. Push to `seamless` (or run the workflow by hand) and `.github/workflows/seamless.yml` builds:

- `lan-mouse-windows-daemon`: `lan-mouse.exe` without GTK (`--no-default-features`), enough for `lan-mouse.exe daemon`.
- `lan-mouse-macos-arm64-macos-15` and `-macos-26`: the app bundle, ad hoc signed, built on two runner pools because one is often out of capacity. Same app; take whichever job finished.

    gh run list --repo namorbrowneyes/lan-mouse --workflow seamless.yml --limit 1
    gh run download <run-id> --repo namorbrowneyes/lan-mouse -n lan-mouse-windows-daemon -D <dir>

macOS arm64 runners are sometimes out of capacity and the job is cancelled after 15 min in the queue: `gh run rerun <run-id> --failed`.

## Install notes

- Both machines must run the fork (or the same upstream commit): a newer build and v0.11.0 cannot hold a link to each other (drops every 2 s).
- macOS: every new build has a new ad hoc signature, so Accessibility and Input Monitoring must be removed and re-added in System Settings, then the app relaunched.

## Updating from upstream

    git fetch upstream && git merge upstream/main

If PR #548 has been merged upstream by then, the merge brings it in and only patches 2 and 3 remain.
