# Horizon on iPhone, built without a Mac

Run [Horizon](https://github.com/ludovic111/horizon) (Minecraft: Java Edition 26.3, rewritten in Rust)
on your iPhone or iPad, using **only a Linux/Windows/macOS PC, a free GitHub account and a sideloading tool**.
The iOS port itself comes from [`woke/horizon-duo`](https://github.com/woke/horizon-duo). This repo only
documents how to get it built on GitHub's macOS runners instead of your own Mac.

> **Not an official Minecraft product.** Not approved by or associated with Mojang or Microsoft.
> This repo contains **no Minecraft files, no game assets and no IPA**. You build everything from
> **your own legally bought copy** of Minecraft: Java Edition. Never share your prepared files or your IPA:
> they contain material derived from Minecraft. This guide is not legal advice; read the
> [Minecraft EULA](https://www.minecraft.net/en-us/eula) yourself.

## How it works

```
your PC                       your PRIVATE repo               your PUBLIC fork
prepare game files   ──zip──▶ release "assets"  ──download──▶  GitHub Actions (macOS)
(Horizon launcher)                                              builds Horizon.ipa
                                          release "ipa"  ◀──────┘
                                              │
                                              ▼
                                    sideload to your iPhone
```

Why two repos: the game files are derived from Minecraft and must stay private. A fork of a public repo can't
be made private, so the **public fork** holds only the workflow and code, and the **private repo** holds your files
and the finished IPA. Standard GitHub-hosted runners are free for public repos (check your plan); private repos
are billed per minute, and macOS minutes cost more.

## What you need

- Minecraft: Java Edition 26.3, bought and played once
- The [Horizon launcher](https://playhorizon.dev) on your PC, to prepare the game files
- A GitHub account and the [GitHub CLI](https://cli.github.com) (`gh`), logged in with `gh auth login`
- A sideloading tool (see step 7). A free Apple ID works, but the app must be reinstalled every 7 days
- An iPhone or iPad on iOS 16 or newer (the build targets iOS 16.0)

## Steps

### 1. Prepare the game files on your PC

In the Horizon launcher, run **Prepare** (listed as "Prepare resources" on the Play page). It builds the game's
files from your own copy of Minecraft. The result is a folder named after the game version (`26.3`) that
contains `horizon-resources.json`. Find it:

```bash
# Linux
find ~/.local/share/horizon-launcher -name horizon-resources.json
```

On Windows and macOS, search your launcher data folder for the same file name. The folder that contains it is
the one to zip (about 700 MB). The zip must have that folder, called `26.3`, at its top level:

```bash
cd <the folder that CONTAINS 26.3>
zip -r ~/horizon-assets.zip 26.3/
unzip -l ~/horizon-assets.zip | grep horizon-resources.json   # must show 26.3/horizon-resources.json
```

Look inside before you upload anything. It should only hold game resources (`horizon-resources.json`,
`assets/`, ...). Your launcher's `accounts.json` and `settings.json` live elsewhere and must never be uploaded.

### 2. Create a private repo for your files

```bash
gh repo create horizon-assets --private --add-readme
gh release create assets ~/horizon-assets.zip --repo YOUR-NAME/horizon-assets \
  --title "Prepared assets" --notes "horizon-client 26.3"
gh repo view YOUR-NAME/horizon-assets --json visibility    # must say PRIVATE
```

The upload takes a few minutes and shows no progress bar. `--add-readme` matters: a release can't be created
in an empty repo (`HTTP 422: Repository is empty`).

### 3. Fork the iOS port (public)

Fork [`woke/horizon-duo`](https://github.com/woke/horizon-duo) on GitHub and use the branch `ios-iphone-duo`
(set it as the default branch of your fork, otherwise the "Run workflow" button doesn't show up).
The fork contains only code, so public is fine.

### 4. Create an access token for the private repo

GitHub > Settings > Developer settings > Personal access tokens > **Fine-grained tokens**:

- Repository access: **only** `horizon-assets`
- Permissions: **Contents: Read and write**
- Short expiry (30 days is plenty)

Store it as a secret in your fork without printing it anywhere:

```bash
gh secret set ASSETS_TOKEN --repo YOUR-NAME/horizon-port
```

(paste the token at the prompt). Never paste the token anywhere else, and delete it when you are done.

### 5. Add the workflow

Copy [`workflow/build-ios.yml`](workflow/build-ios.yml) to `.github/workflows/build-ios.yml` in your fork and push.
Nothing needs editing: the workflow reads `<your GitHub name>/horizon-assets` from the repo owner. If your private
repo has a different name, change `ASSETS_REPO` at the top of the file.

It downloads your assets from the private repo, runs `packaging/ios/package.sh device`, and uploads the
finished `Horizon.ipa` to a release named `ipa` in the private repo. The IPA is never published publicly.
It is signed ad hoc; your sideloading tool signs it again with your Apple ID during install.

### 6. Build

Fork > Actions > **Build iOS IPA** > Run workflow (or `gh workflow run build-ios.yml --ref ios-iphone-duo`).
In our run the build took under ten minutes.

### 7. Install

Download the IPA from the `ipa` release of your private repo:

```bash
gh release download ipa --repo YOUR-NAME/horizon-assets --pattern Horizon.ipa
```

Then sideload it:

- **Windows or macOS:** [Sideloadly](https://sideloadly.io) or [AltStore](https://altstore.io).
- **Linux:** [Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit): *Sideload IPA* > *Install IPA using Plumesign*.
  Start it with `TMPDIR=~/tmp-sideload ./restore.sh` (create the folder first), because signing needs several GB of
  temporary space and `/tmp` is often a small RAM disk.

On the iPhone: turn on **Developer Mode** (Settings > Privacy & Security), then trust your Apple ID under
Settings > General > VPN & Device Management, then start the app. The first start takes a moment while
the game loads its resources.

## Status

Singleplayer runs on the iPhone with Pocket Edition touch controls. Multiplayer, LAN and Play with Friends
are untried on iOS. This build has no Microsoft sign-in, so it cannot join online-mode servers.

## Troubleshooting

- **Workflow stays queued:** the runner label `xcode-27` (GitHub's runner image with Xcode 27, which the port needs)
  isn't available to your account. Check GitHub's runner-images documentation for the current label and plan limits.
- **`gh release download` fails:** the `ASSETS_TOKEN` secret is missing, expired, or has no access to your assets repo.
- **`test -f .../horizon-resources.json` fails:** your zip has the wrong layout. It needs a top-level `26.3/` folder.
- **`HTTP 422: Repository is empty` on `gh release create`:** the new repo has no commit yet. Create it with `--add-readme`.
- **`gh release view` says "release not found" right after an upload:** check for a draft release in
  `gh api repos/YOUR-NAME/horizon-assets/releases`.
- **`Disk quota exceeded` while signing (Linux):** use `TMPDIR` as shown in step 7.
- **`A device with number ... already exists on this team` (Plumesign):** harmless, the install goes on.
- **App stops opening after 7 days:** normal with a free Apple ID. Reinstall the same IPA.

## Credits

[Horizon](https://github.com/ludovic111/horizon) by Ludovic (MIT licence), and the iOS port by the
[`horizon-duo`](https://github.com/woke/horizon-duo) authors. This guide just connects the pieces.
