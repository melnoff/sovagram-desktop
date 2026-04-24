# Upstream sync & patch management

This fork tracks `telegramdesktop/tdesktop` via a git submodule pinned to a
specific upstream commit. Our changes live as `git format-patch` files in
`patches/<name>/*.patch` and are applied at build time.

## Layout

```
tdesktop/                     git submodule, pinned to an upstream SHA
patches/
  01-vless/                   each folder is one logical patch set
  02-sovagram-branding/       numeric prefix = canonical apply order
  03-ui-hide-round-videos/
  04-sovagram-updater/
resources/logo.png            source icon, overlaid into tdesktop/ at build
.sync/patches.txt             apply order (authoritative)
```

## Apply order

`.sync/patches.txt` — plain text, one `patches/<name>` per line, applied top
to bottom. Comments start with `#`. To disable a patch temporarily, comment
the line.

The numeric prefixes on folder names (`01-`, `02-`, …) are informational
and should match the order in `patches.txt`. Renumber both when inserting
a new patch into the middle of the stack.

## Branches

- `master` — current working build; tracks the latest upstream stable tag
  that our patches apply cleanly onto.
- `v<N>` (e.g. `v6.7.5`) — snapshot of `master` at the time it was pinned
  to upstream tag `v<N>`. Created automatically by the sync workflow
  right before advancing to a newer tag. Frozen once created.
- `sovagram-YYYYMMDD-<sha>` — release tags created by the release workflow.

## Build flow (what CI does)

1. Checkout this repo with `submodules: recursive` — `tdesktop/` lands at
   the pinned commit.
2. Read `.sync/patches.txt`.
3. For each `<name>` in order:
   - `cd tdesktop && git am --3way ../patches/<name>/*.patch`
4. Copy `resources/logo.png` → `tdesktop/Telegram/Resources/art/logo.png`.
5. Generate all required icon sizes (PNG + `.ico`) with ImageMagick.
6. Build `singbox.dll` (Go + CGO + MinGW) from `tdesktop/Telegram/ThirdParty/singbox_bridge/`.
7. Run `configure.bat qt6 -D SOVAGRAM_BUILD_ID=<sha>` then `cmake --build`.
8. Collect `Sovagram.exe`, `Updater.exe`, `singbox.dll` into the artifact.

## Upstream sync

`.github/workflows/sync-upstream.yml` runs daily at 08:00 UTC and on manual
`workflow_dispatch`. It:

1. Picks the latest stable upstream tag (`v*.*.*`) unless `target_ref` input
   overrides.
2. Skips if the submodule is already at that commit, or refuses if the target
   is behind (won't rewind).
3. Checks out the submodule to the new tag.
4. For each `patches/<name>` in order, runs `git am --3way`. If any fails,
   opens an issue and stops without touching `master`.
5. On success:
   a. Regenerates patches from the applied commits (so stored patches
      carry fresh context lines from the new upstream).
   b. Rewinds the submodule back to the clean upstream tag.
   c. Creates a snapshot branch named after the PREVIOUS upstream ref
      (e.g. `v6.7.5`) pointing at the pre-sync `master` head, so the
      state that was current for each upstream version is preserved.
      Existing snapshot branches are left untouched.
   d. Commits the submodule-pointer bump + refreshed patches and pushes
      to `master`.
6. Triggers the build workflow.

Uses only the built-in `GITHUB_TOKEN`. No PAT required.

## Adding a new patch

Easiest: work inside the submodule.

```bash
cd tdesktop
git checkout -b my-feature
# ... make changes, commit one or more times ...
git format-patch --binary master..HEAD -o ../patches/my-feature/
cd ..
# Add 'my-feature' in the right spot in .sync/patches.txt
git add tdesktop patches/my-feature .sync/patches.txt
git commit -m "feat: my-feature"
git push
```

**Important:** after generating patches, reset the submodule HEAD back to
the pinned commit so the submodule pointer doesn't drift:

```bash
cd tdesktop && git checkout $(git merge-base HEAD origin/dev) && cd ..
# or simply: git submodule update --init --force
```

## Disabling a patch

Comment out its line in `.sync/patches.txt` and commit. It stays on disk
(can be re-enabled), but won't be applied to builds.

## Manual sync

To pin to a specific tag or test against a different branch:

Actions → Sync Upstream → Run workflow → set `target_ref` (e.g. `v6.7.6`).
Use `dry_run: true` to preview without pushing.

## Handling sync conflicts

When sync opens an issue, follow the commands in its body. After pushing
resolved patches, re-run `sync-upstream.yml` manually.

Typical causes:

- Upstream renamed or deleted a file our patch touches — requires rewriting
  the patch to match the new location/name.
- Two of our patches touch the same lines but their order in `patches.txt`
  puts the wrong one first — reorder.
- A patch is no longer needed because upstream merged something similar —
  delete the patch folder and the line.
