# claude-desktop-asahi

Tools for running and updating **Claude Desktop for Linux (beta)** on **aarch64 / Apple Silicon
(Asahi Linux)** — a user-local install on Fedora, where the Linux beta has no update path of its own.

Three small, self-contained tools that operate on **your own installed copy** of Claude Desktop.
They do **not** redistribute any Anthropic software — `update` fetches the build-pinned `.deb`
from Anthropic's own update feed and swaps it into your user-local install; `patch-oflags` edits
the `app.asar` you already installed, in place; `preseed-cli` fetches the exact build-pinned
Claude Code CLI from Anthropic's own servers with sha256 verification. No Anthropic code is
included in this repository.

> Not affiliated with or endorsed by Anthropic. Use on software you have already installed and
> are licensed to run.

## Status — what is still needed, and what has retired

This repo began as a fix for an aarch64 bug in the app. **Anthropic fixed that bug upstream in
Claude Desktop 1.44121 (September 2026).** That retires the patch, but not the repo: the reason
the updater exists is unrelated to the bug, and it is still true.

| Tool | Why it exists | Status on ≥ 1.44121 | Retires when |
|---|---|---|---|
| `claude-desktop-update` | The Linux beta's in-app updater is **off by design** (`[updater] Linux: in-app updater off (updates via apt)`), and Fedora Asahi has no apt. There is no other way to update a user-local install. | **Still essential.** Unaffected by the bug fix. | Anthropic ships a Linux update path this box can use: the in-app updater enabled on Linux, or an rpm / Flatpak / AppImage channel. Watch for that log line disappearing, or a non-`.deb` feed at `/api/desktop/linux/arm64/`. |
| `claude-desktop-patch-oflags` | Corrected the x86_64 `O_DIRECTORY` / `O_NOFOLLOW` values the app used on aarch64 (see *History*). | **Redundant.** Detects the upstream fix, reports `upstream-fixed`, exits 0, writes nothing. | Now, for any current build. Keep it only while you might install or roll back to a pre-1.44121 build (e.g. a `claude-desktop.bak-*` taken before the fix); it still patches those. |
| `claude-desktop-preseed-cli` | The app's own CLI downloader died on the same bug before any network I/O, so this fetched the pinned CLI for it. | **Blocker gone.** With private dirs working, the app can fetch its own CLI. Retained as a convenience: `claude-desktop-update` uses it to fetch the pinned CLI and validate the manifest *before* a swap. | Optional today. On the reference box the CLI was preseeded ahead of the 1.44121 update, so the app's own download path has not yet been exercised there. |

**The repo retires completely only when the first row does** — when Anthropic provides a working
Linux update path for this platform. Until then, the two bug-era tools are harmless no-ops that
`claude-desktop-update` runs unconditionally, which keeps one flow working across old and new
builds and across rollbacks.

## The upstream fix (≥ 1.44121)

The app now carries a per-architecture table of the two flags and corrects `fs.constants` at
startup whenever the runtime's values don't match `process.arch`:

```js
S = { arm64: { O_DIRECTORY: 16384, O_NOFOLLOW: 32768 },   // 0o40000, 0o100000 — correct for aarch64
      x64:   { O_DIRECTORY: 65536, O_NOFOLLOW: 131072 } } // 0o200000, 0o400000
```

Every private-directory `open()` uses the corrected constants. On aarch64 the correction fires on
each launch, and the app **logs it at error level and reports it to Sentry**:

```
[error] fs.constants open flags on arm64 carry x64/ia32's values
        { arch: 'arm64', observedArch: 'x64/ia32',
          expected: { O_DIRECTORY: 16384, O_NOFOLLOW: 32768 },
          observed: { O_DIRECTORY: 65536, O_NOFOLLOW: 131072 } }      // OpenFlagAbiMismatchError
```

That line is the fix working, not a failure: the app noticed the ABI mismatch and corrected it.
The proof is what is absent — no `EINVAL` lines, and the private-dir subsystems (CLI download,
scheduled tasks, plugin and skills sync) all complete.

The tools detect the fix by that table's presence. `claude-desktop-patch-oflags --check` prints one
of `already-patched | upstream-fixed | needs-patch (N site(s)) | unknown-layout`; only
`unknown-layout` is an error, and `claude-desktop-update` runs that check against the *staged*
build before it touches your live install.

## Tools

