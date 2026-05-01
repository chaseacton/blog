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
2. Repository **Settings → Pages**: **Build and deployment** source = **GitHub Actions**.
3. **Settings → Pages → Custom domain**: enter `chaseacton.com`. Commit `static/CNAME` (already present) helps Pages keep the domain across builds.
4. At your DNS provider, add GitHub Pages [apex records](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain) for `chaseacton.com`, then enable **Enforce HTTPS** once the certificate is ready.

Workflow file: `.github/workflows/hugo.yml` (Hugo Extended, checkout with submodules).
