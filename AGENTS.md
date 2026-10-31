# AGENTS.md

AUR `-bin` packaging repo for [openOMSI](https://github.com/openOMSI-org/openOMSI). Nothing is
compiled here: the workflow repackages upstream's `openOMSI-<version>-linux-{x64,arm64}.zip`
release artifacts and pushes the result to the AUR.

## Map

| Path | Role |
|---|---|
| `PKGBUILD.tmpl` | Single source of truth for the package. `@VERSION@`, `@SHA256_X86_64@`, `@SHA256_AARCH64@`, `@SHA256_LICENSE@` are substituted in CI |
| `.github/workflows/aur-update.yml` | Cron (12 h) + manual tracker. Detects the upstream tag, renders the PKGBUILD, regenerates `.SRCINFO`, pushes to AUR |
| `LICENSE` | Upstream AGPL-3.0 text; also a PKGBUILD source with its own checksum and shipped in the package |
| `README.md` | User-facing description of the automation and package behaviour |

`PKGBUILD`, `.SRCINFO`, `pkg/` and `src/` are generated locally and are gitignored.

## Landmines

- **AUR rejects non-fast-forward pushes** (`hook declined to update refs/heads/master`). Never try to
  rewrite AUR history; the workflow only ever appends a commit. The branch there is `master`, not
  `main`.
- **The AUR RPC index lags behind a fresh push.** `rpc/v5/info/<pkg>` can return `resultcount: 0`
  for minutes after a package is created. The version check therefore falls back to reading
  `pkgver` from cgit (`cgit/aur.git/plain/PKGBUILD?h=<pkg>`). Do not drop that fallback, and guard
  every lookup with `|| true` - GitHub runs steps with `pipefail`, and a 404 on a package that does
  not exist yet must not fail the run.
- **The install layout is not arbitrary.** Upstream locates its launcher as a *sibling* of the game
  binary ("It belongs in the same folder as the game"), not through `PATH`. Everything therefore
  lives in `/usr/lib/openomsi/` with `/usr/bin/{openomsi,openomsi-launcher,openomsi-launcher-app}`
  relative symlinks. Keep the files together when changing the layout.
- **Stripping is intentional and verified.** Upstream ships unstripped binaries (~570 MB installed);
  makepkg's default strip brings the package to ~117 MB. The stripped engine was checked to start
  and report its version. Do not add `options=('!strip')` without re-testing.
- **Dependencies follow the binaries, not the docs.** Upstream's README names Debian/Fedora
  packages. The `depends` list was derived from the actual `NEEDED` entries of the three ELF files
  (plus the runtime-dlopened graphics libs) and each name verified against the official repos.
  Re-derive it if upstream changes its stack.
- **Identity:** commits in this repo and from CI use `DepsCian <deps2ak@gmail.com>`. The global
  `~/.gitconfig` on the dev host uses a different (CI) identity, so the correct values are pinned
  repo-locally.

## Validating a change

```bash
sed -e 's/@VERSION@/0.3.45/' -e 's/@SHA256_X86_64@/<sha>/' -e 's/@SHA256_AARCH64@/<sha>/' \
    -e "s/@SHA256_LICENSE@/$(sha256sum LICENSE | cut -d' ' -f1)/" PKGBUILD.tmpl > /tmp/PKGBUILD
cd /tmp && makepkg --printsrcinfo          # .SRCINFO must render for both arches
SRCDEST=<dir with the release zip> makepkg -f   # full build, then check pkg/ layout
```

CI can be triggered by hand: **Actions → Update AUR Package → Run workflow**, with an optional
`version` and `force`. `AUR_SSH_PRIVATE_KEY` must stay the key registered on the AUR account (AUR
allows one key per account, shared with the account's other packages).
