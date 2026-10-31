# openomsi-bin

AUR binary package for [openOMSI](https://github.com/openOMSI-org/openOMSI) - OMSI 2 recreated
from scratch in Rust. Tracks upstream releases and pushes the PKGBUILD to AUR every 12 hours.

Nothing is compiled here: the package repackages the `openOMSI-<version>-linux-{x64,arm64}.zip`
artifacts published by upstream.

## Layout

| Path | Purpose |
|---|---|
| `PKGBUILD.tmpl` | Package template; `@VERSION@` and `@SHA256_*@` are substituted in CI |
| `.github/workflows/aur-update.yml` | Release tracker: detects a new upstream tag, regenerates `PKGBUILD` and `.SRCINFO`, pushes to AUR |
| `LICENSE` | Upstream AGPL-3.0 text, shipped in the package |

`PKGBUILD` and `.SRCINFO` are generated artifacts and are not committed here; they live in the
[AUR repository](https://aur.archlinux.org/packages/openomsi-bin).

## How the automation works

1. Reads the latest tag from `openOMSI-org/openOMSI`.
2. Compares it with the version currently on AUR; stops if equal (`force` input overrides).
3. Takes the SHA256 of both Linux zips from the release assets, falling back to the release's
   `SHA256SUMS` file when the API reports no digest.
4. Substitutes the values into `PKGBUILD.tmpl`, regenerates `.SRCINFO` with `makepkg` inside an
   `archlinux:latest` container, and pushes `PKGBUILD`, `.SRCINFO` and `LICENSE` to
   `ssh://aur@aur.archlinux.org/openomsi-bin.git`.
5. Opens an issue labeled `update-failed` if any of the above fails.

The job can be triggered by hand with **Actions → Update AUR Package → Run workflow**, optionally
with an explicit `version` or `force: true`.

### Required secret

`AUR_SSH_PRIVATE_KEY` - private key whose public half is registered on the AUR account
(`https://aur.archlinux.org` → My Account → SSH Public Key). AUR allows one key per account, so
this is the same key other `-bin` packages of this account use.

## Package notes

- Installed to `/usr/lib/openomsi/`, with `/usr/bin/{openomsi,openomsi-launcher,openomsi-launcher-app}`
  symlinks. The launcher is located as a sibling of the game binary, not through `PATH`.
- Upstream binaries ship with debug info (~570 MB). `makepkg` strips them during packaging
  (~120 MB installed); the stripped engine was verified to start and report its version.
- `openOMSI` normally keeps mods, screenshots and its self-updater beside the program, which is
  read-only under `/usr`. Updates therefore come from `pacman`, and a writable content folder has to
  be pointed at with `OMSI_CONTENT` (or `--content`/settings). Installing the upstream zip into a
  writable folder stays the alternative for users who want the self-updating portable layout.
- `libsteam_api.so` is redistributed as shipped by upstream.

## Manual build

```bash
sed -e 's/@VERSION@/0.3.45/' \
    -e 's/@SHA256_X86_64@/<sha256>/' \
    -e 's/@SHA256_AARCH64@/<sha256>/' \
    -e "s/@SHA256_LICENSE@/$(sha256sum LICENSE | cut -d' ' -f1)/" \
    PKGBUILD.tmpl > PKGBUILD
makepkg --printsrcinfo > .SRCINFO
```
