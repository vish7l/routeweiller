# routeweiller

A lightweight C daemon for macOS that watches a single file for changes (writes, deletes, renames, attribute changes) using the BSD kernel event API (`kqueue`/`EVFILT_VNODE`), and fires a native desktop notification when something happens.

## Why

Most file-watcher examples online use Linux's `inotify`. macOS doesn't have that — this project watches at the kernel level using the BSD-native `kqueue` mechanism instead, and dispatches to `terminal-notifier` for the actual notification (macOS has no `libnotify`/D-Bus-style userspace notification bus).

## Build

```bash
clang routeweiller.c -o routeweiller
```

No external libraries required for the watcher itself. Desktop notifications currently shell out to [`terminal-notifier`](https://github.com/julienXX/terminal-notifier):

```bash
brew install terminal-notifier
```

## Usage

```bash
./routeweiller /path/to/file
```

The daemon blocks and watches the given file. On write, delete, rename, attribute change, or extend, it prints the event and fires a notification. `Ctrl+C` / `SIGTERM` triggers a clean shutdown (closes the kqueue and file descriptor).

## Design notes

- **`kqueue` over `FSEvents`**: `kqueue`'s `EVFILT_VNODE` filter gives per-file granularity natively, with no extra framework linking, which fits a single-file watcher better than `FSEvents` (which is directory/volume-oriented by default).
- **No read/access detection**: `NOTE_READ`/`NOTE_CLOSE` exist in newer kqueue headers but are unreliable across macOS versions for regular files — the header being present doesn't guarantee the kernel actually fires the event. Real read/access auditing would require Apple's Endpoint Security framework, which needs a special entitlement and root privileges — out of scope for this project.
- **kqueue is inode-based, not path-based**: editors that save atomically (write to a temp file, delete the original, rename the temp file into place — the standard "safe save" pattern used by many text editors) will show up as `NOTE_DELETE` + `NOTE_RENAME` on the watched fd, not `NOTE_WRITE`, because the original inode is gone. The daemon needs to re-open and re-register the watch on the new file at that path to keep watching after this happens (in-place writers, like most IDEs, don't trigger this).

## Notable bugs found and fixed during development

A few worth mentioning here because they came from genuinely instructive failure modes, not typos:

- **Invalid `free()` from pointer reassignment**: a helper variable meant to hold a path's basename was reassigned inside a `strtok` loop, losing the original pointer returned by `malloc`. Calling `free()` on the reassigned pointer triggered macOS's allocator corruption check, crashing with `SIGTRAP` ("trace trap") — not an obvious symptom to trace back to a stale pointer.
- **Variable shadowing broke signal-driven cleanup**: a file descriptor was declared at global scope for use in the `SIGINT`/`SIGTERM` handler, but a second local declaration with the same name inside `main()` shadowed it. The handler was closing the (always `-1`, never-updated) global instead of the real fd — a silent no-op cleanup bug.
- **`EINVAL` from `kevent()`**: registering a kqueue watch with the wrong file descriptor in the `ident` field (passing the queue's own fd instead of the target file's fd) is rejected by the kernel with `EINVAL`, not a more obvious error.

## Known limitations

- Watches a single file, not a directory tree.
- No read/access-only detection (see Design notes above).
- Notification delivery depends on `terminal-notifier` being installed and on `$PATH`.