### `bin/claude-desktop-update`
One-shot updater for the whole app, so you don't have to drive the two tools below by hand. The
Linux beta's in-app updater is off by design (it logs *"updates via apt"*), and Fedora Asahi has
no apt — so this does what apt would: reads Anthropic's update feed, fetches the build-pinned
`.deb` for this architecture, verifies its size, extracts it, and swaps it into your user-local
install (`~/.local/lib/claude-desktop`). Before the swap it pre-validates the staged build with
both tools below (an unrecognised layout aborts early, with the stage kept for inspection); after
the swap it re-runs them, which on current builds is a no-op and on pre-1.44121 builds re-applies
the patch. It keeps a timestamped backup and rolls back on failure.

Quit Claude Desktop before applying — `patch-oflags` refuses to rewrite a live install, and
overwriting one is unsafe. If you run it while the app is up (e.g. from a terminal the app itself
spawned), it stages everything and stops, so you can quit the app and finish from a fresh
terminal with `--apply`.

```sh
claude-desktop-update          # check + download + stage; applies if the app is already quit
claude-desktop-update --check  # just show installed vs latest, then exit
claude-desktop-update --stage  # download + stage only, never swap
claude-desktop-update --apply  # finish a staged update (run after quitting the app)
```

To read the feed from `releases.claude.com` instead of `api.anthropic.com` (e.g. if the latter is
blocked), set `CLAUDE_DESKTOP_UPDATE_HOST=https://releases.claude.com`.

### `bin/claude-desktop-patch-oflags`
Rewrites the buggy flag expression **in place and byte-for-byte the same length** to the correct
literal for this machine — so no `app.asar` offset or size changes, and the symlink-race
hardening (`O_DIRECTORY | O_NOFOLLOW` as *values*) is preserved. Backs up `app.asar` first, and
zeroes the matching V8 compile-cache header so the patched source is actually used (V8 keys its
cache check on source length). On builds that carry the upstream fix (≥ 1.44121) it reports
`upstream-fixed` and exits 0 without writing.

```sh
claude-desktop-patch-oflags          # quit Claude Desktop first
claude-desktop-patch-oflags --check  # read-only status: already-patched | upstream-fixed |
                                     #   needs-patch (N site(s)) | unknown-layout
# restore:  cp app.asar.prepatch-<stamp> app.asar
```

### `bin/claude-desktop-preseed-cli`
Before 1.44121 the app's own CLI downloader hit the `O_DIRECTORY` bug before any network I/O, so
this did the download for it. It still does exactly what the app would: reads the version +
checksum the installed app is pinned to, fetches that Claude Code build from Anthropic's server,
verifies the sha256, and installs it where the app looks
(`~/.config/Claude/claude-code/<version>/claude`). It finds the pin structurally rather than by a
minified function name, so it survives bundler renames. Today it is a convenience:
`claude-desktop-update` points it at a staged build to fetch the newly pinned CLI and prove the
manifest is readable before the swap.

```sh
claude-desktop-preseed-cli
# then restart Claude Desktop
```

## History: the bug (pre-1.44121)

Claude Desktop's private-directory helper opened directories with the **x86_64 numeric values**
of `O_DIRECTORY` / `O_NOFOLLOW`:

```js
open(dir, O_RDONLY | (O_DIRECTORY ?? 0) | (O_NOFOLLOW ?? 0))
```

On aarch64, x86_64's `O_DIRECTORY` (`0o200000`) is actually **`O_DIRECT`**, and `O_DIRECT` on a
directory always returns `EINVAL`. So every subsystem that creates a private dir failed — most
visibly the Claude Code CLI download, which the UI reported (misleadingly) as *"No path to Claude
code executable / Download failed. Check your internet connection."* The network was fine; the
flag constant was wrong for the architecture. It was reported upstream and fixed in 1.44121; the
patch tool remains for anyone still on, or rolling back to, an older build.

## Install

```sh
install -m755 bin/claude-desktop-* ~/.local/bin/
```

## Bonus: switch between the Desktop app and the CLI without losing your place

Because Claude Desktop drives the same Claude Code session store (`~/.claude/projects/`) as the
standalone `claude` CLI, you can hand a live conversation between them. On an 8 GB box this is
also a big RAM win (the CLI is tens of MB vs. the Electron app's hundreds):

```sh
# Desktop -> CLI: quit the app, then in a terminal, from your project dir:
claude --continue          # resumes the most recent conversation here
# CLI -> Desktop: Ctrl+D, relaunch claude-desktop, reopen the same chat.
```
## AI Disclosure

Developed and maintained with Anthropic Claude Fable 5


## License

MIT — see [`LICENSE`](LICENSE). These are original fix tools and carry no Anthropic code.
