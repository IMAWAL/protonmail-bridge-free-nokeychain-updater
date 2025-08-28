# protonmail-bridge-free-nokeychain-updater

GitHub Actions workflow that monitors the upstream Proton Bridge fork (mnixry/proton-bridge) and automatically updates the AUR package `protonmail-bridge-free-nokeychain` when a new upstream commit changes the internal version.

## What it does
- Schedules (every 6 hours) or manual dispatch.
- Uses an AUR SSH private key (stored as secret `AUR_SSH_PRIVATE_KEY`) to clone and push to the AUR repo via `ssh://aur@aur.archlinux.org/protonmail-bridge-free-nokeychain.git`.
- Derives a pkgver of the form `X.Y.Z.r0.g<shortrev>` from the upstream Makefile + commit.
- Updates `PKGBUILD` (and resets `pkgrel` to 1) when upstream version changes.
- Builds the package in a clean Arch Linux container (non-root build user), regenerates `.SRCINFO`, optionally installs the built artifact for a sanity check, and pushes changes back to AUR.

## Requirements
1. A valid AUR SSH private key with upload rights to the package repo; add it in repository Settings -> Secrets -> Actions as `AUR_SSH_PRIVATE_KEY`.
2. The AUR repository (`protonmail-bridge-free-nokeychain`) must already exist and contain a working `PKGBUILD` and related files.
3. GitHub Actions must be enabled for this repository.

## Secrets
| Name | Purpose |
|------|---------|
| `AUR_SSH_PRIVATE_KEY` | Private key corresponding to your registered AUR public key |

The key should be an OpenSSH private key (beginning with `-----BEGIN OPENSSH PRIVATE KEY-----`). Do not add passphrase-protected keys unless you also configure an `SSH_PASSPHRASE` handling step (not currently implemented).

## Workflow Overview
The workflow file lives at `.github/workflows/update-aur.yml`.

Steps:
1. Checkout repository (only contains the workflow itself; packaging is pulled from AUR).
2. Configure SSH (adds host key + private key to agent).
3. Clone the AUR package repo.
4. Clone upstream (shallow) and compute new version.
5. If version changed, patch `PKGBUILD` (reset `pkgrel`).
6. If changes present (or manually forced), tar the repo and run an Arch container build:
   - Regenerate `.SRCINFO` using `makepkg --printsrcinfo`.
   - Build with `makepkg -sf` (dependencies installed as needed).
7. Commit and push modifications to `master` branch of AUR repo.

## Forcing a Run
Use the manual "Run workflow" button and set `force=true` to force rebuild/push even without detected upstream change.

## Extending
Potential enhancements:
- Auto-increment `pkgrel` on local-only modifications (patch/service file changes).
- Add linting via `namcap`.
- Cache Go module downloads between runs for speed.
- Notify via matrix/Discord/webhook on failures.

## Local Testing
You can simulate the version extraction locally:
```bash
grep -E 'BRIDGE_APP_VERSION\?=' Makefile | sed -E 's/.*= *([^ ]+).*/\1/'
```
Then construct: `<version>.r0.g$(git rev-parse --short HEAD)`.

## License
GPL-3.0-or-later. See `LICENSE`.

---
This project is not affiliated with Proton AG.
