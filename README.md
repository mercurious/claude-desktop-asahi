# claude-desktop-asahi

Fixes for **Claude Desktop for Linux (beta)** on **aarch64 / Apple Silicon (Asahi Linux)**.

Three small, self-contained tools that update, patch, or preseed **your own installed copy** of
Claude Desktop. They do **not** redistribute any Anthropic software — `update` fetches the
build-pinned `.deb` from Anthropic's own update feed and swaps it into your user-local install;
`patch-oflags` edits the `app.asar` you already installed, in place; `preseed-cli` fetches the
exact build-pinned Claude Code CLI from Anthropic's own servers with sha256 verification. No
Anthropic code is included in this repository.

> Not affiliated with or endorsed by Anthropic. Use on software you have already installed and
> are licensed to run.

## The bug these fix

Claude Desktop's private-directory helper opens directories with the **x86_64 numeric values**
of `O_DIRECTORY` / `O_NOFOLLOW`:

```js
open(dir, O_RDONLY | (O_DIRECTORY ?? 0) | (O_NOFOLLOW ?? 0))
```

On aarch64, x86_64's `O_DIRECTORY` (`0o200000`) is actually **`O_DIRECT`**, and `O_DIRECT` on a
directory always returns `EINVAL`. So every subsystem that creates a private dir fails — most
visibly the Claude Code CLI download, which the UI reports (misleadingly) as *"No path to Claude
code executable / Download failed. Check your internet connection."* The network is fine; the
flag constant is wrong for the architecture. (Reported upstream; these let you run in the
meantime, and after each app update, which replaces `app.asar`.)

**Fixed upstream in Claude Desktop ≥ 1.44121.** The app now ships a per-architecture constants
table and corrects `fs.constants` itself when the runtime's values don't match `process.arch`.
The tools detect this: on such builds `patch-oflags` reports `upstream-fixed` and exits 0
without touching anything, so `claude-desktop-update` keeps working across the transition and
older builds still get patched.

## Tools

### `bin/claude-desktop-update`
One-shot updater for the whole app, so you don't have to drive the two tools below by hand. The
Linux beta's in-app updater is off by design (it logs *"updates via apt"*), and Fedora Asahi has
no apt — so this does what apt would: reads Anthropic's update feed, fetches the build-pinned
`.deb` for this architecture, verifies its size, extracts it, and swaps it into your user-local
install (`~/.local/lib/claude-desktop`). Because an update replaces `app.asar` (reverting the
O_DIRECTORY fix) and repins the Claude Code CLI, it then re-runs `patch-oflags` and `preseed-cli`
for you. It keeps a timestamped backup and rolls back on failure.

Quit Claude Desktop before applying — `patch-oflags` refuses while it runs, and overwriting a
live install is unsafe. If you run it while the app is up (e.g. from a terminal the app itself
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
Rewrites that flag expression **in place and byte-for-byte the same length** to the correct
literal for this machine — so no `app.asar` offset or size changes, and the symlink-race
hardening (`O_DIRECTORY | O_NOFOLLOW` as *values*) is preserved. Backs up `app.asar` first, and
zeroes the matching V8 compile-cache header so the patched source is actually used (V8 keys its
cache check on source length). Re-run after each Claude Desktop update. On builds that already
carry the upstream fix (≥ 1.44121) it reports `upstream-fixed` and exits 0 without writing.

```sh
claude-desktop-patch-oflags          # quit Claude Desktop first
claude-desktop-patch-oflags --check  # read-only status: already-patched | upstream-fixed |
                                     #   needs-patch (N site(s)) | unknown-layout
# restore:  cp app.asar.prepatch-<stamp> app.asar
```

### `bin/claude-desktop-preseed-cli`
The app's own CLI downloader hits the same `O_DIRECTORY` bug before any network I/O. This does
what the app would have: reads the version + checksum the installed app is pinned to, fetches
that exact Claude Code build from Anthropic's server, verifies the sha256, and installs it where
the app looks (`~/.config/Claude/claude-code/<version>/claude`). Re-run after updates.

```sh
claude-desktop-preseed-cli
# then restart Claude Desktop
```

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

## License

MIT — see [`LICENSE`](LICENSE). These are original fix tools and carry no Anthropic code.
