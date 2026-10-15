# euony.me website

Static site built with [Hakyll](https://jaspervdj.be/hakyll/), served by GitHub Pages.

## Layout

- `hakyll` branch — sources: `posts/`, `templates/`, `css/`, `site.hs`.
- `main` branch — generated HTML that GitHub Pages serves (has the `CNAME`).
- `_site/` — git submodule checked out at `main`; Hakyll writes its output here.

So publishing = build into `_site`, commit+push `_site` (→ `main`),
then bump the submodule pointer on `hakyll`.

## Toolchain

`cabal`/`ghc` aren't on the system PATH; get them via nix, e.g.

```sh
nix shell nixpkgs#ghc nixpkgs#cabal-install
cabal build          # produces the `site` executable
```

If nothing in `site.hs` changed, the previously built binary works as-is:

```sh
SITE=$(cabal list-bin site)   # or: dist-newstyle/build/x86_64-linux/ghc-9.8.4/euonyme-0.1.0/x/site/build/site/site
```

## Write & preview

```sh
$SITE watch          # http://127.0.0.1:8000, rebuilds on change
```

Do NOT use `$SITE clean` / `$SITE rebuild`: they delete `_site/` including
the submodule's `.git` link (recover with `git submodule update --init`).

## Publish

```sh
# 1. commit sources on the hakyll branch
git add posts/ css/ templates/ site.hs
git commit -m "chore: ..."

# 2. build into _site
$SITE build

# 3. commit & push the generated site (this is what goes live)
cd _site
git add -A
git commit -m "chore: ..."
git push origin main
cd ..

# 4. record the new _site commit in the sources branch and push it
git add _site
git commit -m "chore: update _site"
git push origin hakyll
```

GitHub Pages picks up the push to `main` within a minute or two.
