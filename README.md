# Sovagram Desktop

Unofficial fork of [Telegram Desktop](https://github.com/telegramdesktop/tdesktop) with:

- **VLESS proxy support** via a bundled [sing-box](https://github.com/SagerNet/sing-box) sidecar.
- **Hide round video messages** UI toggle in Advanced settings.
- **Sovagram branding** — renamed executable, custom logo, GPL attribution in About.
- **GitHub Releases updater** — checks this repo for new builds on startup.

Based on Telegram Desktop by Telegram FZ-LLC. Licensed under GPLv3 (see `tdesktop/LEGAL`).
Comes with **ABSOLUTELY NO WARRANTY**, to the extent permitted by applicable law.

## Repository layout

```
.
├── .github/workflows/        our CI (build, sync-upstream, release)
├── .sync/patches.txt         patch apply order
├── patches/NN-<name>/*.patch git-am patches applied to tdesktop/ in order
├── resources/logo.png        source icon (scaled to all sizes in CI)
├── tdesktop/                 git submodule, pinned to an upstream commit
├── SYNC.md                   how upstream sync works
└── README.md
```

Nothing here except the submodule pointer is upstream code — everything in this repo is our delta.

## Build locally

```bash
git clone --recurse-submodules https://github.com/melnoff/sovagram-desktop.git
cd sovagram-desktop
# Apply patches onto the pinned tdesktop:
cd tdesktop
for p in $(grep -Ev '^\s*(#|$)' ../.sync/patches.txt); do
  git am --3way ../patches/$p/*.patch
done
# Copy the logo overlay:
cp ../resources/logo.png Telegram/Resources/art/logo.png
# Generate icon sizes (requires ImageMagick):
# ...see .github/workflows/build.yml for the exact loop
# Then configure + build per tdesktop's normal Windows/Linux/macOS build docs.
```

For CI-equivalent Windows build see `.github/workflows/build.yml`.

## See also

- `SYNC.md` — how the upstream-tracking workflow works and how to add / remove patches.
- Original Telegram Desktop source: https://github.com/telegramdesktop/tdesktop
