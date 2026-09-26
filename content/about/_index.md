---
title: "About"
---

## Background

I'm a security researcher focused on AI-assisted vulnerability research.

I first got hands-on with security during an internship on SecureWorks' threat intelligence team in Atlanta, doing vulnerability research and threat hunting — that's where I built my manual fundamentals: working attack surfaces by hand, reading source, and testing authorization and business-logic paths branch by branch. Alongside that I did authorized vulnerability testing on the side, mostly bug bounty work on my [Bugcrowd profile](https://bugcrowd.com/h/Bryce_Alan_Hayes) — 8 validated vulnerabilities at 100% accuracy, with Hall of Fame recognition on Pantheon and HostGator LATAM Bug Bounty. A lot of my [research write-ups](/research/) are me consolidating what I learned through that period.

What changed things was AI. Once capable AI tooling arrived, I started experimenting — pointing it at real, authorized targets to see how far AI-assisted hunting could actually go — and was genuinely surprised by how well it worked. It turned up real, credited findings: three CVEs in aircheng-org iWebShop-5 (CVE-2026-86665/86666/86667), plus a series of vulnerabilities disclosed through vendor Security Response Centers — Kuaishou SRC (Kling AI) and JD SRC (JD Cloud Lingjing AI). I've written up how I actually use this tooling, end to end, in [AI Coding Agent Workbench](/research/ai-agent-workbench/).

The pace since then has been fast, and I want to keep contributing to security. My goal is to use AI to make more meaningful contributions and to build the tooling that supports that work — I've published some of the tools I'd been maintaining on my [GitHub](https://github.com/HAYES-cyber) (including [promptprobe](https://github.com/HAYES-cyber/promptprobe)), and I plan to build more AI-assisted security tools from here.

My work runs on two tracks: using AI-assisted methodology to accelerate traditional vulnerability research (web application security, authorization bypasses, deserialization/RCE), and studying the security of AI systems themselves — prompt injection, adversarial robustness, RAG data exposure, AI agent authorization boundaries, and supply-chain risk in AI-generated code.

## Research Focus

- AI-assisted vulnerability research: attack-surface mapping, code-audit assistance, and attack-chain reasoning with AI coding agents
- AI/LLM security: prompt injection, adversarial robustness, RAG data exposure, AI agent authorization boundaries, and supply-chain risk in AI-generated code
- Web application security (authorization flaws, business logic vulnerabilities, SSRF)
- Deserialization and RCE chain research
- Vulnerability disclosure coordination (CNA/vendor SRC workflows)
- Security tooling for exploit verification and CTF operations

## Experience

- **Independent security research** — AI-assisted vulnerability discovery: three CVEs (CVE-2026-86665/86666/86667) via VulDB's CNA process, and vulnerability disclosures through Kuaishou SRC and JD SRC
- **Summer 2024 (Junior Year)** — Threat Intelligence Intern — SecureWorks (Atlanta, GA) — vulnerability research and threat hunting
- **Bug bounty** — [Bugcrowd](https://bugcrowd.com/h/Bryce_Alan_Hayes): 8 validated vulnerabilities, 100% accuracy, Hall of Fame recognition

## Education

- **B.S., Information Science and Technology** (concentration: Systems & Network Security) — Georgia Gwinnett College, **2025**

## Affiliations

- TimelineSec (member, team 2) — bug bounty / CTF team

## Platform Handles

Some of my disclosures are credited under a platform-specific reporter handle rather than my real name. For verification purposes, here's the mapping:

- **VulDB / CVE reporter handle: `yunshen`** ("云深" on the Chinese-localized CVE record) — this is the reporter credited in the official [Acknowledgments](https://www.cve.org/CVERecord?id=CVE-2026-86666) section of CVE-2026-86665/86666/86667. Each VulDB entry backing those CVEs publicly states the submitter directly, no login required:
  - CVE-2026-86665 → [VDB-399756](https://vuldb.com/vuln/399756) — "Submitter: yunshen", "Submit #908923 ... (by yunshen)" ([archived snapshot](/img/vuldb-evidence/vdb-399756-entry.png))
  - CVE-2026-86666 → [VDB-399757](https://vuldb.com/vuln/399757) — "Submitter: yunshen", "Submit #908924 ... (by yunshen)" ([archived snapshot](/img/vuldb-evidence/vdb-399757-entry.png))
  - CVE-2026-86667 → [VDB-399758](https://vuldb.com/vuln/399758) — "Submitter: yunshen", "Submit #908925 ... (by yunshen)" ([archived snapshot](/img/vuldb-evidence/vdb-399758-entry.png))

  The account's [public VulDB profile](https://vuldb.com/user/100191) (no login required, [archived snapshot](/img/vuldb-evidence/yunshen-profile.png)) rolls all three up in one place under "Submits (3)".

  As a reverse check — intended to prove the `yunshen` account is actually controlled by me, not just that my site claims it — I posted a comment as `yunshen` on the VulDB discussion threads identifying myself as Bryce Alan Hayes / brycealanhayes.com. **All three are now published and public on VulDB** (no login required), so a reviewer can read the reporter's own self-identification directly on the primary source:
  - [VDB-399756 discussion](https://vuldb.com/vuln/399756#comments) (CVE-2026-86665) — comment published, moderation status Accepted
  - [VDB-399757 discussion](https://vuldb.com/vuln/399757#comments) (CVE-2026-86666) — comment published, moderation status Accepted
  - [VDB-399758 discussion](https://vuldb.com/vuln/399758#comments) (CVE-2026-86667) — comment published, moderation status Accepted

  Archived snapshots of the submissions are also kept locally ([VDB-399756](/img/vuldb-evidence/vdb-399756-comment.png), [VDB-399757](/img/vuldb-evidence/vdb-399757-comment.png)) but the live VulDB links above are the authoritative source.
- **Bugcrowd** — the [public profile](https://bugcrowd.com/h/Bryce_Alan_Hayes) is under my real name directly, with a reciprocal link back to brycealanhayes.com in its Website field.
