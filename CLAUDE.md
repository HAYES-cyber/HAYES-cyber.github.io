# Security research blog — project context

Personal security-research portfolio for **Bryce Alan Hayes** (GitHub: [HAYES-cyber](https://github.com/HAYES-cyber)), built to support an application to a certification/network program that reviews CVE records, bug bounty history, open-source tools, and education as evidence of active research.

- Hugo static site, custom layouts under `layouts/`, no external theme.
- Deployed via `.github/workflows/hugo.yml` to GitHub Pages at repo `HAYES-cyber/HAYES-cyber.github.io` → https://hayes-cyber.github.io/, branch `main`.
- Separate GitHub profile README template lives in `github-profile-readme/README.md`, meant for a standalone `HAYES-cyber/HAYES-cyber` repo (not yet created).

## Content rule — do not skip

`data/cves.yaml`, `data/bounties.yaml`, `data/tools.yaml`, and the About/Contact pages must only contain **real, independently verifiable** information:
CVE IDs that resolve on cve.org, bug bounty reports that resolve on the actual platform (HackerOne/YesWeHack/Immunefi) and credit this person, tool repos that actually exist and are maintained by them, and real degrees/institutions.

Never invent or embellish an entry to make the portfolio look more active. When the user supplies new CVE numbers, report links, or tool repos, check that the links actually resolve and credit them before adding them — don't just transcribe dictated text. Placeholder entries (e.g. `CVE-0000-00000`) must stay obviously fake until replaced with checked, real data.

## Local build

```sh
hugo server -D        # preview with drafts
hugo --gc --minify    # production build, output in public/ (gitignored)
```
