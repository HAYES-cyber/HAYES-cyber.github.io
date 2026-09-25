# [username].github.io — security research blog

Static site built with [Hugo](https://gohugo.io), deployed to GitHub Pages via GitHub Actions. Custom theme lives entirely in this repo under `layouts/` and `static/` — no external theme dependency.

## Structure

```
.
├── archetypes/default.md      # `hugo new` scaffold for posts
├── content/
│   ├── _index.md              # homepage intro (optional)
│   ├── about/_index.md
│   ├── cve/_index.md          # renders data/cves.yaml
│   ├── research/              # blog posts live here
│   ├── talks/_index.md
│   └── contact/_index.md
├── data/cves.yaml             # single source of truth for CVE table
├── layouts/                   # custom templates (base, home, single, list, cve table)
├── static/
│   ├── css/main.css           # light/dark theme via CSS variables
│   ├── js/theme.js            # theme + mobile-nav toggle
│   └── .well-known/security.txt
├── .github/workflows/hugo.yml # build + deploy on push to main
└── hugo.yaml                  # site config
```

## Prerequisites

- [Hugo extended](https://gohugo.io/installation/) — use the **extended** binary (required for some features) — version matching `HUGO_VERSION` in `.github/workflows/hugo.yml`.
- Git, and a GitHub account.

## 1. Configure

Edit `hugo.yaml`:

- `baseURL` → `https://<your-username>.github.io/`
- `title`, `params.author`, `params.handle`, `params.tagline`
- `params.email`, `params.pgpFingerprint`
- `params.github`, `params.linkedin`, `params.hackerone`, `params.bugcrowd`

Replace bracketed placeholders `[...]` in:

- `content/about/_index.md`
- `content/contact/_index.md`
- `content/talks/_index.md` (or delete this section + its menu entry in `hugo.yaml` if not applicable)
- `static/.well-known/security.txt`

## 2. Add your CVEs

Edit `data/cves.yaml`. Remove the placeholder `CVE-0000-00000` entry and add one block per disclosure:

```yaml
- id: "CVE-YYYY-NNNNN"
  product: "Vendor Product X.Y"
  type: "Vulnerability class"
  cvss: "0.0 (vector string)"
  cve_url: "https://www.cve.org/CVERecord?id=CVE-YYYY-NNNNN"
  advisory_url: "https://vendor.example/security/advisory-id"
  writeup_url: "/research/your-post-slug/"
  date: "YYYY-MM-DD"
```

Only add entries after the vendor's coordinated disclosure embargo has lifted, and only with a real `cve_url` / `advisory_url` you can verify.

## 3. Write a post

```sh
hugo new content/research/my-new-finding.md
```

This scaffolds from `archetypes/default.md` with `draft: true`. Edit the front matter (`title`, `date`, `tags`, `summary`, `toc`) and body, then set `draft: false` when ready to publish.

## 4. Preview locally

```sh
hugo server -D
```

`-D` includes draft content. Open `http://localhost:1313`. The server live-reloads on file changes.

To preview exactly as production will build it (no drafts, minified):

```sh
hugo --gc --minify
hugo server --environment production
```

## 5. Deploy to GitHub Pages

1. Create a repository named **`<your-username>.github.io`** on GitHub (this exact name is required for a user site).
2. Push this project to it:
   ```sh
   git init
   git add .
   git commit -m "Initial site"
   git branch -M main
   git remote add origin https://github.com/<your-username>/<your-username>.github.io.git
   git push -u origin main
   ```
3. In the repo, go to **Settings → Pages → Build and deployment** and set **Source** to **GitHub Actions**.
4. Push to `main` (or re-run the workflow from the **Actions** tab). `.github/workflows/hugo.yml` builds the site with Hugo and deploys it via `actions/deploy-pages`.
5. Your site will be live at `https://<your-username>.github.io/`.

### Custom domain (optional)

1. Add a `static/CNAME` file containing just your domain, e.g. `blog.example.com`.
2. At your DNS provider, add a `CNAME` record pointing your subdomain at `<your-username>.github.io`, or `A`/`AAAA` records to [GitHub's Pages IPs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site) for an apex domain.
3. In **Settings → Pages**, set the custom domain and enable **Enforce HTTPS** once the certificate is issued (can take a few minutes to hours).
4. Update `baseURL` in `hugo.yaml` to the custom domain.

## Notes

- RSS feeds: sitewide at `/index.xml`, per-section at `/research/index.xml`.
- Code highlighting is Hugo's built-in Chroma renderer (server-side, no JS) — style set in `hugo.yaml` under `markup.highlight`.
- Table of contents: set `toc: true` in a post's front matter.
- Tags: set `tags: [...]` in front matter; tag archive pages are generated automatically at `/tags/<tag>/`.
