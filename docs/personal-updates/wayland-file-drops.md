# Native Wayland file drag and drop

*PersonalUpdates only. Added 2026-10-09. Not part of upstream PhotoCraft.*

Dragging files from a file manager (Cosmic Files, Nautilus, …) into PhotoCraft now works on a
native Wayland session, without starting PhotoCraft under XWayland. Drops behave as they do on
X11: onto the canvas places the file as a layer, onto the tab strip opens it at that position,
and onto the panels opens it as a new document.

Tested on Pop!_OS 24.04 with the COSMIC desktop (cosmic-comp), two 2560×1440 monitors at 125%.

## Why it didn't work before

PhotoCraft's window comes from eframe 0.36, which uses **winit 0.30.13**. That winit never binds
Wayland's drag-and-drop protocol (`wl_data_device`), so no drop events reached the app. Upstream
worked around it with a "use File › Open or start under XWayland" notice, a different start-screen
hint, and the Preferences › Performance › Linux display server = X11 option.

winit **0.31** (beta) has native Wayland drag and drop, but egui/eframe hasn't moved to it yet,
and porting eframe to a winit beta would be a large fork to maintain. Instead, the feature was added
to the winit 0.30.13 that PhotoCraft already uses.

## The pieces

### 1. winit fork: `ToddoUwU/winit`, branch `wayland-support-for-photocraft`

Local clone: `~/ArtCraft/winit`. The branch starts from the `v0.30.13` tag and adds:

| Commit | What |
|---|---|
| `dabb1f1a` | `src/platform_impl/linux/wayland/dnd.rs` (new), wired into `state.rs`, `seat/mod.rs` and `mod.rs`; `platform::wayland::FILE_DROPS` marker |
| `f4d7d760` | Use the crates.io `dpi` crate, as the published winit does, so the patch doesn't pull in a second copy |

How the drop code works:

- Binds `wl_data_device_manager` (v3) and creates a data device for every seat, including seats
  that appear later.
- When a drag carrying `text/uri-list` enters one of our windows, it accepts it **as a copy only**.
  Never a move: a move would make the file manager delete the originals after the drop.
- The URI list is read through a pipe on winit's event loop, without blocking it (capped at 4 MiB).
- It sends the same events the X11 backend sends: `HoveredFile` per file while hovering,
  `DroppedFile` per file on drop, and `HoveredFileCancelled` when the drag leaves.
- Drag motion is also sent as `CursorMoved`. Wayland sends no pointer events during a drag,
  and this is how PhotoCraft knows whether the files are over the canvas, tab strip or a panel.
- Only local `file://` URIs are used (empty or `localhost` host). They are percent-decoded, and
  comments, other schemes (such as `https://` from browsers) and malformed lines are skipped.
- `finish` is only sent when the compositor selected the copy action. Sending it otherwise is a
  protocol error that would kill the connection.

The fork never goes upstream:

- The `upstream` remote's push URL is `DISABLED-read-only-upstream`.
- No pull requests.
- Commit messages carry no `#123`-style issue references, so GitHub never posts
  "referenced this issue" notes on winit's issues.

### 2. PhotoCraft (`dd211124` on PersonalUpdates)

| File | Change |
|---|---|
| `Cargo.toml` | `[patch.crates-io] winit = { git = "https://github.com/ToddoUwU/winit", branch = "wayland-support-for-photocraft" }` |
| `Cargo.lock` | Only winit's `source` line changes (3 lines), to keep upstream merges simple |
| `apps/photocraft/Cargo.toml` | The direct winit dependency also enables its `wayland` feature |
| `apps/photocraft/src/cursor.rs` | On Wayland, the drop position is egui's own pointer, which the drag motion now updates |
| `apps/photocraft/src/main.rs` | Sets `Services::wayland_file_drops` on Wayland |
| `crates/ui-egui/src/lib.rs` | New `Services::wayland_file_drops` field |
| `crates/ui-egui/src/canvas.rs` | Start screen says "Drop an image or PSD anywhere to open it" when drops work |
| `crates/ui-egui/src/notices.rs` | The "drag-and-drop is unavailable on Wayland" notice is no longer posted |

**Safety net:** `cursor.rs` and `main.rs` reference `winit::platform::wayland::FILE_DROPS`, which only
exists in the fork. If a future `Cargo.lock` change ever stopped the patch from applying, the
build fails with an error instead of drops quietly breaking.

## Tests

- winit fork: `cargo test --lib dnd` covers URI-list parsing (spaces, UTF-8, a non-UTF-8 path,
  `localhost`, other hosts, `https://`, bad escapes, NUL bytes, comments, junk input).
- PhotoCraft: `cargo test -p photocraft-ui-egui --lib -- wayland` covers the hint and notice with
  and without native drops.
- Checked by hand on COSMIC: an image dropped from the file manager opened (a Display P3 image
  correctly brought up the Embedded Profile Mismatch dialog).

To check the protocol traffic if a drop ever does nothing:

```sh
WAYLAND_DEBUG=client ~/ArtCraft/photocraft/target/release/photocraft 2>&1 | grep -E 'wl_data_(offer|device)'
```

A working drop shows `accept`, `set_actions(1, 1)` (1 = copy) and then `drop`. A `leave` instead
of `drop` means the compositor cancelled it because the offer wasn't accepted.

## Maintenance

- **Changing the fork:** commit on `wayland-support-for-photocraft` in `~/ArtCraft/winit`, push to
  `origin`, then run `cargo update -p winit` in `~/ArtCraft/photocraft` and commit the
  `Cargo.lock` change. The `artcraft` launcher does not sync the winit fork.
- **Upstream merges:** the launcher merges `upstream/main` into PersonalUpdates. If that ever conflicts
  in the files above, keep the PersonalUpdates behaviour.
- **When eframe moves to winit 0.31:** remove the `[patch.crates-io]` entry, the `FILE_DROPS`
  references and `Services::wayland_file_drops` (or point it at winit 0.31's own drag events).
  winit 0.31 also adds native Wayland pen input, which would remove the need to switch to XWayland
  when a pen tablet is attached.
- Flatpak is not supported by this change and isn't needed: PhotoCraft is built from source.
