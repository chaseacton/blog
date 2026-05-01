# chaseacton.com

Hugo blog using [hugo-coder](https://github.com/luizdepra/hugo-coder), deployed to **GitHub Pages** with custom domain **chaseacton.com**.

## Local development

Install [Hugo Extended](https://gohugo.io/installation/) (same major version as CI, see `.github/workflows/hugo.yml`), then:

```bash
git submodule update --init --recursive
hugo server --buildDrafts
```

Open `http://localhost:1313/`.

## Production build

```bash
hugo --minify
```

Output is in `public/`.

## GitHub Pages setup

1. Create a repository on GitHub and push this project (include submodules: `git push` after first push may need `git submodule update --init` on clones).
2. Repository **Settings → Pages → Build and deployment**: set **Source** to **GitHub Actions** — not “Deploy from a branch.” This site is **Hugo**, not Jekyll. If Source is left on a branch, GitHub runs **Jekyll** on your Markdown and fails on Jinja2-style `{%` / `{%-` in posts (Liquid “Unknown tag” errors).
3. After the first successful **Actions** run, open the green check → **deploy** / **pages-build-deployment** and confirm the site URL. **Settings → Pages → Custom domain**: enter `chaseacton.com`. The built site includes **`static/.nojekyll`** so Pages will not try to Jekyll-process the published HTML.
4. At your DNS provider, add GitHub Pages [apex records](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain) for `chaseacton.com`, then enable **Enforce HTTPS** once the certificate is ready.

Workflow file: `.github/workflows/hugo.yml` (Hugo Extended, checkout with submodules).
