---
title: "About"
---

## Background

I'm a security researcher working across both offense and defense. I graduated from Georgia Gwinnett College in 2025 with a background in Information Science and Technology, concentrating in systems and network security. During undergrad I interned with SecureWorks' threat intelligence team at their Atlanta headquarters, doing vulnerability research and threat hunting — the year that built my manual bug-hunting fundamentals: working attack surfaces by hand, reading source, and testing authorization and business-logic paths branch by branch to find real vulnerabilities. That manual work is documented on my [Bugcrowd profile](https://bugcrowd.com/h/equa1) — 8 validated vulnerabilities at 100% accuracy, with Hall of Fame recognition on Pantheon and HostGator LATAM Bug Bounty.

The turning point came in **March 2026**, when I started porting that red-team methodology onto LLMs. I've taken remote AI Red Team engagements — through platforms like Mercor — running adversarial jailbreak and prompt-injection testing against frontier models (I built [promptprobe](https://github.com/HAYES-cyber/promptprobe) to automate the baseline sweep for that work), while in parallel folding AI tooling into my own vulnerability research: AI-assisted attack-surface mapping, code-audit assistance, payload generation, and attack-chain reasoning — a methodology I've written up in detail in [AI Coding Agent Workbench](/research/ai-agent-workbench/). The efficiency gain wasn't incremental; targets that used to take days to get a foothold on now take hours.

That experience raised an obvious follow-up question: if AI can attack systems this effectively, how secure is the AI itself? My research now runs on two tracks — using AI-assisted methodology to accelerate traditional vulnerability research (web application security, authorization bypasses, deserialization/RCE), and studying the security of AI systems themselves: prompt injection, adversarial robustness, RAG data exposure, AI agent authorization boundaries, and supply-chain risk hidden in AI-generated code.

My background also includes vulnerability disclosure through vendor Security Response Centers (Kuaishou SRC, JD SRC) and CVE-credited findings in open-source projects.

## Research Focus

- AI-assisted vulnerability research: attack-surface mapping, code-audit assistance, and attack-chain reasoning with AI coding agents
- AI/LLM security: prompt injection, adversarial robustness, RAG data exposure, AI agent authorization boundaries, and supply-chain risk in AI-generated code
- Web application security (authorization flaws, business logic vulnerabilities, SSRF)
- Deserialization and RCE chain research
- Vulnerability disclosure coordination (CNA/vendor SRC workflows)
- Security tooling for exploit verification and CTF operations

## Experience

- **March 2026 – Present** — AI Red Teamer (remote, via Mercor) — adversarial jailbreak and prompt-injection testing against frontier LLMs
- **Summer 2024 (Junior Year)** — Threat Intelligence Intern — SecureWorks (Atlanta, GA) — vulnerability research and threat hunting

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
- **Bugcrowd handle: `equa1`** — see the [public profile](https://bugcrowd.com/h/equa1) linked above.
