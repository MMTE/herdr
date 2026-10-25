# Fork changelog — MMTE/herdr

Session records for work done on this fork. Upstream files (`CHANGELOG.md`,
`docs/next/CHANGELOG.md`) are herdrdev-owned and never edited here.

## 2026-10-25 — Persian/RTL rendering: root cause, fix, upstream attempt, install

**Problem:** herdr 0.9.5 rendered Persian (and all RTL) text broken in iTerm2:
reversed order and unjoined letterforms, while the same text renders correctly
in a plain iTerm2 tab.

### Root cause (byte-verified)

`src/protocol/render_ansi.rs` gates inline run-continuation on
`cell.symbol.is_ascii() && width == 1` at three sites (`write_all_cells`,
`write_changed_cells`, `blit_patch_to`). Every non-ASCII cell forces the next
cell into an explicit CUP, so RTL words never reach the host terminal as a
contiguous run — host-side bidi reordering and Arabic-script shaping (both
applied per contiguous run) can never engage. Captured from a live client:
`printf 'س` `ESC[2;70Hل` `ESC[2;71Hا` `ESC[2;72Hم` … (issue herdrdev/herdr#5118).

### Fix — branch [`bidi-persian`](https://github.com/MMTE/herdr/tree/bidi-persian)

- `18a31d43` fix: keep rtl runs contiguous in ansi blit — new
  `advances_one_column()` predicate (single-scalar, width 1 under **both**
  normal and CJK width tables; EA-ambiguous, multi-scalar clusters, and wide
  cells stay cursor-addressed). Replaces the `is_ascii()` gate at all three
  emission sites. Persian/Hebrew/Arabic now flow as contiguous logical-order
  runs; iTerm2 renders them joined and correctly ordered.
- `038d0e67` test: assert rtl run contiguity across the content span.

### Verification

- `cargo test --bins render_ansi` → 54/54 (3 new) on tdh-dev-01.
- End-to-end pty captures (real pane, real client) on Linux debug and macOS
  release builds: `persian-contiguous=True`; before/after bytes recorded.
- Visual proof in iTerm2 (user + vision read of window screenshots): joined,
  correct RTL order.

### Upstream

- PR [herdrdev/herdr#5558](https://github.com/herdrdev/herdr/pull/5558) opened
  from this branch — **auto-closed** by `kangal-bot`: repo policy forbids
  implementation PRs from accounts not in `.github/APPROVED_CONTRIBUTORS`.
- Evidence comments posted: [#5500 (PR)](https://github.com/herdrdev/herdr/pull/5500#issuecomment-6399019623),
  [#5118 (issue)](https://github.com/herdrdev/herdr/issues/5118#issuecomment-6399020965).
- PR #5500 (barnuri, UAX#9 reorder + BDSM handshake) tested locally on iTerm2:
  **renders Persian joined and correctly ordered** (user-typed phrases +
  screenshot vision analysis). This disproves our #5500 comment's "joining
  still breaks" claim *for iTerm2* (iTerm2 shapes Arabic at paint time over the
  row). A correction comment on #5500 is still owed.
- iTerm2 misclassification note ("no bidi") in #5500 remains valid: iTerm2 has
  its own bidi; it just doesn't answer the BDSM probe.

### Installed state (Mac, this session)

- `~/.local/bin/herdr` = release build of `bidi-persian` @ `038d0e67`
  (23,474,464 bytes). Rollback: `~/.local/bin/herdr.stock-0.9.5`.
- Live handoff via `server.live_handoff` socket API
  (`{"method":"server.live_handoff","params":{"import_exe":…,"expected_protocol":22,"expected_version":"0.9.5"}}`,
  newline-delimited JSON over `~/.config/herdr/herdr.sock`): old server (Oct 20)
  exited cleanly, new server adopted all pane ptys; all agent panes and the
  attached client survived.
- **Warning:** `herdr update` will silently replace this binary with stock on
  the next upstream release. Check herdrdev/herdr#5118 / #5500 first, or
  rebuild this fork from the new tag.

### Build environment created (for future rebuilds)

- tdh-dev-01: rustup 1.96.1 (`~/.cargo`), `build-essential` (sudo apt),
  Zig 0.16.0 (`~/.local/opt/zig-x86_64-linux-0.16.0`, symlink `~/.local/bin/zig`),
  checkout `~/herdr-bidi/herdr`, zig dep cache `~/.cache/zig`.
- Mac: Zig 0.16.0 (`~/.local/opt/zig-aarch64-macos-0.16.0`, symlink
  `~/.local/bin/zig`), zig dep cache `~/.cache/zig`.
- Network trap: codeberg.org (translate-c zig dep) is blocked from both the Mac
  and dev-01. Workaround used: download on nuremberg (Hetzner VPS, reachable),
  place the tarball into the zig global cache as
  `~/.cache/zig/p/<name>-<version>-<hash>.tar.gz` (zig 0.16 keeps tarballs,
  not extracted dirs).
- iTerm2 was granted Screen Recording permission (macOS Settings) during
  screenshot verification.

### Open items

- [ ] Post correction comment on herdrdev/herdr#5500 (joining works on iTerm2).
- [ ] Watch #5500 / #5118 for maintainer response; answer issue-agent follow-ups.
- [ ] Before any `herdr update`: confirm upstream RTL state or re-fork the tag.
- [ ] dev-01 scratch checkout `~/herdr-bidi/herdr` (~3 GB with target/) — keep
      for Linux rebuilds or delete when no longer needed.
