# ketchup-pkg

The `[ketchup-pkg]` pacman repository for [Ketchup](https://github.com/ANANDHUAJITH/ketchup) —
the small set of first-party packages (`ketchup-tools`, `ketchup-branding`) that let
Ketchup's own tools and artwork update with `pacman -Syu` instead of being baked
into the ISO, plus the `release.yml` manifest that drives major-train bumps in
`ketchup-upgrade`. Background and design rationale: see
[`docs/OWN-REPO.md`](https://github.com/ANANDHUAJITH/ketchup/blob/main/docs/OWN-REPO.md)
and [`docs/UPDATES.md`](https://github.com/ANANDHUAJITH/ketchup/blob/main/docs/UPDATES.md)
in the main repo.

## Layout

```
pkg-ketchup-tools/      PKGBUILD for ketchup-tools (CLI tools, systemd units, hooks)
pkg-ketchup-branding/   PKGBUILD for ketchup-branding (wallpapers, icons, Plymouth, Calamares)
release.yml             version/codename read by ketchup-upgrade for major-train bumps
.github/workflows/      CI: builds both packages in an archlinux container and
                         publishes the signed-or-unsigned repo db to GitHub Pages
```

Each `PKGBUILD` pulls the actual files from the main `ketchup` repo's
`archiso/airootfs` tree at build time (`source=` is a git clone of that repo),
so this repo never needs a copy of those files checked in — it just packages
them.

## First-time setup

1. Push this directory to a new GitHub repo named **`ketchup-pkg`** under your
   account (adjust the `url=` / clone URLs in both `PKGBUILD`s first if your
   fork isn't `ANANDHUAJITH/ketchup`):

   ```bash
   cd ketchup-pkg
   git remote add origin git@github.com:<you>/ketchup-pkg.git
   git add -A
   git commit -m "Initial ketchup-pkg repo"
   git branch -M main
   git push -u origin main
   ```

2. In the new repo's **Settings → Pages**, set the source to **GitHub
   Actions** (the included workflow deploys there automatically on every
   push to `main`).

3. Point installs at it. `archiso/pacman.conf` and the installed system's
   `/etc/pacman.conf` already carry:

   ```ini
   [ketchup-pkg]
   SigLevel = Required DatabaseOptional
   Server = https://<you>.github.io/ketchup-pkg/$arch
   ```

   Update `<you>` to match your GitHub username/org, and keep
   `/etc/ketchup/upgrade.conf`'s `KETCHUP_PKG_REPO_NAME` in sync with the
   section name above.

4. Push to `main` (or run the workflow manually) to build and publish the
   first `ketchup-pkg.db.tar.gz` + packages. Until then, pacman just logs a
   harmless "could not sync" warning for `[ketchup-pkg]`.

## Signing (recommended before real use)

The workflow currently publishes an **unsigned** repo
(`SigLevel = Required DatabaseOptional` tolerates that, but it's weaker than
it could be). To sign:

- Generate a GPG key offline, export the private key, and store it as a
  GitHub Actions secret (e.g. `KETCHUP_PKG_GPG_KEY`) with environment
  protection on the `github-pages` environment.
- Import it in the workflow before the `repo-add` step and pass
  `repo-add --sign` / `makepkg --sign` with `GPG_TTY` / `--batch` handling
  for a headless signer.
- Publish the public key (`ketchup-pkg.gpg`) alongside the repo and document
  `pacman-key --add ketchup-pkg.gpg && pacman-key --lsign-key <KEYID>` for
  installs.

## Bumping a release train

Edit `release.yml`:

```yaml
version: "27.0"
codename: "Habanero"
```

and push. `ketchup-upgrade` compares this file's `version` against the
installed system's `/etc/ketchup-release` and shows the major-train banner
the next time it runs.

## Adding a new package

1. `mkdir pkg-<name>` and write a `PKGBUILD` there (copy an existing one as a
   template — most of Ketchup's own packages just lift files out of
   `archiso/airootfs` in `package()`).
2. Add a `makepkg` + copy step for it in
   `.github/workflows/build-repo.yml` next to the existing two.
3. Push — the workflow rebuilds the whole repo database on every push to
   `main`.
