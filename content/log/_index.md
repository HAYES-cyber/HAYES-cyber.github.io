---
title: "Field Notes"
---

A running log of what I've actually been doing, in roughly chronological order. Not a highlight reel — the CVE and Bounty pages already cover the disclosures with full evidence. This is more the "why" and "how I got there."

## September 2026

Spent most of the month consolidating my public presence under this identity — migrated my tools over, cleaned up an old handle, and finally published [promptprobe](https://github.com/HAYES-cyber/promptprobe), a small CLI for running a categorized set of prompt-injection/jailbreak payloads against LLM-backed endpoints. I'd been doing this by hand for months on AI Red Team engagements; writing the tool was mostly about not re-typing the same dozen test prompts into every new target.

Also submitted three reports through VulDB's CNA pipeline for `aircheng-org/iWebShop-5` — missing authorization, unrestricted upload, and SQL injection, which came back as CVE-2026-86665/86666/86667. First time going through a CNA submission instead of reporting straight to a live vendor SRC program; the process is a lot more mechanical, and slower, than I expected.

On the writing side: finished the [Kubernetes attack playbook](/research/kubernetes-attack-playbook/), the [Windows client RCE hunting methodology](/research/windows-client-rce-playbook/), and the [AI Coding Agent Workbench](/research/ai-agent-workbench/) piece. The last one is the one I actually care about — it's the real writeup of how I use AI-assisted tooling for this work, not marketing copy for it.

## July – August 2026

Three reports through Kuaishou SRC on Kling AI (可灵AI): a Claw Login device-authorization flaw that let you pull a valid session without the user ever consenting, a Canvas bug that let you force yourself into someone else's project as a collaborator, and an internal `/dev/kconf-editor` tool that had no business being reachable by any logged-in user.

One report through JD SRC on JD Cloud LingJing AI (京东云灵境AI) — a billing-logic bug in `executeByApiId` combining a race condition with a missing quantity check and a balance-check bypass. That one took longer to pin down than the severity rating suggests; the race window was narrow enough that I had to script the concurrent requests instead of firing them by hand.

Also finally wrote up the [Spring DispatcherServlet routing-bypass piece](/research/spring-auth-bypass/) — a finding I'd been sitting on for a while and kept meaning to properly document.

## March 2026

The actual turning point. I started taking remote AI Red Team engagements through [Mercor](https://mercor.com) — adversarial jailbreak and prompt-injection testing against frontier models. By that point I'd been doing manual bug hunting for about a year and a half, and watching a model get through defenses faster than I could by hand was what got me paying attention to AI security as its own subject, not just a tool for accelerating the same old web app work.

## 2024 – 2025

Bug bounty work on [Bugcrowd](https://bugcrowd.com/h/Bryce_Alan_Hayes) picked up through this period, mostly against web application targets — Hall of Fame recognition on Pantheon and HostGator LATAM Bug Bounty among the programs I tested against.

**Summer 2024**, junior year: interned with SecureWorks' threat intelligence team out of their Atlanta headquarters. This is where the manual-testing fundamentals actually got built — working attack surfaces by hand, reading source, testing authorization paths branch by branch. No AI involved yet; that came later.

**2025**: graduated from Georgia Gwinnett College with a B.S. in Information Science and Technology, concentrating in systems and network security.
