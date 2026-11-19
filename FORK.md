# Roman's fork (branch `seamless`)

Upstream: https://github.com/feschber/lan-mouse. This branch is upstream `main` (69958c6, 2026-10-25) plus:

1. **Upstream PR #548** (kirebyte), merged as is: open the link to a client as soon as it is activated instead of on the first crossing.
2. **Keep links connected** (`src/service.rs`, `keep_connected`): every 15 s, and whenever a peer connects in, reopen the link to any active client that has none. Without it a link closed by a peer restart or a network blip is only reopened by the next crossing, which is refused once with "not connected".
3. **Windows 10054** (`vendor/webrtc-util/src/conn/conn_udp_listener.rs`, wired in with `[patch.crates-io]`): the UDP listener loop no longer exits on `ConnectionReset`. On Windows an ICMP port-unreachable for an earlier send shows up as os error 10054 on the listening socket; upstream then stops listening while the process keeps running (upstream issue #262).

Not patched because upstream `main` already has it: `LAN_MOUSE_HIDDEN=1` starts the macOS app with no window (menu bar icon only). The log level variable is `LAN_MOUSE_LOG_LEVEL`.

## Build

No local toolchain needed. Push to `seamless` (or run the workflow by hand) and `.github/workflows/seamless.yml` builds:

- `lan-mouse-windows-daemon`: `lan-mouse.exe` without GTK (`--no-default-features`), enough for `lan-mouse.exe daemon`.
- `lan-mouse-macos-arm64`: the app bundle, ad hoc signed.

    gh run list --repo namorbrowneyes/lan-mouse --workflow seamless.yml --limit 1
    gh run download <run-id> --repo namorbrowneyes/lan-mouse -n lan-mouse-windows-daemon -D <dir>

macOS arm64 runners are sometimes out of capacity and the job is cancelled after 15 min in the queue: `gh run rerun <run-id> --failed`.

## Install notes

- Both machines must run the fork (or the same upstream commit): a newer build and v0.11.0 cannot hold a link to each other (drops every 2 s).
- macOS: every new build has a new ad hoc signature, so Accessibility and Input Monitoring must be removed and re-added in System Settings, then the app relaunched.

## Updating from upstream

    git fetch upstream && git merge upstream/main

If PR #548 has been merged upstream by then, the merge brings it in and only patches 2 and 3 remain.
