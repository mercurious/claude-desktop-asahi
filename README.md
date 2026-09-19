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
Claude Desktop 1.44121 (September 2026)**, which retired the patch. Then
[#1](https://github.com/mercurious/claude-desktop-asahi/issues/1) pointed out what retires most of
the rest: a third-party project already publishes a **signed DNF repo carrying aarch64 packages**,
so `sudo dnf upgrade` keeps Claude Desktop current on Fedora Asahi. That was installed and
verified on the reference box on 2026-09-19 — see
[Alternative: the third-party DNF repo](#alternative-the-third-party-dnf-repo).

So the claim this section used to lead with — *there is no other way to update a user-local
install* — is **no longer true**, and the table below no longer says it. What survives is a
narrower reason to exist: the DNF package is a **repackage** of Anthropic's `.deb`, carrying a
launcher and five patches to `app.asar`, signed by a third party. `claude-desktop-update` is now
the option you pick when that distinction matters to you, not the only road out.

| Tool | Why it exists | Status | Retires when |
|---|---|---|---|
| `claude-desktop-update` | Anthropic still ships no Linux update path — 2.2553.1 still logs `[updater] Linux: in-app updater off (updates via apt)`, and Fedora Asahi has no apt. This fetches **Anthropic's own** build-pinned `.deb`, into a user-local install, with a timestamped backup and rollback. | **Optional — no longer the only option.** A third-party DNF repo now covers the same gap. Keep this if you want unmodified Anthropic bytes, a user-local (non-root) install, or a rollback you control. | Anthropic ships a Linux update path this box can use (in-app updater enabled on Linux, or an official rpm / Flatpak / AppImage channel) — **or** you accept the third-party repackage and move to `dnf`. |
| `claude-desktop-patch-oflags` | Corrected the x86_64 `O_DIRECTORY` / `O_NOFOLLOW` values the app used on aarch64 (see *History*). | **Redundant.** Detects the upstream fix, reports `upstream-fixed`, exits 0, writes nothing — confirmed against 2.2553.1. | Now, for any current build. Keep it only while you might install or roll back to a pre-1.44121 build (e.g. a `claude-desktop.bak-*` taken before the fix); it still patches those. |
| `claude-desktop-preseed-cli` | The app's own CLI downloader died on the same bug before any network I/O, so this fetched the pinned CLI for it. | **Blocker gone.** Retained as a convenience: `claude-desktop-update` uses it to fetch the pinned CLI and validate the manifest *before* a swap. | Optional today. |

**What would retire the repo completely:** switching this box to the DNF repo. All three tools
then have no job on current builds — `dnf` does the updating, the bug is fixed upstream, and the
app fetches its own CLI. The repo stays up for two cases that repackaging does not serve: wanting
Anthropic's bytes unmodified, and rolling back to a pre-1.44121 build.

## Alternative: the third-party DNF repo

[aaddrick/claude-desktop-debian](https://github.com/aaddrick/claude-desktop-debian) (led by
[@sabiut](https://github.com/sabiut) since September 2026) repackages Anthropic's official Linux
`.deb` into formats Anthropic doesn't ship, and serves them from signed APT and DNF repos —
including **aarch64**. Raised in [#1](https://github.com/mercurious/claude-desktop-asahi/issues/1)
by [@steals](https://github.com/steals).

```sh
sudo curl -fsSL https://pkg.claude-desktop-debian.dev/rpm/claude-desktop-unofficial.repo \
  -o /etc/yum.repos.d/claude-desktop-unofficial.repo
sudo dnf install claude-desktop-unofficial
```

Updates then arrive with `sudo dnf upgrade`, like any other package.

**Verified on this box** — Fedora Asahi Remix 44, aarch64, 16 KB pages, 2026-09-19:

- The `aarch64` repo tree is real and current; it served
  `claude-desktop-unofficial-2.2553.1-3.2.4.fc42.aarch64` the same day Anthropic's feed offered
  2.2553.1. Built for `fc42`, installs clean on `fc44` (its only dependency is `/bin/sh`).
- The GPG key dnf imports — `87494CF73ACC0F23AA9557B86E29E413B912E0F1` — matched the key fetched
  independently beforehand.
- Their mirrored copy of Anthropic's `arm64` `.deb` is the same size to the byte (166,755,520) as
  the one Anthropic's own feed serves, consistent with repackaging rather than rebuilding.
- `claude-desktop-patch-oflags --check` against the packaged `app.asar` reports `upstream-fixed`,
  so the arm64 flag table is present in the shipped build.
- `claude-desktop-unofficial --doctor` passes on Asahi, correctly identifying Wayland/KDE,
  Electron 44.2.0, the keyring, and the running instance's lock. It also reports *"Version: in
  sync with the official pool."*
- It **launches on a 16 KB-page kernel**: GPU process on Wayland, network service, renderers and
  audio service all came up, the app reached the claude.ai login page, and the log shows zero
  `[error]` lines and no `EINVAL` — tested against an isolated config so nothing live was touched.
  (One benign `vaInitialize failed` warning: there is no VA-API on Asahi, same as the official
  build.)

**What you trade for it:**

- **Not Anthropic's bytes.** The shipped `app.asar` carries five patches (virtiofsd path
  resolution, the Linux org-plugins path, tray-icon selection, Quick Entry focus, and an opt-in
  bubblewrap Cowork backend) plus their launcher and `--doctor`. Sensible fixes, openly documented
  — but a modified app, signed by someone other than Anthropic.
- **Trust-on-first-use.** The signing key's fingerprint is served by the same host it
  authenticates and isn't published in their repo, so there's nothing to check it against. Usual
  for third-party repos; still worth naming.
- **Root, system-wide** (`/usr/lib/claude-desktop-unofficial`) rather than user-local.
- **Thin aarch64 field testing.** Per release, the aarch64 RPM sees tens of downloads against
  thousands for x86_64, and the arm64 AppImage single digits. The verification above is one more
  data point on Asahi specifically, not a track record.
- Both installs share `~/.config/Claude`, so only one can run at a time — but they sit at
  different paths and coexist fine while you decide.

## The upstream fix (≥ 1.44121)

The app now carries a per-architecture table of the two flags and corrects `fs.constants` at
startup whenever the runtime's values don't match `process.arch`:

```js
S = { arm64: { O_DIRECTORY: 16384, O_NOFOLLOW: 32768 },   // 0o40000, 0o100000 — correct for aarch64
      x64:   { O_DIRECTORY: 65536, O_NOFOLLOW: 131072 } } // 0o200000, 0o400000
```

Every private-directory `open()` uses the corrected constants. When the correction fires, the app
**logs it at error level and reports it to Sentry**:

```
[error] fs.constants open flags on arm64 carry x64/ia32's values
        { arch: 'arm64', observedArch: 'x64/ia32',
          expected: { O_DIRECTORY: 16384, O_NOFOLLOW: 32768 },
          observed: { O_DIRECTORY: 65536, O_NOFOLLOW: 131072 } }      // OpenFlagAbiMismatchError
```

That line is the fix working, not a failure: the app noticed the ABI mismatch and corrected it.
The proof is what is absent — no `EINVAL` lines, and the private-dir subsystems (CLI download,
scheduled tasks, plugin and skills sync) all complete.

**It has since gone quiet.** On this box the line appeared on five of eight 1.44121.4 launches
(2026-09-03 to 09-07) and has not appeared since — not on 1.49585.0, 1.52386.x, or 2.2553.1. The
guard is still in the bundle (`OpenFlagAbiMismatchError` is present in every build checked), so it
isn't that upstream removed it: the runtime simply stopped reporting x64 constants on arm64, and
there is nothing left to correct. Both layers of the fix now hold, and a silent log is the
expected state.

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

If you would rather let the package manager do this, see
[Alternative: the third-party DNF repo](#alternative-the-third-party-dnf-repo) — fewer moving
parts, at the cost of running a repackaged `app.asar` signed by someone other than Anthropic.

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

Developed and maintained with Anthropic Claude — Fable 5, and Opus 5 for the 2026-09-19 DNF-repo
verification and README revision.


## License

MIT — see [`LICENSE`](LICENSE). These are original fix tools and carry no Anthropic code.
