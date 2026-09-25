---
title: "AI Coding Agent Workbench: Setup and Field Practice"
date: 2026-09-20
tags: ["ai-security", "agent", "web", "mobile-security", "methodology"]
summary: "Wrapping coding agents like Claude Code in four engineering layers — rules, memory, capabilities, and a ledger — turns them into a pipeline that keeps producing gradeable reports against authorized targets: a field methodology spanning cold start through Web / mini-program / APP target types."
toc: true
draft: false
---

> The methodology backbone comes from Devansh's *Needle in the Haystack*; tool details come from public documentation; the field data comes from our own authorized-testing ledger. Data as of 2026-09-20. Scope: written-authorized security testing only.

How to stand this whole thing up from a bare machine, what you'll run into along the way, how to approach Web, mini-programs, and APPs differently, and finally — how to actually talk to the model.

## Three Sentences for Management

**1.**
A one-time investment of about five person-days. Every new target after that cashes in on the dividend — cold start drops from days to hours.

**2.** AI
takes over the grunt work: reading through code, enumerating, replaying, drafting. What counts as a vulnerability, how bad it is, and when to stop — that's still a human sign-off, and the process requires it.

**3.**
Compliance lives in the config, not in good intentions. Authorization allowlists, write-operation bans, rate caps, data redaction, full logging throughout — more auditable than pure manual testing.

| **Prerequisite statement**　Everything in this post applies only to security testing that has written authorization, a clearly defined scope, and is fully auditable. None of these methods apply to unauthorized targets. Every tool and technique here comes from public sources — the value of this post is in assembling them into a reusable pipeline, not in providing an attack plan for any specific target. Any field data mentioned here comes from projects that were authorized and already reported to the vendor. |
|----|

## 00 · Summary: The One-Page Version


------------------------------------------------------------------------

Wrapping Claude Code
and agents like it in four engineering layers — rules, memory, capabilities, ledger — turns them into a pipeline that keeps producing gradeable reports against authorized targets. It's not a better prompt; it's a directory structure plus a set of conventions.

#### Four numbers

| **5 person-days** | **¥50 – 300** | **Days → hours** | **~50%** |
|----|----|----|----|
| Building from zero to a working first loop | API cost for one deep pass on a single business line | Cold-start period for a new target | Cost drop starting from the second round |

#### What it solves

Dump a repo in and say "find all the vulnerabilities," and what comes back looks professional but isn't actually usable. A stronger model doesn't help either — we ran the same target through
Opus 5 and GLM-5.3,
and the output quality barely differed; neither was usable. The problem is the method, not the model. Using it raw has four structural flaws, and the workbench's four layers are built to fix them one for one:

| **Problem with raw usage** | **Countermeasure** | **What it becomes** |
|----|----|----|
| No threat model, so the model has no concept of "impact" — it can only list CWEs generically | Rules layer: thin slices + trust boundaries | CLAUDE.md + one threat-model.md page per target |
| Broad prompts trigger breadth-first hallucination — coverage bought at the cost of zero depth | Method convention: break invariants into a two-step question | Task-brief template (goal / steps / criteria / budget) |
| Sessions are stateless — assumptions disproven last round get retested this round, burning money twice | Memory layer + ledger layer | memory/ three files + tested.md hypothesis matrix |
| No sense of boundaries — it'll poke at adjacent hosts or trigger a write operation without meaning to | Rules-layer hard limits + three-tier scope guardrails | Browser allowlist + proxy allow_hosts + hard outbound cutoff |

#### What it looks like once it's built

A directory skeleton you can copy, a one-page rules file, a set of tools fitted to the target's form factor, plus a ledger recording every hypothesis and its outcome. All three target types share the same skeleton — only the capability layer's tools change:

| **Form factor** | **What's in the capability layer** | **What's reused** |
|----|----|----|
| Web | DevTools MCP + proxy + frontend white-box | Rules layer, memory layer, ledger layer, report template — identical across all three form factors |
| Client (mini-program) | wxapkg / asar unpacking + UIA automation | Same as above |
| APP | jadx / Frida + unpacking + unbinding | Same as above |

#### A ruler that runs through the whole piece

Devansh's Needle in the Haystack gives a useful budget split: scaffolding no more than
10%, slice audits 60 to 80%, and the remaining 20 to 30% for verification.

| **Budget segment** | **Share** | **Spent on** |
|----|----|----|
| Scaffolding | < 10% | One-page threat model, invariant checklist, hard limits |
| Slice audit | 60 – 80% | Targeted audit of one trust boundary: white-box code reading, black-box verification, sub-agent slicing |
| Verification | 20 – 30% | Evidence, not conclusions: reproduction, adversarial self-review, human sign-off |

The ratio itself isn't sacred — its use is that when you notice scaffolding has gone over
10%, you should cut the rules file, not add more context window. This runs through every section of this piece — why the rules file has to stay to one page, why large files have to go through an index, why every conclusion needs human review — it can all be explained by this ratio.

| **How to convert the ratio into a word count**　Rough estimate: about 1 token per Chinese character, so one A4 page of Chinese text is roughly 800–1000 characters, i.e. around 1k tokens. With a 200k-token effective window, a 10% scaffolding cap is 20k tokens — about 20 pages worth. Sounds generous. But don't forget tool returns eat into that budget too: a single list_network_requests call can run several thousand tokens, and a mid-sized MCP server's tool definitions alone can cost a few thousand. What's actually left for the rules file is about one page. |
|----|

## Part One: Why Using AI Raw Doesn't Work


------------------------------------------------------------------------

Most people's first move with Claude Code
is to dump the whole repo in and say "find all the vulnerabilities." That doesn't work, for four reasons: the first two are model behavior, the last two are missing engineering. None of the four get fixed by switching to a stronger model.

### 1.1 Without a Threat Model, the Model Just Does Retrieval

If you don't say who the attacker is, where the trust boundaries are, or what the attacker can control, the model has no concept of "impact." It can guess at some generic
CWEs from its training data, but it guesses badly. The result is a long list of "theoretically possible XSS," "suggest checking SQL
concatenation" — no priority, and you can't pick out what's worth following up on from the noise.

The technical reason is worth spelling out, because almost every design choice later in this piece is aimed at it. Without constraints, an
LLM
does pattern recall, not reachability analysis. It retrieves "what vulnerabilities this kind of code usually has" from its training distribution and outputs a prior. What a real vulnerability determination actually needs to answer is a different question:

| Given this trust boundary, can the attacker reach this code; and once they've reached it, can they cross the boundary. |
|--------------------------------------------------------------------|

Written as a conditional probability, it's roughly P(exploitable | code, trust boundary, attacker capability,
reachable path). If you don't supply the last three conditioning terms, all the model has left is P(exploitable |
code) — an unconditioned prior stripped of context. That pile of "theoretical vulnerabilities" it outputs is essentially reciting the average risk of that kind of code from its training set.

Devansh's
piece has a line worth copying straight into your rules file: threat modeling is the ultimate compression algorithm for security auditing. A one-page threat model can replace dozens of pages of generic rules, because it fills in conditioning terms, not raw information. The difference on the
token
ledger is enormous: generic rules add noise to the context; conditioning terms cut the search space by orders of magnitude.

### 1.2 Broad Questions Dilute the Reasoning Budget

This one's sneakier, because the output looks abundant. A broad question draws a broad answer — the model matches whatever common patterns it's seen, even when there's no way they'd trigger in your context. You end up auditing paths no attacker could ever reach, spending tens of thousands of
tokens on a list that doesn't reproduce. The original piece calls this breadth-first hallucination.

There are two mechanisms at play here, worth separating out:

**▪ The reasoning budget gets diluted.**
A broad question spanning the whole repo splits the model's limited reasoning across every file, so each file only gets a surface-level pass. Attention is a finite resource — the bigger the scope you give it, the less falls on any given unit of code.

**▪ The output strategy optimizes for the wrong target.** Under a broad question, "listed 20
directions" looks more like it answered "all the vulnerabilities" than "went deep on 1
direction" does. So the model tends toward coverage — it's optimizing for what it thinks you want, not what you actually want.

There's another phenomenon called context rot
stacked on top of this: the longer the context, the less reliable the model gets, and it doesn't suddenly break at the window limit — it degrades gradually the whole way. So "stuff more material in" is itself the wrong move. A nominal
200k-token
window has a stable, effective reasoning portion far smaller than that number — plan around the effective window, not the nominal one.

### 1.3 Sessions Are Stateless — Duplicate Work Burns Money Twice

Every new session starts from zero. A hypothesis already tested and disproven last round gets rerun this round under different wording; patterns summarized from the last project are completely unavailable to this one.

The cost is doubled. Tokens
burn, and so does the target's request quota — and the latter is worth more, because it's bound by the target's rate-limiting/risk-control, a hard constraint. Run out of tokens and you can top up; get your IP banned and you have to wait.

This problem compounds with multiple people working together. Three people each run their own sessions, the same endpoint gets tested three times — wasting budget and genuinely raising the odds of tripping risk control. We paid for this on an
OTA
target: two people separately tested the same order-query broken access control, worded differently, same conclusion — the extra pass produced no new information but did land in the target's risk-control counter.

### 1.4 No Sense of Boundaries — This Is a Risk Problem, Not an Efficiency One

AI
has no sense of legality. Give it a domain, and it'll casually go poke at adjacent hosts; ask it to verify broken access control on an order-placement endpoint, and it might actually place an order; ask it to test an SMS endpoint, and it'll send the verification code to a real person.

These aren't model
bugs — the model was never told where the boundary is. It only knows a boundary if a human writes it into a file; if it isn't written down, the model decides for itself, and it usually decides wrong — its judgment is based on "does this help complete the task," not "does this cross a line."

The first three points are efficiency problems; this one is a risk problem. For an internal report, it's also the point most worth emphasizing: the workbench's rules layer doesn't just make
AI
run more accurately — it turns compliance from "relying on the tester's good judgment" into "written into config, logged the whole way through." Part Three gives the concrete configuration for the three-tier
scope guardrails.

### 1.5 Three Counter-Examples: How Other People Actually Found Them

The author of Needle in the Haystack reported 30-plus
vulnerabilities in two months using this method. Three worth picking apart:

#### Parse Server　CVE-2026-29182 / 30228 / 30229

Started by going through historical
CVEs and found they were all incomplete permission checks. Used that to have the model generate a threat model focused on "permission-boundary enforcement," then zeroed in on one specific boundary: the read-only admin
key. The model found multiple route handlers that checked isMaster but not
isReadOnly. That's a pattern, not a single point, so a narrower prompt had it enumerate every handler matching that pattern —
three CVEs, one root cause.

#### ElysiaJS　Cookie Signature Verification

The slice chosen was cookie signature verification. The bug found was a single initialization flipped backward — let decoded =
true should have been false, which meant invalid cookies weren't rejected during key rotation. Static scanners basically can't catch this kind of bug, because the syntax is completely valid — what's wrong is the semantics, and semantics happen to be the one place an
LLM
has a real edge over a scanner.

#### harden-runner　CVE-2026-25598

The threat model was one sentence: can an attacker bypass egress controls to exfiltrate data. Then did a syscall-coverage analysis and found the
UDP calls (sendto / sendmsg /
sendmmsg) weren't being monitored. Asking the same question of a different project,
BullFrog, produced a completely different slice, because it uses a network-layer firewall instead of syscall instrumentation: DNS
resolution, IP-to-domain binding, privilege-escalation paths.

Notice what these three have in common: no 20-page Agent.md, no giant Skill
library, no full-repo scan. Just four things — pick a good slice, work the threat model backward from history, have the model hunt for invariant violations, follow up on the signal.

Anthropic auditing Firefox with Claude followed the same order: started narrow with the JavaScript
engine, and only after that loop was working did it expand to roughly 6,000 C++ files, eventually filing 112
reports, most of which were fixed in Firefox 148.0, at an
API cost of about four thousand dollars. Narrow first, then broad; get the loop working before scaling up — that's the one structural thing all these cases share.

### 1.6 A Few Known Model Biases, While We're At It

These will turn into concrete prompting techniques in Part Seven — laid out here first, because a lot of what follows is designed around them:

**▪ Primacy/recency effect.**
Information placed at the two ends of the context window gets used better; anything buried in the middle is the easiest to overlook. So key constraints should either go right up front, or get restated in the task brief.

**▪ Default compliance.**
Without a specific prompt, the model tends to say the code is fine. "Overall looks OK, a few minor points to watch" is the path of least resistance.

**▪ Completeness front-loading.** Its most confident findings come first; the subtle bugs only surface if you keep asking. So "anything else?" needs to be asked several rounds running.

**▪ Abstract instructions don't stick, concrete boundaries do.** Telling it "pay attention to security" does nothing; telling it "the read-only key must never trigger a write operation" gives it something to actually check against.

**▪ Tends to rationalize code.**
Seeing a suspicious-looking piece of code, its default assumption is "the author must have had a reason for this," and then it starts making excuses on the author's behalf. This one gets specifically countered in technique 07 of section 7.3.

Of these five, the first two determine how the rules file is laid out and how long it should be; the last three determine how you phrase questions. They aren't a list of flaws — they're design inputs for this whole method.

## Part Two: Architecture


------------------------------------------------------------------------

The workbench is six layers stacked together, so the model can see the target, take action, remember its conclusions, and stay within bounds. Each layer can be swapped out independently, and that's the single most valuable property of the whole design — not because it's elegant, but because in 2026 pricing, availability, and compliance requirements in this space shift every quarter.

### 2.1 The Six-Layer Structure

| **Layer** | **Name** | **What's in it** | **Notes** |
|----|----|----|----|
| L6 | Ledger layer | journal/　tested.md　findings.md | What's been tested on this target, the outcome, where the evidence is. A ledger against duplicate work, and a handoff document |
| L5 | Capability layer | Browser driver · traffic capture · unpacking · decompilation · runtime hooking | The model's hands. The only layer that differs across the three target form factors |
| L4 | Memory layer | patterns.md　techniques.md　tools.md | Knowledge reused across targets. What a new target's cold start relies on |
| L3 | Rules layer | CLAUDE.md + per-target scope.md / threat-model.md | Authorization scope, hard limits, working conventions |
| L2 | Terminal layer | Claude Code / Codex / compatible-API clients | Manages sessions, tool calls, sub-agent dispatch, context |
| L1 | Model layer | Primary model + fallback model | Pure reasoning capability. Swappable, and should stay swappable |

L1 and L2 are bought, L3 and L4 are written, L5 is installed, L6
is generated by running the process. The one-time investment mostly goes into L3 and L5; the long-term value accumulates in L4 and
L6. For a report, this line works directly: **what your five person-days bought is L3 and
L5; what you're left holding is L4 and L6.**

### 2.2 Data Flow: The Hypothesis Is a First-Class Citizen

The core object of the whole pipeline isn't "code" or "vulnerability" — it's the hypothesis: a falsifiable assertion of the form "the read-only key must never trigger a write operation." Every hypothesis has a number, a status, and evidence. Organize around hypotheses and the pipeline closes the loop on its own.

| **Stage** | **What happens**                         | **Output**           |
|----------|------------------------------------|--------------------|
| Observation     | Traffic capture / unpacking / decompilation               | Raw material + index    |
| Asset-building   | Compress raw material into structured tables           | Route table, identity-field table |
| Invariants   | Find what the code claims always holds true           | invariants.md      |
| Hypothesis     | Assign each invariant a number, a criterion, a request budget | New row in tested.md      |
| Verification     | AI drives the tooling to try to disprove it, stops on a hit        | The evidence three-piece set         |
| Review     | Independent human reproduction                         | Hit gets downgraded or confirmed     |
| Ledger     | Records both hits and disproven hypotheses                     | Feeds back into the next round       |

The key part is that last feedback loop: the ledger feeds back into the next round's hypothesis stage. Without that feedback, the whole pipeline is open-loop, and every round just repeats the last one. A lot of teams ship halfway through building this, run it for a few weeks, and find costs haven't dropped — the root cause is almost always right here.

#### The Hypothesis State Machine

The ledger works because a hypothesis's status set is finite and decidable. We allow only these states — not one more:

| **State** | **Meaning**             | **Entry condition**                                 |
|----------|----------------------|----------------------------------------------|
| hit      | Confirmed hit                 | Has the evidence three-piece set, and passed human review                   |
| miss     | Tested and disproven           | Tested, with the reason for disproving it written down                       |
| blocked  | Tested but blocked or undecidable | WAF blocked it, a required precondition couldn't be obtained, or it's outside authorized scope |

There's no "pending." Anything pending stays in journal/, it doesn't go into
tested.md. This restriction looks rigid, but it's the precondition for the ledger being reliably usable by the model: the moment you open up the state enum, the model starts inventing unfilterable values like "partial hit" or "suspected," and the ledger degrades straight into prose.

#### What an Invariant Is

An invariant is something the code claims always holds. A few examples:

**▪** A read-only key must never trigger a write operation

**▪** A JWT's issuer must equal the configured value

**▪** This field must come from the server-side session, never from the request body

**▪** The identity the dispatcher forwards downstream must come from the context validated by the gateway

**▪**
Privileged interfaces exposed via preload can only be called by pages on the app's own domain (this is the case study in Part Five, where the root cause was this exact invariant not holding)

There's a detail you have to follow when hunting for invariants: split "list the hypotheses" and "determine whether a hypothesis is violated" into two separate questions. The model is quite good at "list all the implicit security assumptions in this module" on its own, and it also reasons deeply when asked separately whether "hypothesis
X
holds on this path." Cram both into the same prompt and the result is usually shallow — it tries to do both at once and settles early for a passing-grade answer. Part Seven turns this into a template.

### 2.3 Three Things Layering Buys You

| **Property** | **Which layer's isolation causes it** | **What it actually means** |
|----|----|----|
| Swapping models doesn't touch the pipeline | L1 decoupled from L3‒L6 | When a foreign model's price jumps, an account gets banned, or a client requires data to stay in-country, switching to a domestic model is just an environment-variable change — rules, memory, ledger, and tools all keep working as-is |
| Swapping targets doesn't lose knowledge | L4 separated from L6 | The memory layer stores "what kind of place tends to break"; the ledger layer stores "how far testing has gotten on this target." A new target inherits the former and starts the latter fresh |
| Swapping people doesn't lose context | L6 is structured | The ledger itself is a handoff document. When staff rotate or multiple people work in parallel, reading tested.md tells you exactly where to pick up |

The first point has been especially concrete in 2026
. We switched primary models twice this year — once because of a price hike, once because a client required data to stay in-country. Both switches were, in practice, changing two environment variables and running a health check; the rules file, memory layer, ledger, and MCP
config didn't change by a single character. If the method had been hard-wired into one vendor's Skill
system from the start, each of those switches would have cost a week.

### 2.4 How Much Scaffolding Is Just Right

This is the point most teams get backward. The first instinct for a lot of teams is to write an exhaustive rules document and preload a giant knowledge base — and the result performs worse than writing nothing at all.

| **Scaffolding that's enough**   | **Scaffolding that's too much**                            |
|--------------------|---------------------------------------------|
| A one-page threat model     | A twenty-page Agent.md stuffed with every policy and style guideline     |
| A list of three to five key functions | A huge Skill library, dozens of preloaded files           |
| A handful of clear invariants   | Telling AI to "find all the vulnerabilities" and expecting it to figure out the context itself |
| Pointing at one trust boundary   | Dumping the whole codebase in with no direction                |
| An on-demand file index | Feeding an entire large file into context whole                      |

Scaffolding's job is to anchor attention on the right place — one page is enough. A twenty-page rules file actually dilutes attention: the model has to spend
tokens understanding your rules, which leaves less reasoning room for analyzing the code.

This can be worked out directly. A 20-page rules file eats up roughly fifteen to twenty thousand tokens. Working backward from the 10%
cap, it's only barely compliant if your effective working window is above two hundred thousand tokens
— and the effective window is far smaller than the nominal one, which is exactly what context rot
means. And the rules file isn't the only standing overhead, either: MCP
tool definitions, the system prompt, directory listings all take up space, and together they often outweigh the rules file itself.

| **An actionable rule of thumb**　If your rules file goes over one page, that's the moment to ask whether you've mixed "hard limits that must be followed every single time" with "methods only needed for certain task types." The former belongs in CLAUDE.md as standing content; the latter belongs in the task brief, supplied only when needed. This split is what lets the standing portion stay reliably under one page — it's the core design behind Step 2 in the next part. |
|----|

## Part Three: Building It


------------------------------------------------------------------------

Seven steps, from wiring up the model all the way to a minimal closed-loop acceptance check. Each step comes with configuration you can copy directly and the pitfalls specific to it. Done in order, it takes about five person-days. The tools specific to each of the three target form factors are in Parts Four through Six — this part only builds the form-factor-agnostic skeleton.

| **Section**     | **What this step produces**                               |
|--------------|--------------------------------------------------|
| 3.0 Connecting the model   | Primary + fallback dual-model setup + one-key switch + health-check script                   |
| 3.1 Directory skeleton | A copyable workbench/ — new targets change config, not process        |
| 3.2 Rules layer   | A one-page CLAUDE.md, six hard limits locked in                   |
| 3.3 Memory layer   | The patterns / techniques / tools trio and recall prompts |
| 3.4 Capability layer   | .mcp.json, indexing strategy, three-tier scope guardrails             |
| 3.5 Ledger layer   | The tested.md hypothesis matrix                               |
| 3.6 Sub-agents | Task-brief template and slicing principles                             |
| 3.7 Acceptance     | Ten observable checks — missing even one means it's not done               |

### 3.0 Connecting the Model

The conclusion up front: run one primary and one fallback, don't pick just one. Compatible APIs make switching cost nearly nothing — not using that is wasteful.

#### Four Variables for the Decision

| **Variable** | **What to look at** |
|----|----|
| Capability | One-shot pass rate on writing code, editing large projects, running long tasks. Check third-party leaderboards, not just the vendor's own numbers |
| Price | Coding agents produce a lot of output, so look mainly at output pricing, not input pricing |
| Availability | Can you register, can you pay, do you need a proxy node, will the account get banned |
| Data compliance | Can client code and traffic leave the country. If not, foreign models are ruled out immediately |

#### The Lineup as of 2026-09

| **Vendor** | **Primary model** | **Notes** | **Access method** |
|----|----|----|----|
| Anthropic | Claude Fable 5.1 / Opus 5 / Sonnet 5 / Haiku 4.5 | Top tier for coding agents; Claude Code is the official tool | Pro / Max subscription, or API |
| OpenAI | GPT-6 Astra / GPT-5.6 Sol · Terra / GPT-5.3 Codex | Same tier; Codex is the official tool | Go / Plus / Pro subscription, or API |
| DeepSeek | V4 Pro / V4 Flash | Cheap, open weights; price hike with peak/off-peak pricing starting August | Pay-as-you-go API, no subscription |
| Zhipu | GLM-5.3 / GLM-5.3-Flash | Strongest coding performance among open models; has an Anthropic-compatible API | Coding Plan subscription, or API |
| Moonshot AI | Kimi K3 / K2.7-code | K3 is the largest domestic model (2.8T); has a compatible API | Kimi Code plan, or API |
| Alibaba | Qwen3.8-Max | Large ecosystem, Bailian platform; discounted at night | Token Plan, or API |

Artificial Analysis's agentic index (as of 2026-09-18) ranks them in this order: Claude
Fable 5.1 tied with GPT-6 Astra at 60, Claude Opus 5 at 59, GPT-5.6 Sol
at 55, GLM-5.3 at 54, Kimi K3 at 52, Qwen3.8-Max tied with DeepSeek V4 Pro at 43. A
10-point
gap doesn't mean unusable — it means a somewhat lower one-shot pass rate on complex refactors and long tasks; an extra round usually gets it done anyway.

API output pricing is the big line item. Foreign top-tier models run 25 to 50 USD per million tokens; domestic GLM-5.3
and DeepSeek run only around 4 USD — a 6-to-12x difference. On subscriptions: Claude Pro is 20 USD
/ Max 100 / Max 20x 200; GLM Coding Plan ¥118‒1078; Kimi Code
¥49‒699; DeepSeek is pure pay-as-you-go.

#### Four Hurdles With Foreign Models

| **Blocker** | **Situation** | **Workaround** |
|----|----|----|
| Region | Not offered to mainland China | Needs a proxy node; companies go through a cloud-hosted entry point like Bedrock / Vertex |
| Registration | Doesn't accept mainland phone numbers; Claude has required KYC since 2026-07 | An overseas phone number + real ID |
| Payment | Only accepts foreign credit cards | Company issues a shared card; individuals can go through an App Store subscription |
| Bans | Frequent IP switching or payment anomalies get accounts banned | A fixed node, a fixed device, never share accounts |

#### How to Configure the Primary/Fallback Pair

GLM and Kimi both offer Anthropic-compatible APIs, meaning the Claude Code
terminal can point straight at them — just two environment variables. This is the single best cost-to-value design in the whole workbench: the terminal layer stays fixed, the model layer switches anytime.

> # Primary: Anthropic official
>
> wb-main() {
>
> unset ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN
>
> export ANTHROPIC_API_KEY="\$(security find-generic-password -s
> wb-anthropic -w)"
>
> export ANTHROPIC_MODEL="claude-opus-5"
>
> }
>
> # Fallback: GLM-compatible API, same Claude Code terminal
>
> wb-backup() {
>
> unset ANTHROPIC_API_KEY
>
> export ANTHROPIC_BASE_URL="https://open.bigmodel.cn/api/anthropic"
>
> export ANTHROPIC_AUTH_TOKEN="\$(security find-generic-password -s
> wb-glm -w)"
>
> export ANTHROPIC_MODEL="glm-5.3"
>
> }
>
> # Health check: fire one off right after switching, don't find out it's down halfway through a task
>
> # Must include one real tool call — testing chat alone won't surface compatibility-layer issues
>
> wb-ping() {
>
> claude -p "Use the Read tool to read the first line of ./CLAUDE.md, reply with only that line verbatim"
> --max-turns 3
>
> }

Keys go through the system keychain, never written in plaintext into an rc
file. Workspace directories get packaged up and carried around all the time — a plaintext key riding along with it is a very real risk.

#### Known Differences in the Compatibility Layer

Worth spelling out the root cause here, because knowing it saves a lot of time when you hit these. Anthropic's
Messages API expresses tool calls as tool_use / tool_result
content blocks in the message; the OpenAI family uses the function_call approach — the parameter containers, ID
correlation, and how parallel calls are expressed all differ between the two. What a compatible API does is field mapping — simple calls map cleanly, complex ones aren't guaranteed to.

| **Difference** | **Symptom** | **Countermeasure** |
|----|----|----|
| Tool-call format | A small number of calls with complex nested parameters get rewritten or lose fields | The health check must include one real tool call, ideally one with nested parameters |
| Long-context truncation | May silently truncate near the window limit instead of erroring | Large files always go through the index for excerpts, see 3.4 |
| Streaming interruption | A long task disconnects partway through, and the tokens up to that point are already spent | Keep tasks small; write conclusions to the ledger immediately, don't save them up for the end |
| Model alias drift | The actual version an alias points to gets quietly swapped | Pin a specific version number; log the actual model identifier returned at each weekly health check |

#### Three Recommended Configurations

| **Setup** | **Primary** | **Fallback** | **Per month** |
|----|----|----|----|
| A · Has overseas payment | Claude Pro (\$20) or Max 5x (\$100) | GLM Coding Plan Lite (¥118) | ¥260 – ¥830 |
| B · No overseas payment | GLM Coding Plan Pro (¥538) | DeepSeek V4 API | ~¥590 |
| C · Testing the waters | GLM Coding Plan Lite (¥118) | DeepSeek V4 API | ~¥170 |

RMB converted at 1 USD = 7.1 RMB. Prices change monthly — check the official pricing page before committing.

| **Answer this one question before choosing a setup**　Is this target's code, traffic, and logs allowed to leave the country. If not, Setup A is immediately out, and there's no middle ground of "just send out the irrelevant parts" — production data bleeding into traffic-capture output is the norm, and manual filtering isn't reliable. In that case, go with Setup B, or further, use a locally-deployed open-weight model. |
|----|

### 3.1 The Directory Skeleton

Copy this directly. New targets change config, not process — that's the key step in turning one-off experience into a reusable asset.

> workbench/
>
> ├── CLAUDE.md # L3 standing content · one page · hard limits locked in here
>
> ├── .mcp.json # L5 MCP server declarations
>
> ├── .env.example # the real .env never goes into version control
>
> │
>
> ├── memory/ # L4 reused across targets · append-only
>
> │ ├── patterns.md # vulnerability patterns: what kind of place tends to break
>
> │ ├── techniques.md # techniques: how to test, how to get around things
>
> │ └── tools.md # tools and commands: environment setup, version mappings
>
> │
>
> ├── targets/<slug>/
>
> │ ├── scope.md # authorization summary: allowlist, time window, contact
>
> │ ├── threat-model.md # one-page threat model
>
> │ ├── assets/ # structured results from observation
>
> │ │ ├── routes.md # endpoint route table
>
> │ │ ├── identity.md # identity fields: server-issued vs. client-self-reported
>
> │ │ └── invariants.md # invariant checklist, the source of hypotheses
>
> │ ├── capture/
>
> │ │ ├── flows/ # raw traffic
>
> │ │ └── index.jsonl # structured index — this is what AI searches
>
> │ ├── decompiled/
>
> │ │ ├── INDEX.tsv # file listing + size + first line
>
> │ │ └── INDEX-keywords.md # keyword heat map, decides which files to read first
>
> │ ├── journal/<date>.md # L6 daily log
>
> │ ├── tested.md # L6 hypothesis matrix — the core anti-duplication mechanism
>
> │ ├── findings.md # L6 hits + evidence three-piece set
>
> │ └── report/
>
> │
>
> └── tools/
>
> ├── mitm-addons/ # writes structured jsonl to disk
>
> ├── unpack/ # mini-program / asar unpacking
>
> └── frida/ # hook script library

| **Item** | **Rule** |
|----|----|
| Goes into version control | CLAUDE.md, .mcp.json, memory/, tools/, and each target's scope.md / threat-model.md / tested.md / findings.md |
| Never goes into version control | .env, capture/flows/, decompiled/, and any artifact containing real credentials or third-party PII. Write .gitignore on day one |
| How to start a new target | Copy targets/_template/, fill in scope.md, clear the ledger, inherit the memory layer as-is |

Those two INDEX files deserve a separate note. capture/index.jsonl and
decompiled/INDEX-keywords.md
aren't raw data — they're a retrieval entry point prepared for the model. Having it read mitmproxy's
binary stream directly, or thousands of decompiled Java
files, is the number-one cause of context blowup. Build the index first, then pull excerpts on demand — that alone can cut a single task's
token spend by an order of magnitude. See 3.4 for how.

### 3.2 Rules Layer: How to Write CLAUDE.md

Only hard limits go in the standing content; methods go in the task brief. That's what keeps the standing portion reliably under one page.

#### Six Things That Must Be Locked In

**1. Target allowlist**　An explicit list of
domains / AppIDs /
package names — anything off the list is forbidden. Skip this and the model will casually probe adjacent hosts and related domains.

**2.
Read-only by default**　Placing orders, changing passwords, deleting, paying, sending messages, nudging orders along — every write operation is forbidden by default and needs an explicit exemption.

**3. Rate and batch caps**　≥ 1.2 seconds between requests, ≤
500 per batch, logged the whole way through. The model doesn't know that hammering too fast gets you banned.

**4. Never trigger verification-code endpoints**　SMS / email /
voice verification codes are always forbidden. This harasses a real person — at that point it's not a technical question anymore.

**5. Verification reads of third-party data ≤ 2
records**　Proving broken access control only needs one record. Stop on the first hit, redact in the report.

**6. Credentials and PII
never land in the body text**　Not in the report body, not pasted into the conversation, not into version control, not sent to a third-party model.

#### How You Phrase It Directly Determines the Behavior

"Be careful not to accidentally trigger a write operation" is a suggestion — the model will weigh it and cross the line whenever it judges that necessary. "All write operations are forbidden by default unless this task brief explicitly lists an exempt endpoint" is a rule — the model treats it as a hard constraint and comes to ask you before attempting a write. Same underlying intent, but the violation rate for the former is noticeably higher than the latter. Wording in the rules layer isn't a matter of style.

> # Authorization scope
>
> This workspace is for security testing with written authorization only. Authorization letter: targets/<slug>/scope.md
>
> The allowlist in scope.md
> is authoritative; any host, domain, or AppID not on the allowlist may not receive any request.
>
> # Hard limits (forbidden by default, needs explicit exemption in the task brief)
>
> 1\. Write operations: placing orders / payment / password change / deletion / sending messages / nudging orders / form submission —
> all forbidden
>
> 2\. Verification-code endpoints (SMS / email / voice) — never trigger these
>
> 3\. Rate: >= 1.2s between requests; <= 500 per batch; stop and ask if exceeding this
>
> 4\. Third-party data: verification reads <= 2 records, stop on first hit, redact in the report
>
> 5\. Credentials and PII: never written into findings.md body text, never pasted into the conversation, never leaves the workspace
>
> 6\. Destructive actions: rm / drop / bulk file writes — forbidden
>
> # Working conventions
>
> - Required reading before starting: targets/<slug>/tested.md, memory/patterns.md
>
> - Write conclusions immediately: append to tested.md right after verifying each hypothesis, don't save it up for the end
>
> - Never read large files whole: read the INDEX first, then pull excerpts on demand (<= 400 lines at a time)
>
> - Stop on a hit: once a hypothesis is confirmed, stop expanding and hand off to human review
>
> - When unsure whether something crosses a line: stop and ask, don't decide on your own
>
> # Output format
>
> - Write hypothesis conclusions as: ID | Surface | Hypothesis | Status | Evidence path | Date
>
> - Every hit must include the evidence three-piece set: raw request / raw response / reproduction steps
>
> - Status can only be one of hit / miss / blocked

| **Something worth knowing: rules stop working**　The rules file doesn't stay in effect forever. Late in a long session, once the context has piled up dozens of rounds of tool results, that opening rules block's pull on current attention noticeably fades. What this looks like: three hours in, the model starts doing things it absolutely wouldn't have done in hour one, like probing a subdomain that isn't on the allowlist. This is a direct consequence of the primacy/recency effect from 1.6 — the model hasn't "gone bad." What actually holds the line is the three-tier tool-level guardrails in 3.4. |
|----|

### 3.3 Memory Layer: How Knowledge Gets Captured and Recalled

This is where the long-term value comes from. On your first target you're just using the tools; starting from the third target, you're using the experience from the first two.

| **File** | **What it stores** | **Criterion** |
|----|----|----|
| patterns.md | Vulnerability patterns: what kind of place tends to break | "Would this conclusion still hold on a different target?" Write it only if yes |
| techniques.md | Techniques: how to test, how to get around things | An actionable sequence of steps. Example: mini-program automation needs accessibility mode activated first |
| tools.md | Tools, commands, and version mappings | Commands you can paste and run directly + known gotchas. Example: which unpacking script a given WeChat version needs |

#### Entry Format: With Hit History and Counter-Examples

The format isn't ceremony. Hit history tells you how trustworthy a pattern is — three hits and one hit carry different weight. Counter-examples guard against memory-layer contamination — an over-generalized bad pattern will mislead every subsequent project, and because it's written into memory and read every single time, the error keeps getting reinforced.

> ## P-012 Dispatcher Trusts a Self-Reported Identity in the Request Body
>
> Shape A gateway / dispatcher treats userId / userName in the request body as a trusted identity,
>
> and passes it straight through downstream without re-verification.
>
> Trigger conditions Microservice architecture + gateway-centralized auth + downstream designed on the assumption that "internal network = trusted."
>
> Doesn't hold for monoliths or downstream services that verify independently.
>
> Detection steps 1) Find requests in traffic that carry both a token and a self-reported identity field
>
> 2\) Keep the token unchanged, modify only the self-reported field
>
> 3\) Call a query endpoint that returns the user's private data
>
> 4\) Check whether the response contains someone else's data
>
> Hit history OTA-A (2026-08, high severity) OTA-B (2026-09, high severity)
>
> Counter-example Fintech target F-1: the gateway binds userId
> into the signature with HMAC — modifying the field invalidates the signature.
>
> Whenever a request carries a signature field, reverse-engineer the signing algorithm first — don't just try modifying it directly.
>
> Priority High — if a new target hits the trigger conditions, recommend putting this in the first three test surfaces

| **Question** | **Practice** |
|----|----|
| When to write it | Write it all at once at project wrap-up, not casually during the work. Judgments made mid-process haven't been reviewed yet, and it's easy to write a false positive into long-term memory |
| What to write | Only what's transferable. Not "this target's /api/v2/order has broken access control" — instead "this kind of dispatcher architecture tends to fail at X" |
| How to recall it | The first thing at the start of a new target: have the model read patterns.md + threat-model.md and output the 5 most likely-to-hit patterns, ranked |
| How to prevent contamination | Review quarterly. Entries with zero hit history and unverified across more than three projects get downgraded or deleted |

> Read memory/patterns.md and targets/<slug>/threat-model.md.
>
> Based on this target's architectural characteristics, pick the 5 patterns from patterns.md
>
> most likely to hit, ranked by how well the trigger conditions match. For each one, give:
>
> - Pattern number and name
>
> - Why you think it matches this target (pointing to specific evidence in threat-model)
>
> - What the first verification action on this target would be
>
> - Whether the counter-example condition holds on this target (if it does, downgrade or exclude)
>
> Don't output generic security advice — output only the ranking and rationale for these 5.

This step is the direct reason cold start drops from days to hours. Take P-012 as an example: once it was confirmed on one OTA
target, the next similar target put it straight into the top three test surfaces and reproduced the same class of issue on day one. What that saves is the one-or-two days that would otherwise go into feeling out the architecture and guessing at weak points.

The Electron case study in Part Five also went into the memory layer, as an entry roughly along the lines of "domain-wide preload injection +
external links opened in-app = UXSS escalatable to RCE," with trigger conditions written as "Electron client +
in-app browser +
has a social-sharing external-link entry point." This entry got checked first on both of the next two desktop targets — one hit, one miss — and the miss was just as valuable, since it let us narrow the trigger conditions.

### 3.4 Capability Layer: MCP, Indexing Strategy, Three-Tier Guardrails

This layer decides whether the model has hands. MCP
wraps tools into atomic capabilities the model can call directly. Without it, the model can only write you a script, you run it, and paste the result back — a human is the bottleneck, and one loop takes minutes. With it, the model calls the tool itself, reads the result itself, decides the next step itself — one loop takes seconds. On tasks like "enumerate
200 handlers and verify each one," that's a qualitative shift.

#### What MCP Actually Is, and Its Hidden Cost

Stripped down, it's just an agreed-upon JSON-RPC 2.0
channel: the terminal (client) launches the server process you declared, communicates over stdio or HTTP, does an
initialize handshake, then pulls the tool list and each tool's JSON Schema
via tools/list — after that, every time the model uses a tool it's one
tools/call. The server can be twenty lines of
Python, or an entire Burp extension.

What matters is where that tool list sits — **it's part of the standing context**. Every tool's name, description, and parameter
schema add up — a few dozen tokens at minimum, two or three hundred for ones with complex parameters. Install a server exposing 30-plus
tools and the tool definitions alone can eat four to eight thousand
tokens, sitting there occupying space in every single turn. That gives "which MCPs are worth installing" a quantifiable criterion:

| Whether an MCP server is worth keeping resident comes down to its tool-definition token count divided by how many times it's actually called in this task. A server with thirty tools where only two get used is better replaced with a thin wrapper, or just called via Bash on the command line. The jadx-mcp-server in Part Six exposes thirty-plus tools; what we actually do is only mount it when working an APP target — it's absent from the Web-target configuration. |
|----|

The minimal form-factor-agnostic configuration has just two servers:

> {
>
> "mcpServers": {
>
> "capture-query": {
>
> "command": "python3",
>
> "args": ["tools/mitm-addons/query_server.py"],
>
> "env": { "INDEX": "targets/\${TARGET}/capture/index.jsonl" }
>
> },
>
> "code-search": {
>
> "command": "python3",
>
> "args": ["tools/search/rg_server.py"],
>
> "env": { "ROOT": "targets/\${TARGET}/decompiled" }
>
> }
>
> }
>
> }

One queries the traffic-capture index, the other queries code — both return only excerpts, never the whole thing. Form-factor-specific
MCPs (browser, jadx, Frida, etc.) are given separately in Parts Four through Six.

#### Three-Tier Scope Guardrails

Section 1.4
already said boundaries can't be held by a prompt alone. Whatever can be blocked at the tool level shouldn't be left only in the rules file. Authorization scope should land in three places at once, so that any one layer failing alone still leaves a backstop:

| **Layer** | **Where it lives** | **What it blocks** |
|----|----|----|
| Layer 0 · Prompt | CLAUDE.md allowlist + restated in the task brief | Decays over a long session — a reminder, not a defense line |
| Layer 1 · Client | Browser --allowedUrlPattern / --blockedUrlPattern | Navigation and sub-resources outside scope simply can't be sent |
| Layer 2 · Proxy rules | mitmproxy allow_hosts regex + addon write-to-disk allowlist | Out-of-scope traffic never touches disk, cutting off data contamination at the source |
| Layer 3 · Hard outbound cutoff | Set data.server.error in the server_connect hook | The connection is killed outright, independent of any upper-layer config |

> # Layer 2: only process allowlisted hosts, let everything else through without writing to disk
>
> mitmdump --listen-port 8080 \\
>
> --set allow_hosts='^(.*\\)?target\\.example\\.com\$' \\
>
> --set anticomp=true \\
>
> --set hardump=session.har \\
>
> -s ./tools/mitm-addons/index_dump.py
>
> # Layer 3: hard outbound cutoff, written into an addon
>
> from mitmproxy import ctx
>
> SCOPE = {"api.example.com", "m.example.com"}
>
> def server_connect(data):
>
> if data.server.address[0] not in SCOPE:
>
> data.server.error = "out of scope" # connection killed outright
>
> ctx.log.warn(f"BLOCKED {data.server.address[0]}")

Layer 3
is the backstop — it catches "the config was written wrong" and "the model got around the upper layers." In practice it doesn't fire often, but every time it has, it's been a real incident — once we ourselves missed an escape character in the
allow_hosts regex, the . matched an unrelated domain, and Layer 3 caught it.

#### Traffic-Capture Output Has to Be Structured

Installing the CA cert and configuring the proxy are the basics. What actually affects efficiency is the on-disk format. Having the model read
mitmproxy's flow
files directly is a disaster: binary, huge, and a single read fills up the context. The right approach is to use an
addon to compress each flow into one line of JSON as it's captured, so the model queries it like a search index.

> # Key points from tools/mitm-addons/index_dump.py
>
> import json, hashlib, pathlib
>
> from mitmproxy import http
>
> SCOPE = {"api.example.com", "m.example.com"}
>
> OUT = pathlib.Path("capture/index.jsonl").open("a")
>
> REDACT = ("authorization", "cookie", "x-token", "set-cookie")
>
> def response(flow: http.HTTPFlow):
>
> if flow.request.pretty_host not in SCOPE:
>
> return # out of scope, don't write to disk
>
> body = flow.request.get_text(strict=False) or ""
>
> rec = {
>
> "ts": flow.request.timestamp_start,
>
> "method": flow.request.method,
>
> "host": flow.request.pretty_host,
>
> "path": flow.request.path.split("?")[0],
>
> # keep only param keys, not values, so PII never enters the index
>
> "q_keys": sorted(flow.request.query.keys()),
>
> "b_keys": sorted(json.loads(body).keys())
>
> if body.startswith("{") else [],
>
> # keep only a fingerprint of credentials, for identity comparison, never the raw value
>
> "auth_fp": {h: hashlib.sha256(
>
> flow.request.headers[h].encode()).hexdigest()[:12]
>
> for h in REDACT if h in flow.request.headers},
>
> "status": flow.response.status_code,
>
> "len": len(flow.response.content or b""),
>
> # raw flow stored separately; the index only gives the path, fetched only when needed
>
> "raw": f"flows/{flow.id}.raw",
>
> }
>
> OUT.write(json.dumps(rec, ensure_ascii=False) + "\n"); OUT.flush()

Three design points, each corresponding to a class of real incident:

**▪ The allowlist takes effect at write time.**
Out-of-scope traffic never touches disk at all — this shuts down the compliance problem of "unrelated target data leaking into test data" at the source.

**▪ Parameters keep only the key, never the value.**
What the model needs is "which fields does this endpoint accept," not the field values. Values often carry phone numbers, ID numbers, order numbers — not writing them to disk means they can't leak.

**▪ Credentials keep a fingerprint, never the raw value.**
Determining "were these two requests sent by the same identity" only needs matching fingerprints — you never need to see the
token itself.

#### Indexing Strategy

This is the single most important practice in the capability layer, and the easiest one to skip. Decompiling a mid-sized APP
produces thousands of Java files, hundreds of MB; unpacking a mini-program produces hundreds of JS
files. Feed the whole thing in and the context blows up instantly, and the model drowns in irrelevant code besides.

Do the math and it's obvious why. One traffic-capture index record compressed to JSON runs about 150 to 250
tokens; a thousand requests is on the order of two hundred thousand
tokens — reading the index in full just once fills the window, never mind the raw flows. Decompiled output is even more extreme: the Java source from a mid-sized
APP typically runs into the tens of millions of tokens
— three orders of magnitude bigger than any model's context window. So "just feed it in" was never an option here — the only option is retrieval.

**▪ Build the index first, let the model see the directory.**
Generate file paths, sizes, package names, a summary of the first several lines, and keyword hit counts. This index is usually only a few hundred lines, a few thousand
tokens.

**▪ Let the model decide what to read, pulled by excerpt.** After reading the index it says "I want to see lines 40 to 120 of
com/x/net/SignUtil.java," and the retrieval tool returns just that slice. This alone can drop a single task's token spend by an order of magnitude.

> # File listing + size + first line
>
> fd -e java . decompiled/ -x sh -c \\
>
> 'printf "%s\t%s\t%s\n" "\$1" "\$(wc -c <"\$1")" "\$(sed -n 2p
> "\$1")"' _ {} \\
>
> > decompiled/INDEX.tsv
>
> # Keyword heat map: decides which files to look at first
>
> for kw in http encrypt sign token secret exported WebView \\
>
> addJavascriptInterface setAllowUniversalAccessFromFileURLs; do
>
> echo "## \$kw"; rg -l --no-heading "\$kw" decompiled/ | head -30
>
> done > decompiled/INDEX-keywords.md

Don't underestimate this keyword heat-map step — in the field, an anomaly in INDEX-keywords.md like "this keyword's hit count is noticeably higher than the number of routes declared in the manifest" has more than once led straight to a hidden route.

### 3.5 Ledger Layer: The Hypothesis Matrix

The step most likely to get dismissed as paperwork and skipped, when it's actually the single biggest money-saver. On one target we accumulated
88 test nodes, of which about 60% were disproven.

| **File** | **Granularity** | **Role** |
|----|----|----|
| journal/<date>.md | Daily log, one line per action | A human-readable process record. For tracing back what happened on a given day, and also audit material |
| tested.md | By hypothesis, one line per hypothesis | For the model to read. Required reading at the start of every new session — directly determines what this round skips retesting |
| findings.md | By hit | Severity, reproduction steps, evidence three-piece set. The direct source for the report |

The format has to be a fixed-column table, not free prose. The model needs to be able to reliably parse it, append to it, and filter it by status.

> | ID | Surface | Hypothesis | Status | Evidence | Date |
>
> |-------|-----------|---------------------------------|----------|------------------|-------|
>
> | H-001 | Order query | orderId is enumerable, no ownership check | hit | ev/H-001/
> | 09-12 |
>
> | H-002 | Order query | Modifying the self-reported userId field enables broken access control | miss |
> gateway signature binding | 09-12 |
>
> | H-003 | Dispatcher | Downstream doesn't re-verify the gateway's identity | hit | ev/H-003/ | 09-13
> |
>
> | H-004 | File upload | Extension check bypassable (case / double extension) | miss |
> server-side allowlist | 09-13 |
>
> | H-005 | File upload | Upload path traversal | blocked | blocked by WAF, not penetrated
> | 09-13 |
>
> | H-006 | User center | Avatar URL SSRF | miss | only accepts allowlisted domains |
> 09-14 |

| **Rule** | **Explanation** |
|----|----|
| Status must be one of exactly three | hit confirmed · miss tested and disproven · blocked tested but blocked or undecidable. "Pending" is not allowed — pending items stay in journal |
| A miss must state a reason | "gateway signature binding" is ten times more useful than "no issue found." It tells the next round this path is blocked by a signature — reverse the signing algorithm first |
| How to force it to be read | Three places: CLAUDE.md says "required reading before starting" to cover the main controller; the task brief embeds the disproven-hypothesis list to cover sub-agents; at the start of a new round, have the model state up front "what this round will not test" |

| **A disproven conclusion isn't garbage**　Which surface was tested and why it didn't pan out — these records carry double value: they're both an anti-duplication ledger and proof of coverage for the report's "everything that should have been done, was." When a client asks "did you test for SSRF," you can point straight at H-006 and say yes, here's the conclusion, here's the date. That 60% of disproven results is what saves every subsequent round from repeated effort — it's the main source of the roughly 50% cost drop starting from round two. |
|----|

### 3.6 Parallel Sub-Agents

Running tests single-threaded wastes 80% of an agent's value. But the ceiling on parallelism's payoff isn't set by local compute — it's set by the target's rate limiting.

The main controller handles judgment and integration: reading the ledger, choosing surfaces, splitting up tasks, reviewing conclusions, deciding whether to stop or continue. Sub-
agents handle execution: enumerating, verifying, and reporting within a narrow scope. Each sub-agent
has its own context and its own request budget, and gets discarded once it's used up.

Context isolation is the real value of parallelism — far more important than "runs faster at the same time." If the main controller session enumerates
200
handlers itself, the tool results will blow out its context, and eventually it won't even remember what it was looking for anymore — that's the classic cause of "AI
starts forgetting." Swap in sub-agents instead, and those tens of thousands of tokens of exploration junk stay in the sub-agent's
context; all that comes back to the main controller is a conclusion table a few hundred tokens
long. The main controller's context stays clear the whole time as a result, and it can run for a full day without losing the thread.

#### Four Elements of a Task Brief

| **Element** | **Requirement** |
|----|----|
| ① Precise target | Down to the endpoint or file level. Not "check the order module," but "check /api/v3/order/detail and its 6 sibling endpoints" |
| ② Steps | A sequence of actions, including which tool to use and in what order. Don't let the sub-agent improvise its own method |
| ③ Hit criteria | What counts as a hit, written as a decidable condition. And lock in stop-on-hit — don't let it keep expanding after confirming one |
| ④ Request cap | A hard budget. Once used up, it must stop and report back — no self-granted extensions |

> ## Goal
>
> The 6
> endpoints marked [order-family] in targets/<slug>/assets/routes.md.
>
> Test only these 6 — do not expand to any other endpoint.
>
> ## Hard limits (apply to this task, must not be violated)
>
> - Read-only. Any write operation is forbidden, including but not limited to placing, canceling, or modifying orders
>
> - Request interval >= 1.2s, this task's request cap is 120, stop once used up
>
> - Stop immediately on a third-party-data hit, read at most 2 records
>
> ## Already disproven, do not retest
>
> - H-002 modifying the self-reported userId field (gateway signature binding, modification invalidates it)
>
> - H-004 upload-extension bypass (server-side allowlist)
>
> ## Steps
>
> 1\. Use capture-query to pull the real request templates for these 6 endpoints
>
> 2\. For each endpoint, list every client-controllable parameter (cross-reference assets/identity.md)
>
> 3\. For each controllable parameter, construct one request that "changes it to a value belonging to someone else"
>
> 4\. Compare responses: status code, length, whether it contains another user's identifying fields
>
> ## Hit criteria
>
> A hit is any response containing a business identifier (order number / phone number /
> name) that doesn't belong to the current test account.
>
> Stop this task immediately on a hit and report the following — do not continue testing the remaining endpoints.
>
> ## Report format
>
> | Endpoint | Parameter | Conclusion hit/miss/blocked | Evidence file | Requests used |
>
> Plus a judgment summary of 100 words or fewer. No security advice, no remediation suggestions.

#### Slicing Principles

**▪**
Slice by business line (one for trains, one for hotels) or by attack surface (one for the dispatcher, one for storage). Don't mix the two approaches — mixing them causes the same endpoint to end up covered by two slices.

**▪**
Slices must not have overlapping endpoints. Concurrent testing of the same endpoint causes the responses to trample each other — both sides get contaminated data and draw wrong conclusions, and it's also more likely to trip rate limiting. Before dispatching, intersect the endpoint lists across slices — if the intersection isn't empty, refuse to dispatch. This step can be scripted; it's a ten-line-of-code thing.

**▪** Concurrency is limited by the target side — 3 to 5
concurrent sub-agents in practice. The test is whether the global request rate is still within the hard limit: 5 sub-agents each holding a 1.2
-second interval add up to 4
requests per second combined, which is quite likely to already exceed what the target can tolerate. **The rate limit is calculated globally, not per
agent** — we only remember this rule because we got an IP banned learning it.

#### Two Failure Modes You Must Guard Against

| **A sub-agent's conclusion can't be trusted at face value.** It will hallucinate hits that don't reproduce. The main controller must independently review every hit with its evidence three-piece set, and downgrade anything that doesn't hold up to miss, with the reason recorded. A sub-agent's report is a lead, not a conclusion — no exceptions to this. |
|----|

| **A sub-agent can't see the main controller's rules file.** It has its own independent context — CLAUDE.md doesn't automatically carry over. Every hard limit has to be copied into the task brief — that's exactly why the template puts hard limits in the second section. This is the single easiest place in the whole pipeline for a compliance incident to happen. |
|----|

### 3.7 Acceptance Checklist

Don't take this straight to a real target once it's built. Verify item by item first — each one is an observable behavior, not just "is it installed."

**1.**
Get both the primary and fallback model to complete one real tool call each. Not just "the chat works" — make it actually call
Read or an MCP tool and get a correct result back. Almost every compatibility-layer pitfall shows up in tool calls.

**2.**
In a new session, have the model recite the hard limits. Ask "what operations are forbidden in this workspace," and it should list all six accurately. An incomplete list means the rules file either isn't being read, or it's too long and got diluted.

**3.**
The proxy can capture the target's traffic in cleartext, and the allowlist works. Visit a non-target site and check that it wasn't written to disk.

**4.**
Verify all three guardrail tiers individually. Have the model try to visit an out-of-scope domain and confirm it's blocked at the client layer; disable Layer
1 and try again, confirming Layers 2 and 3 catch it.

**5.**
The model can retrieve a specified endpoint from the index. Give it an endpoint name and have it return the request template and parameter list. This verifies index usability, not traffic-capture usability.

**6.** The model can read memory and output a pattern prediction. Use the cold-start prompt from 3.3
and see whether it can produce 5
ranked patterns with rationale. A vague output means the memory-layer entries aren't specific enough.

**7.** One sub-agent
dispatch can retrieve a structured conclusion. Dispatch a minimal task and check whether the report matches the table format the task brief required, and whether it respected the request cap.

**8. The ledger actually influences behavior.** Plant a fake miss
record in tested.md and have the model plan the next round — it should proactively say "H-00X already disproven, skipping this round."

**9.** Large files don't get read whole. Give it a 5,000
-line file and see whether it reads the index first or just reads the whole thing.

Item 8
is the most important. Every other item verifies that a part is installed; only this one verifies the loop is actually closed. If the ledger gets written but never read, the mechanism is open-loop, and none of the earlier investment pays off.

## Part Four: Web


------------------------------------------------------------------------

The easiest of the three form factors to work with. The browser is itself the best debugger, the tooling is the most mature, and the model has the most entry points. The core idea is to let it drive the browser itself: click, fill in fields, read results, and judge for itself — a human only reviews the evidence at the end.

| **Section** | **What it covers**                               |
|----------|------------------------------------------|
| 4.1      | What Chrome DevTools MCP can and can't do |
| 4.2      | Who handles the modify-the-packet half                         |
| 4.3      | Frontend JS white-box: turning minified code into two tables       |
| 4.4      | One complete broken-access-control verification chain                   |
| 4.5      | Deterministic verification: don't let the model judge whether there's a vulnerability on its own   |
| 4.6      | Trade-offs and hard limits specific to Web                  |

### 4.1 Chrome DevTools MCP

The official MCP server from Google, driving Chrome under the hood with Puppeteer, doing analysis with the DevTools
frontend, communicating over
CDP. Two of its design principles matter a lot for security testing: it returns semantic summaries rather than raw data streams, and heavy assets (screenshots, traces) come back as file paths rather than content. That means it's not natively a "hand you the complete
HTTP message" kind of tool.

> {
>
> "mcpServers": {
>
> "chrome-devtools": {
>
> "command": "npx",
>
> "args": ["-y", "chrome-devtools-mcp@latest",
>
> "--isolated=true",
>
> "--allowedUrlPattern=https://*.target.example.com/*",
>
> "--proxyServer=http://127.0.0.1:8080",
>
> "--acceptInsecureCerts",
>
> "--redactNetworkHeaders",
>
> "--no-performance-crux",
>
> "--no-usage-statistics"]
>
> }
>
> }
>
> }

The one-line install is claude mcp add chrome-devtools --scope user npx
chrome-devtools-mcp@latest, but for authorized testing, use the parameterized version above — see the list of default behaviors at the end of this section for why.

#### Commonly Used Tools

| **Tool** | **What it does** | **What it's used for in testing** |
|----|----|----|
| take_snapshot | Gets the page's accessibility tree, with element UIDs | Confirm current state, locate elements. Subsequent click / fill calls reference the UID from here |
| evaluate_script | Executes a JS function in the page's context | Read identity fields from localStorage / sessionStorage, call the page's internal wrapper functions |
| list_network_requests | Lists requests since this navigation | Gets the real request list. Add includePreservedRequests to span across navigations |
| get_network_request | Gets details for a single request | With requestFilePath / responseFilePath it can write the message to a file — the only way to get the full body |
| navigate_page | Navigate | The initScript parameter can inject a script before the page loads, useful for hooking frontend functions |
| click / fill / fill_form | Input automation | Walk through business flows, fill in forms |
| take_screenshot | Screenshot | Evidence and self-checks. Not for locating elements — use snapshot for that |

#### Why Snapshot Matters More Than Screenshot

This is the efficiency gap that's easiest to overlook after installing the MCP, and it's not a small one.

A screenshot is pixels. The model has to do visual recognition first just to figure out where the login button is, and image tokens
scale with dimensions — a 1500×960 screenshot alone is one or two thousand tokens, and if it misreads it, it has to try again. A snapshot
is the accessibility tree — the model gets the element's role, accessible name, and reference
ID directly, and the next step can say precisely "click
uid=42" — for the same page, the structure tree usually costs only a few hundred tokens, with no ambiguity.

Practical convention: snapshot before each action to confirm state, snapshot again after to confirm the result, and only screenshot when you actually need to preserve evidence. If you do need a screenshot, remember image tokens
scale with dimensions, not file size — JPEG or WebP run three to five times smaller than PNG.

#### It Can't Modify Packets

This is the biggest pitfall when using it for security testing, and it's an officially documented limitation: there is currently no automated way to intercept and modify network requests through
Chrome DevTools MCP.

The underlying capability actually exists — CDP has a Fetch domain, and Puppeteer has a request-interception
API — but that capability isn't exposed as a tool by this MCP. There's a community PR working on it, and as of that issue no maintainer had weighed in. So don't count on it — architecturally you have to split the work:

| **DevTools MCP · Driving and Observation** | **Proxy · Modifying and Replaying** |
|----|----|
| Walk complete business flows: login, order-placement preconditions, multi-step forms | Write full traffic to disk, structured index, allowlist enforced at write time |
| Read internal page state: localStorage, frontend variables, DOM structure tree | Modify parameters and replay: Burp Repeater / mitmproxy replay |
| Perform deterministic verification: whether a payload actually executed | Hard outbound cutoff: out-of-scope connections killed outright |
| ✗ Intercept / tamper with / replay requests | ✗ Understand page state, drive multi-step flows |

Try to use them as one tool and the result is that neither job gets done well.

#### A Few Default Behaviors You Have to Know

**▪ The performance tooling sends URLs out.** Traces can get sent to Google's CrUX
API. In authorized testing, always add --no-performance-crux, or the target URL leaks out.

**▪ Telemetry is on by default**, and it's independent of Chrome
browser's own telemetry — opting out of one doesn't affect the other. Add --no-usage-statistics.

**▪ Can't run as root.** Chrome exits immediately when run as root — a common trip-up in containers and CI
images. Create an unprivileged user in the image and switch to it.

**▪ Sandboxes conflict with it.** With macOS Seatbelt or a Linux container sandbox enabled, it can't launch
Chrome — the workaround is to connect to a
Chrome instance launched manually outside the sandbox (--browser-url=http://127.0.0.1:9222).

**▪ Opening the remote debugging port carries risk.**
The official docs explicitly warn: any application on the machine can connect to that port and control the browser. Close it when you're done.

### 4.2 Who Handles the Modify-the-Packet Half

Three options, ordered by "official support + auditability."

#### Burp Suite: Has an Official MCP

PortSwigger's own extension, listed on the BApp Store, works with both Professional and Community
— only the two Collaborator-related tools are Pro-only. After building, load the JAR under Extensions
— it listens on 127.0.0.1:9876 by default.

| **Tool** | **Use** |
|----|----|
| get_proxy_http_history_regex | Filter proxy history by regex. The main way to have the model fish target endpoints out of tens of thousands of requests |
| send_http1_request / send_http2_request | Send an arbitrary request and get the response — this is what modify-and-replay relies on |
| create_repeater_tab / send_to_intruder | Send a request into Repeater or Intruder, kept in Burp for later human review |
| get_scanner_issues | Pull scanner findings and have the model pick which ones are worth digging into (Pro) |
| generate_collaborator_payload / get_collaborator_interactions | Out-of-band verification, needed for things like SSRF (Pro) |
| set_proxy_intercept_state / set_task_execution_engine_state | Toggle interception, pause the task engine |
| get_proxy_websocket_history_regex | WebSocket history, regex filtering |

#### Caido: No Official Version, but the Community One Is More Capable

Caido's own blog explicitly says they don't have an official MCP — what the blog covers is a community implementation that communicates with a local instance over GraphQL,
with the README listing 66
tools. Coverage spans request replay, session management, fuzzing, findings, interception, packet-modification rules, WebSocket, plus tools like
caido_is_in_scope that let the model do its own scope validation.

There's a point in Caido's own blog worth lifting straight into an internal standard: everything an agent does lands in Caido's
History, Sitemap, and Replay collections
and persists there, so a team can review it after the fact instead of being left with nothing but a chat log. For the auditability of authorized testing, that matters more than a few extra tools.

#### mitmproxy: Lightweight, Programmable, Good for Building Your Own

There's a community MCP implementation for mitmproxy, with tools including
set_scope, search_traffic, replay_flow, add_interception_rule, export_openapi_spec (reverse-engineering an
OpenAPI spec from traffic), and detect_auth_pattern. But the more common approach is skipping MCP entirely and just writing an addon as in 3.4
to write to disk as
jsonl, then exposing it to the model through your own retrieval service. More controllable, and you write the guardrails yourself. This is the path we use internally.

| **On that whole pile of "pentest MCPs"**　You can find dozens of MCP projects on GitHub wrapping nmap / nuclei / ffuf / sqlmap — neither ProjectDiscovery nor ffuf has an official MCP of their own. The vast majority of these projects are personal work with no security audit, and at their core they're handing the model arbitrary command-line execution. Before using one internally, at minimum confirm three things: are parameters escaped, is there command-injection protection, and is there a scope restriction. One well-known aggregator project has already been archived and is no longer maintained. |
|----|

### 4.3 Frontend JS White-Box: Turning Minified Code Into Two Tables

A modern SPA's
route table, permission checks, and encryption logic all live on the frontend. This is the closest thing to white-box material you'll get in black-box testing, and it requires no reverse engineering at all. Three approaches, ranked by output quality.

#### 1. Source Map Recovery

A lot of production apps ship .map files alongside their bundles. A source map is itself just JSON: sources
is the list of original file paths, sourcesContent inlines the original source directly, and mappings is a
Base64 VLQ-encoded position mapping. As long as sourcesContent
is present, what you recover is the real, pre-bundling source code — comments and all.

> go install github.com/denandz/sourcemapper@latest
>
> sourcemapper -output ./src -url https://target/assets/app.js.map
>
> sourcemapper -output ./src -jsurl https://target/assets/app.js #
> automatically follows the .map reference
>
> sourcemapper -output ./src -dir ./maps # batch mode
>
> # with auth / routed through a proxy for a paper trail
>
> sourcemapper -output ./src -jsurl https://target/app.js \\
>
> -header "Cookie: session=..." -proxy http://127.0.0.1:8080

What comes out is source code with its original filenames and directory structure. At this point you're back to ordinary white-box auditing, and can apply the three-stage process from 4.5
directly.

#### 2. AST Extraction (The Workhorse When There's No Map)

jsluice uses tree-sitter to parse the syntax tree and find places where a URL is known to get used — assigned to
document.location, passed to fetch(), or window.open(), for example.

What sets it apart from regex tools at a fundamental level is that it understands string concatenation: expressions that can't be statically evaluated get replaced with an
EXPR placeholder. A concatenated template-string call like fetch("/api/v2/users/" + id +
"/roles") gets recovered as /api/v2/users/EXPR/roles. This is enormously valuable when rebuilding an API
route table — plain regex can't do it, since regex only sees a pile of template-string fragments, while AST
sees a single route.

> go install github.com/BishopFox/jsluice/cmd/jsluice@latest
>
> jsluice urls --resolve-paths https://target app.js # structured JSON: url
> / queryParams / method / type / source
>
> jsluice secrets app.js

#### 3. Regex as a Fallback and Cross-Check

LinkFinder uses four regexes to match complete
URLs, absolute paths, and relative paths with and without a leading slash. It's nearly useless against concatenated paths after minification/obfuscation and is noisy, but it's still useful as a second cross-validation source — it occasionally catches something jsluice
missed. The Burp-side equivalent is the JS Miner
extension, which can passively scan and actively guess at .map files — higher false-positive rate, needs manual review.

#### What This Step Should Actually Produce: Two Tables

| **Table** | **Contents** | **Where it goes** |
|----|----|----|
| API inventory | Method, path (segments that can't be statically evaluated use EXPR), parameter names, the module that calls it | assets/routes.md |
| Identity-field classification | Which fields are server-issued vs. client-self-reported, and which endpoints the latter are used on | assets/identity.md |

The second table is the source for every broken-access-control hypothesis downstream, and it's also the detection entry point for the P-012
pattern from 3.3. With both tables done, testing stops being "not sure where to start" and becomes "verify row by row against the table."

| A role check on the frontend doesn't mean the backend has one too. Auth logic found on the frontend is only a lead — you must go back to the proxy side and do server-side verification. |
|----|

### 4.4 One Complete Broken-Access-Control Verification Chain

Stitch the previous three sections together, and a realistic-shaped closed loop looks like this:

| **Step** | **Who does it** | **Action** |
|----|----|----|
| ① | Model | Extract the route table with jsluice |
| ② | Model | Classify identity fields, find the client-self-reported ones |
| ③ | Model | Use DevTools to read localStorage / sessionStorage, dynamically confirm the field exists and is modifiable |
| ④ | Model | Modify the value on the proxy side and replay |
| ⑤ | Model | Response contains someone else's data → broken access control confirmed |
| ⑥ | **Human** | Stop immediately, take 1 piece of evidence, review independently |

The model walks through ① to ⑤ on its own; a human only does ⑥. The "stop immediately" in step ⑥
is enforced at the rules layer: proving broken access control only needs one record — continuing to pull more turns testing into data harvesting, and that's a difference in kind, not degree.

Step ③
deserves its own note, because it's the key to cutting false positives. A static conclusion says "this field is client-self-reported"; a dynamic check confirms "it actually exists and its value is modifiable" — only once both line up do you move to step
④. Going straight to packet modification off a static conclusion alone produces a much higher false-positive rate — dead code left behind by the bundler, or branches disabled by a
feature flag, both look real under static analysis alone.

### 4.5 Don't Let the Model Judge Whether There's a Vulnerability on Its Own

This section covers the 20% to 30% verification portion of the token budget. Web
is the easiest of the three form factors to make verification solid on, because the browser can give a deterministic answer.

#### Deterministic Verification: How XBOW Does It

XBOW became the #1 ranked account on HackerOne's US leaderboard in June 2025 — the first autonomous
AI to top it. They haven't published much architectural detail, but the verification step is described clearly: they built their own set of
validators that confirm each finding one by one.

XSS gets verified by having a headless browser visit the target and confirming whether the JavaScript payload
actually executed. It's not the model judging "this looks like it could be
XSS" — it's the browser producing a binary answer.

Carry that idea over to the workbench: whatever criterion can be written as a "deterministic check" should never be left to the model's judgment.

| **Vulnerability type** | **Deterministic criterion** |
|----|----|
| XSS | Whether the payload actually executed (headless-browser callback / DOM change) |
| Broken access control | Whether the response contains a business identifier that doesn't belong to the test account |
| SSRF | Whether the out-of-band channel received a callback |
| Arbitrary code execution | Whether the target process actually started (the case study in Part Five used a calc.exe popup) |

All of these can go straight into a task brief's hit criteria, written as decidable conditions rather than "looks suspicious."

#### Adversarial Self-Review: Have the Same Model Try to Disprove Itself

Andrew Hoffman's three-phase method for white-box auditing with Claude Code
is worth copying directly, especially phase two:

| **Phase** | **What happens** |
|----|----|
| Phase 0 · Setup | Pick the strongest model; configure it to read-only source, writes forbidden; disable telemetry so code doesn't leak into logs; run it on a throwaway git branch |
| Phase 1 · Recon | Map the structure, frameworks, dependencies, identify entry points and trust boundaries. Deliberately skip deep digging to save tokens, output to structured markdown, deliberately scoped to a small file set |
| Phase 2 · Three-Step Review | Find high-severity issues first, giving each finding a 0‒100 confidence score, a CVSS estimate, reproduction steps, and a unique ID; then the same agent flips its goal and tries to disprove its own findings; finally, a verdict, producing a report with OWASP / CVSS metadata |

He ran this against WordPress 4.7 and turned up CVE-2017-6818 (a DOM
XSS in tags-box.js), medium confidence, CVSS estimated at 5.4, correctly classified as CWE-79
. But he also documented an even more valuable failure: Claude cited mitigation filters supposedly present in 4.7
that actually only existed in later versions — a textbook example of "confident wrong analysis." Of the three findings in his initial report, two were disproven by the model itself during adversarial re-review.

Adversarial self-review works precisely because it exploits the "default compliance" bias from 1.6
. Asked directly "is this finding correct," the model tends to just confirm it for you; asked in reverse "prove this finding is wrong," that same tendency turns into an attack on its own conclusion. The cost is just one extra round of
tokens; the payoff is a noticeably lower false-positive rate. Worth running a few times with different phrasing.

#### A Range You're Free to Beat Up

XBOW open-sourced its validation-benchmarks: 104 CTF-style Web
challenges, each with a hidden flag, spun up with one Docker Compose command, with the
flag injected at build time rather than hardcoded. The repo itself warns to use it only in an isolated environment.

This is genuinely useful for rolling things out internally: a ready-made, legal, self-hostable range for evaluating your own
agent pipeline, tuning prompts, and getting a feel for the process — all without ever touching a real target. PentestGPT scored 86.5%
success (90/104) on this benchmark in a December 2025 experiment, which can serve as a reference baseline. Worth noting: as of mid-
2026
this benchmark has been flagged as dated, with performance mostly saturated — it's fine for practice now, not for proving who's stronger.

### 4.6 Trade-offs and Hard Limits

| **Advantages** | **Disadvantages** |
|----|----|
| Lowest barrier to entry — you can get the first closed loop running in half a day | WAF and rate limiting are the first things you run into — the model requests fast, easy to trip rate limits or even get an IP banned |
| The most complete tooling — traffic capture, debugging, automation all have ready-made solutions | Modern SPAs carry a lot of state, and the model occasionally gets lost in repeated re-renders |
| Frontend JS is ready-made white-box material — no reverse engineering needed | CSP and anti-automation scripts can break evaluate_script or get it detected |
| The highest degree to which the model can drive itself — clicking, filling, reading, judging, all unattended | Frontend encryption/signature fields will break replay — you have to reverse the algorithm first |
| Verification can be made deterministic, giving this form factor a naturally lower false-positive rate than the other two | DevTools MCP can't modify packets — you have to stand up a separate proxy layer |

| **A hard limit specific to Web: rate**　The model doesn't know that hammering too fast gets you banned. Lock "≥ 1.2 seconds between requests" into the rules file, and remember the point from 3.6: when multiple sub-agents run in parallel, the rate is calculated globally. Skip this and the typical outcome is that testing is only a third done and the egress IP is already blacklisted, with the rest of the work waiting on an IP change or the ban expiring. Authorized-testing time windows are usually limited, and that's what gets lost. When you hit rate limiting, stop — don't try to get around it. Bypassing rate control is itself usually outside the authorized scope. |
|----|

## Part Five: Client Side — Mini-Programs and Electron


------------------------------------------------------------------------

This class of target sits between Web and
APP: the code is bundled locally, but not compiled to machine code — extract it and you get readable
JS. They share one other trait too — auth tends to be weaker than on the
Web side, because developers default to assuming "I wrote the client, the user can't modify it." Nobody dares make that assumption on the Web
anymore, but it's still fairly common in mini-programs and Electron
apps. The case study at the end of this part comes down, at root, to exactly this assumption.

| **Section** | **What it covers**                                          |
|----------|-----------------------------------------------------|
| 5.1      | The structure and decryption of wxapkg                                 |
| 5.2      | Unpacking tools and code recovery                                  |
| 5.3      | Traffic capture: why you need Proxifier                          |
| 5.4      | Automation: UIA is the only path                              |
| 5.5      | Electron: asar, fuses, webPreferences               |
| 5.6      | MCP and trade-offs                                        |

### 5.1 The Structure and Decryption of wxapkg

The format is simple — the whole file is big-endian, a 14
-byte fixed header plus an index region, with the data region being plaintext, uncompressed raw content.

| **Section** | **Length** | **Content** |
|----|----|----|
| Magic number | 1 B | 0xBE |
| unknownInfo | 4 B | — |
| infoListLength | 4 B | Index region length |
| dataLength | 4 B | Data region length |
| End marker | 1 B | 0xED |
| Index region | infoListLength | fileCount, then per file: nameLen / name / fileOff / fileLen |
| Data region | dataLength | Plaintext, uncompressed |

Packages that PC WeChat writes to disk have an extra encryption layer on top of this: the first 6 bytes of the file header are the V1MMWX marker, followed by
1024 bytes under AES-256-CBC, with the rest under single-byte XOR.

> Key derivation key = PBKDF2(password = AppID, salt = b'saltiest',
>
> dkLen = 32, count = 1000, hmac = SHA1)
>
> IV b'the iv: 16 bytes' # fixed 16-byte string
>
> XOR key ASCII value of the second-to-last character of the AppID
>
> Concatenation originData[0:1023] + xorData # note: 1023, not 1024

That 1023 at the end is a classic implementation gotcha: the AES segment decrypts to 1024 bytes, but only the first
1023 bytes get used in the concatenation — write 1024
and the whole package fails to decrypt. Every unpacker that actually works is written this way — just copy it. The AppID is
the {wxid} directory name — everything needed for decryption is right there in the path.

| **A widely-circulated claim that needs correcting**　A lot of Chinese-language blogs describe the wxapkg header as little-endian <4sIII, with a magic of V1MM or wxsg, and a zlib-compressed data region. This doesn't match any actually-working unpacker implementation. V1MMWX is the marker for the PC-side encryption layer, not the package format's magic bytes; the data region isn't compressed either. Go with: starts with 0xBE, ends with 0xED, big-endian, plaintext data region. |
|----|

#### Where the Package Lives

WeChat 4.0 was a dividing line — the PC-side path moved from the Documents folder to AppData.

> Windows < 4.0
>
> `C:\Users\<user>\Documents\WeChat Files\Applet\<wxid>\<n>\__APP__.wxapkg`
>
> Windows >= 4.0
>
> `C:\Users\<user>\AppData\Roaming\Tencent\xwechat\radium\Applet\packages\<wxid>\<n>\__APP__.wxapkg`
>
> macOS >= 4.0
>
> `~/Library/Containers/com.tencent.xinWeChat/Data/Documents/app_data/radium/Applet/packages/<wxid>/<n>/`
>
> Android
>
> `/data/data/com.tencent.mm/MicroMsg/<userHash>/appbrand/pkg/`

#### The Thing Most Often Missed When Unpacking: Sub-Packages

The same `<wxid>/<n>/` directory often has multiple .wxapkg files. The main package has a 2MB
cap, so developers push low-frequency-but-sensitive functionality — admin backends, payment, real-name verification — into sub-packages. Unpacking only the main package gets you the wrong conclusion that "this mini-program barely has any endpoints," when the sub-packages are exactly where the valuable stuff lives.

The old wxappUnpacker requires unpacking the main package first, then passing -s pointing at the main package's output directory to unpack sub-packages, because
$gwx, shared styles, and app-config.json
all live in the main package. Newer tools (wedecode, unveilr) merge them automatically, but require the main and sub-packages to sit in the same directory.

| **After WeChat 4.1.x**　Some tools report the old decryption approach stopped working after 4.1.x, and wedecode claims to support 4.X. But we haven't found any public documentation stating exactly what changed in 4.x (a new magic value, or a new KDF), so no firm conclusion here. In practice: log which tool version works with which WeChat version in memory/tools.md, and check the table next time instead of retrying from scratch. |
|----|

### 5.2 Unpacking Tools and Code Recovery

| **Tool** | **Language** | **Status** | **Notes** |
|----|----|----|----|
| wedecode | Node.js | Active | Currently the community's top pick. Fully automatic, interactively scans local mini-programs, cross-platform, supports mini-games / plugins / sub-packages |
| unveilr | TypeScript | Active | Parses using a Babel AST rather than regex, more stable against newer obfuscation; on Windows, auto-extracts the AppID from the path and auto-decrypts |
| KillWxapkg | Go | Active | Single-file, pure Go; built-in -hook to enable F12, -repack to repackage, -sensitive to export sensitive info |
| wxappUnpacker | Node.js | Unmaintained | The ancestor of everything that came after — regex plus a hand-written parser. Often fails on newer package formats, but its DETAILS.md is still the format documentation |
| wux1an/wxapkg | Go | Archived | Archived 2026-02, author cited inability to keep up with WeChat updates, recommends switching to wedecode |

> npm i wedecode -g
>
> wedecode # interactive, auto-scans local mini-programs
>
> wedecode ./ --out output_path --clear --open-dir
>
> wedecode ./name.wxapkg --unpack-only # unpack only, no decompilation
>
> npm i unveilr -g
>
> unveilr wx /path/to/wxapkg/ -i wx11aa22bb33cc44dd -f

#### What You Get Out of It

A mini-program is a dual-thread architecture: WXML and WXSS live in the render layer (WebView), JS
lives in the logic layer (JSCore), and the two communicate via the WeChat client. The compiled output splits into two big chunks accordingly.

| **Artifact** | **What it is** | **How to use it** |
|----|----|----|
| app-service.js | All logic-layer business JS — each original file wrapped as define("path/to/x.js", function(require, module, exports){...}) | Parse these define calls and use the first argument as the path to write the function body back to disk. Variable names aren't recovered — what you get is "readable, but variable names are still a/b/c" |
| page-frame.html | The render-layer framework, containing $gwx (holds all the wxml) and setCssToHead | The easiest way to verify parsing is correct: drop it into Chrome, run $gwx("./pages/index/index.wxml") in the console, and it returns that template's virtual DOM |
| app-config.json | Global config plus per-page window config | The pages and subPackages fields inside are the first-hand source for the page route table |

#### Where the Route Table and Signing Function Are Hiding

**▪** The route table lives in three places: pages / subPackages in app-config.json; a centralized
config.js or api.js (look for constants like baseURL / BASE_API); and each page's
wx.request({url: ...}) call sites.

**▪** Don't hunt for the signing function file by file. Grep for keywords first (sign / hmac / CryptoJS /
nonce / timestamp / appSecret), then find the unified wrapper layer around
wx.request — the signature is almost always stuffed into a header right there, and tracing the call chain back from it will locate it.

**▪** The typical recovered shape is: parameters sorted lexicographically and concatenated, plus a timestamp, plus a fixed salt, then
MD5. This kind of algorithm recovery is something the model is quite good at — Part Seven has a matching prompt template, and the key line in it is "anything that can't be determined must be flagged, not filled in with a common-guess default."

### 5.3 Traffic Capture: Why You Need Proxifier

PC WeChat's mini-program render process, WeChatAppEx.exe, doesn't go through the Windows
system proxy. Set only the system proxy and Burp sees nothing. You need Proxifier
to force traffic into the proxy at the system level, per-process.

**1. Locate the process**　Find
WeChatAppEx in Task Manager, right-click to open its file location and get the full path. The main process is Weixin.exe (WeChat.exe in older versions) — adding it to the rule alongside the sub-process is more reliable.

**2. Proxifier rule**　Under Proxy Servers, enter 127.0.0.1:8080, protocol
HTTPS; under Proxification Rules, create a new one, put the
exe from the previous step under Applications, point Action at that proxy, and leave the default rule as Direct.

**3. Certificate**　Rename Burp's cacert.der to .cer, and import it via certmgr.msc
into Trusted Root Certification Authorities, not "Personal."

#### Does the Mini-Program Have Certificate Pinning

WeChat's official documentation is explicit: mini-programs can only communicate with domains registered as legitimate in the backend, only
https and wss are supported, and the server certificate must be issued by a trusted CA,
match the domain, be within its validity period, and have a complete chain — iOS doesn't support self-signed certs. This is standard CA
validation, not SSL pinning. So as long as Burp's CA is in the system trust root, PC-side
mini-program traffic can be decrypted.

The "don't validate legitimate domains, TLS version, or HTTPS
certificate" toggle in the developer tools is a development-time setting, only effective in the developer tools and real-device debug mode. We haven't found authoritative material proving the mini-program runtime does any additional
pinning. But note a separate issue: plenty of business teams do application-layer encryption in the mini-program's JS
layer — encrypting and signing the request body — so even with the packet captured, you can't read it. That's the signature-recovery problem covered in 5.2
, a different matter entirely from transport-layer pinning — don't conflate the two.

#### Enabling Debug Mode

Three routes, all community solutions with no official support, and all strongly version-dependent on WeChat.

| **Route** | **Method** | **Cost** |
|----|----|----|
| Ready-made tools | x0tools/WeChatOpenDevTools (Node, double-click the bat file, close WeChat before running), JaveleyQAQ/WeChatOpenDevTools-Python (main.py -x opens mini-program DevTools, -c opens the built-in browser), or KillWxapkg -hook | All do direct process hooking rather than going through a standard debug port, so each one lists which WeChat/mini-program versions it supports — wrong version, doesn't work |
| Memory patching | Use Frida to hook the mini-program's loading function and flip "enable_vconsole":false to true; then swap the DevTools UI from the stripped-down wechat_app.html to the full wechat_web.html | Requires doing it yourself; the manual fallback is searching memory directly for that string with Cheat Engine and overwriting it — watch the byte length. The address shifts between versions |

| **Another claim worth debunking**　Some sources online describe --disable-gpu as a WeChat debug switch. We found no evidence for this. It's the standard Chromium flag for disabling hardware acceleration, commonly used to fix black-screen screenshots — it doesn't enable any debug functionality on its own. |
|----|

### 5.4 Automation: UIA Is the Only Path

Wanting to automate real interactions in a mini-program (logging in, paging through, walking a business flow), the first instinct is usually to synthesize mouse and keyboard events. That path doesn't work. But the reason isn't quite what's commonly claimed, and it's worth getting precise, because it determines the countermeasure.

| **#** | **Fact** | **Explanation** |
|----|----|----|
| One | The OS level can distinguish synthetic input | Windows' low-level hook structure KBDLLHOOKSTRUCT.flags has two bits, LLKHF_INJECTED (0x10) and LLKHF_LOWER_IL_INJECTED (0x02), with a corresponding LLMHF_INJECTED on the mouse side. Any application installing a low-level hook to read this flag can distinguish real input from SendInput. This is an OS-provided capability, not something specific to WeChat |
| Two | PostMessage is even worse | Posted messages bypass the input system entirely — they don't trigger keyboard hooks or update GetKeyState. Any program doing multi-channel cross-validation will spot the inconsistency. Microsoft's own recommended fallback order is: UI Automation first, SendInput second |
| Three | This is the real current gotcha | Since 4.1.5, WeChat abandoned native Windows controls in favor of a self-built rendering framework, and it probes for an accessibility client: when no legitimate UIA client is detected, it only exposes a skeleton UI tree — the full control tree only gets built once it detects something like a screen reader has attached. Detection works by checking whether your process correctly references UIAutomationClient.dll and UIAutomationTypes.dll and has successfully attached |

So the precise way to put it is: it's not that "WeChat filters out SendInput" — it's that WeChat can distinguish synthetic input via
LLKHF_INJECTED, and 4.1.5+ hides the control tree by default from any unrecognized UIA
client. So coordinate-based synthetic input isn't reliable — you have to go through
UIA, and you first have to get WeChat to recognize you as an accessibility client. There are three publicly known approaches: write a minimal C#
UIA client that correctly references those two DLLs and
attaches; launch Narrator first, then run your script (poor compatibility); or do process-level configuration before launching your automation framework.

#### How to Use UIA

The key thing about UIA is that it doesn't simulate input at all — it calls the control's own exposed
provider interface directly. InvokePattern.Invoke()
is equivalent to executing the button's own "I was clicked" logic on its behalf, producing no mouse events the whole time, so none of the detection methods above apply.

| **Pattern** | **Use** |
|----|----|
| InvokePattern | Clicks a triggerable control, no mouse event produced. We've verified it reliably clicks buttons and grid icons in mini-programs |
| ValuePattern | Reads/writes a control's value directly — more reliable than simulated typing for filling in input fields |
| LegacyIAccessiblePattern | Exposes MSAA properties to UIA. A fallback for custom-drawn and non-standard controls — self-built frameworks like WeChat's often only expose this one |
| TextPattern | Reads rich-text content and embedded objects |
| VirtualizedItemPattern | Items in a virtualized list that haven't rendered yet — useful when paging through long lists |

> # 1. Locate the window — use the class name, not the title; the title changes per page
>
> win = auto.WindowControl(searchDepth=1,
> ClassName='Chrome_WidgetWin_0')
>
> # 2. Wait for rendering — mini-programs render asynchronously, the control tree shows up late
>
> # Don't sleep for a fixed duration — poll until the target control appears
>
> btn = win.ButtonControl(Name='My Orders')
>
> if not btn.Exists(maxSearchSeconds=8, searchIntervalSeconds=0.3):
>
> raise RuntimeError('Control did not appear, page may not have finished loading')
>
> # 3. Click — use InvokePattern, not coordinates
>
> btn.GetInvokePattern().Invoke()
>
> # 4. Input — ValuePattern is more reliable than simulated keyboard input
>
> win.EditControl(Name='Search').GetValuePattern().SetValue('test keyword')

Two stability lessons, both learned the hard way by fixing broken scripts:

**▪ Control naming isn't stable.** The same button's Name
can change between versions. Use relative positioning plus fuzzy text matching: locate a stable parent container first, then search within it by a text fragment. A hardcoded absolute path in a mini-program basically never survives one version update.

**▪ Search layer by layer, don't traverse from the root.** uiautomation
searching from the root for a deeply nested control can take hundreds of matches; narrowing searchDepth
layer by layer cuts that to a handful. Before writing a script, use the library's bundled automation.py -t 0
to print out the control tree and which Patterns each control supports.

#### Why the Official SDK Doesn't Work Here

WeChat has an official automation solution,
miniprogram-automator, and it's reasonably capable: controlling navigation, reading page data, triggering element events, injecting code into
AppService, calling any wx interface via callWxMethod, and stubbing with mockWxMethod.

But it requires projectPath
to point at a mini-program project that compiles cleanly in the developer tools — i.e., a source directory containing
project.config.json. So for testing your own mini-program, the official SDK
is the best option; for testing someone else's, all you have is the wxapkg, and the official SDK
won't work unless you decompile it into a compilable project and import that — and at that point the AppID
won't match, so cloud functions, payment, and authorization-related endpoints will all fail. Tencent also has a Python option,
Minium, which supports native controls (authorization popups, maps, camera), but it's likewise driven from the developer tools.

### 5.5 Electron

At its core, this is bundled Chromium. Find app.asar
in the install directory and unpack it to get the full frontend source. The format is extremely simple, uncompressed, and supports random access: an 8-byte
Pickle header gives the header length, the header itself is a chunk of JSON, and the rest is the data region.

> npx asar list app.asar
>
> npx asar extract app.asar ./unpacked
>
> npx asar extract-file app.asar main.js # extract just one file
>
> # Also check the app.asar.unpacked/ directory:
>
> # Files excluded at build time via --unpack go here — native .node modules are always here

Each file in the header JSON has an offset and size. offset is a
UINT64 as a string — because JS's Number is double-precision float with a safe-integer ceiling of
2^53, a large package would overflow it, so it has to be passed as a string. It's also relative to the data region, so computing the real position means adding the Pickle header length plus the
header length. Also, files with identical content are only stored once, with multiple entries pointing at the same
offset — don't be fooled when counting files.

| **Check the fuses before repackaging**　Since 16.0.0 (macOS) and 30.0.0 (Windows), Electron supports ASAR integrity validation, controlled by the EnableEmbeddedAsarIntegrityValidation fuse. Once enabled, it checks the hash in the header at startup and force-terminates the process if app.asar has been modified. Electron Forge 7.4.0+ and Packager 18.3.1+ configure this automatically. So for a modern Electron app, the "unpack, modify main.js, repack" route may simply not work. Run npx @electron/fuses read --app /path/to/App first to check the state before deciding which path to take. |
|----|

#### Focus Area One: webPreferences

This is a highly structured retrieval task, and the model does it very accurately. Have it find the
BrowserWindow constructor arguments directly in the unpacked output:

| **Setting** | **Official default** | **Why it matters** |
|----|----|----|
| contextIsolation | On by default since v12+ | If off, renderer-process JS can rewrite preload's objects and prototype chain — an XSS can escalate directly to RCE |
| nodeIntegration | Off by default since v5+ | If on, the renderer process can call require() directly. The official docs' own words: any renderer that loads remote content must never enable this |
| sandbox | On by default since v20+ | Uses OS-level capability restriction on the renderer process |
| webSecurity | On by default | Turning it off is equivalent to disabling the same-origin policy |
| nodeIntegrationInSubFrames | Off by default | If on, iframes also get Node capability — there's a history of RCEs from this |
| preload | Empty | preload has the full Node API. Even with contextIsolation on, if it exposes an interface via contextBridge that can forward an arbitrary IPC channel, a renderer-process XSS can still punch through |

That last row is the root cause of this part's case study, and worth remembering on its own: **contextIsolation
being on does not mean secure — it only isolates objects, not capabilities.** What actually determines the attack surface is what
preload exposes, and to whom.

#### Focus Area Two: Fuses

Fuses
are switches burned into the binary at build time, taking effect before signing, and irreversible once burned. A few directly relevant to testing:

| **Fuse**                      | **Default** | **Meaning**                         |
|-------------------------------|----------|----------------------------------|
| runAsNode                     | Enabled     | Can be abused as a general-purpose Node interpreter     |
| enableCookieEncryption        | Disabled     | Meaning cookies sit in plaintext SQLite |
| enableNodeCliInspectArguments | Enabled     | This is the precondition for enabling DevTools below     |
| onlyLoadAppFromAsar           | Disabled     | Doesn't block the "drop an app/ directory to override it" path |

#### The Main Process Is the Under-Audited Corner

Vendors often put sensitive logic — local encryption, certificate handling, auto-update, local service ports — in the main process, on the assumption that "the user can't see it." This code tends to be large in volume and never reviewed, making it a high-yield area. package.json's
main field is what points to it.

> # Method A: don't touch the package, add a launch flag (preferred, works on packaged apps too)
>
> open /Applications/YourApp.app --args --remote-debugging-port=8315 #
> macOS
>
> YourApp.exe --remote-debugging-port=8315 # Windows
>
> # Then open http://localhost:8315/ in a browser, or chrome://inspect
>
> # The main process needs the V8 inspector — the in-window DevTools can only debug the renderer process
>
> electron --inspect=9229 your/app
>
> electron --inspect-brk=9229 your/app # breaks on the first line of JS
>
> # Method B: through a proxy
>
> electron ./app --proxy-server=127.0.0.1:8080
> --ignore-certificate-errors
>
> # Method C: electron-inject, connects to the debug port and injects via CDP, doesn't touch the package
>
> pip install electron-inject
>
> electron_inject -d -t 60 - /path/to/application # -d enables F12 / F5

For an Electron app to be visible to UIA, you must add
--force-renderer-accessibility at launch. Same requirement applies to Chrome.

#### Electronegativity

A static-analysis tool from Doyensec that parses ASTs and the DOM to find security-relevant configuration, covering 38
checks in total. It can consume .asar
files directly, no need to unpack first. The project is no longer actively maintained, and the official page points to a commercial version.

> npm install @doyensec/electronegativity -g
>
> electronegativity -i app.asar -o out.sarif # SARIF plugs into CI / GitHub
> code scanning
>
> electronegativity -i /path/to/app -s HIGH -c HIGH

Anything in the output tagged _GLOBAL_ is a cross-file correlated check — for example,
HTTP_RESOURCES_WITH_NODE_INTEGRATION_GLOBAL_CHECK only fires when both "loads HTTP
resources" and "has
nodeIntegration enabled" hold true at the same time. These are the high-value findings — have the model prioritize them when reading the SARIF.

### 5.6 MCP and Trade-offs

The mini-program direction already has a few ready-made MCPs, all personal projects, not official: wxapkg-mcp (7
tools, one-shot from list_wechat_apps discovering the AppID to
decrypt_and_extract_appid), MCP-WEDECODEMCP (built on wedecode, a conversational flow of scan_local_miniapps →
decompile_scanned_miniapp → read_output_file), and e0e1-wx (a
GUI with an
MCP built in — package monitoring, auto-decompilation, CDP, cloud-function scanning, with covered mini-program versions listed in its config).

Electron doesn't have a dedicated MCP. A workable combination is: asar extract to disk, use a filesystem-style
MCP or just have Claude Code read the directory directly, then feed Electronegativity's SARIF
to the model for risk triage. That's exactly the path the case study in 5.7 takes, with no dedicated tooling at all.

| **Advantages** | **Disadvantages** |
|----|----|
| Unpacking is white-box by nature — JS source is directly readable and feedable to the model | Automation has only one path, UIA, and it requires activating accessibility mode first |
| Client-side auth tends to be weak across the board — broken access control and info leaks are common | Unpacking scripts are strongly WeChat-version-dependent, and changes since 4.1.x have no public documentation |
| Mini-programs use standard CA validation, not pinning — get the cert into the system root and you can capture traffic | The UIA control tree drifts between versions, so scripts aren't cheap to maintain |
| Environment cost is much lower than APP — no rooted device needed | Application-layer encryption/signing is common, so a captured packet still may not be readable |
| Auditing Electron config is a structured task the model handles accurately | Electron's integrity validation can shut down the repackaging route entirely |

## Part Six: APP Side


------------------------------------------------------------------------

The largest attack surface of the three: local storage, exported components, WebView, native
.so libraries, and communication encryption are all in play. The cost is the heaviest environment — a rooted
Android device or emulator is the baseline requirement. Precisely because of that, it's worth doing this one last.

| **Section** | **What it covers**                                         |
|----------|----------------------------------------------------|
| 6.1      | Static: jadx and apktool each cover half                     |
| 6.2      | MCP: xref is what the model actually needs                         |
| 6.3      | Frida: server or gadget                          |
| 6.4      | Three things about traffic capture: CA, NSC, pinning                       |
| 6.5      | Hardening and unpacking                                         |
| 6.6      | Attack-surface checklist                                         |
| 6.7      | Closing the loop, trade-offs, and a comparison across all three form factors                           |

### 6.1 Static: jadx and apktool Each Cover Half

Run both — the division of labor is clean: jadx is for reading and understanding, apktool is for modifying and rebuilding.

|  | **jadx** | **apktool** |
|----|----|----|
| dex output | Java-like pseudocode, highly readable, may be inaccurate or drop method bodies | smali, byte-for-byte faithful, modifiable and rebuildable |
| Can it repackage | No | Yes — must be re-signed after apktool b |
| Typical use | Reading logic, finding class/method names, feeding the model and Frida | Modifying network_security_config.xml, editing the manifest, injecting a gadget, patching smali |

#### Spend Ten Seconds First Checking for Hardening

Don't jump straight into waiting on a full decompile. First dump resources only, or just unzip -l to check the filenames in lib/ and
assets/. The mainstream commercial hardening/packing solutions all have distinctive filename signatures (things like libDexHelper.so, ijiami.*, libshell-*.so, libjiagu*.so, libxloader.so, libegis.so
). If you spot hardening, jump straight to 6.5 — don't waste half an hour.

> jadx -s -d out_res target.apk # resources + manifest only, seconds
>
> unzip -l target.apk | grep -E 'lib/|assets/'

#### A Few Key jadx Flags

| **Flag** | **Why add it** |
|----|----|
| --show-bad-code | A must for obfuscated or hardened packages. jadx drops method bodies it fails to decompile by default — without this you see a bunch of empty methods and might think the developer never wrote anything there |
| --deobf | Renames short obfuscated identifiers. Pair with --deobf-cfg-file to save the mapping for reuse next time, and to stay in sync with teammates on symbol names |
| --no-inline-methods | Add this when preparing to write a Frida hook. After inlining, method names no longer match runtime, and your hook point won't be found |
| --no-imports | Writes out full package names everywhere. Model-friendly — eliminates ambiguity from same-named classes |
| --output-format json | Structured output — easier to feed into an LLM pipeline than raw Java text |
| --call-graph json | Exports the whole app's call graph. The ideal input when doing call-chain analysis |
| --single-class | Decompiles just one class. Use it when the model says "I want to look at this class" — orders of magnitude faster than the whole package |

> # A practical combo for hardened / obfuscated packages
>
> jadx -d out --show-bad-code --deobf --no-imports --no-inline-methods
> -j 8 target.apk

jadx's own docs are explicit: for obfuscated or hardened APKs,
it does not guarantee a complete decompile. That's the official stance, not community griping — plan your time budget around that expectation.

#### Using MobSF as a Baseline Scanner

Its value is triage, not conclusions. One command gets you a baseline report, and having the model pick what's worth digging into from that report saves a lot compared to having it read everything from scratch. Spin it up with
Docker — static analysis doesn't need a connected device:

> docker run -it --rm -p 8000:8000
> opensecurity/mobile-security-framework-mobsf:latest
>
> # default mobsf/mobsf, the API key is on the instance's own /api_docs page
>
> curl -F 'file=@target.apk' http://localhost:8000/api/v1/upload -H
> "Authorization: KEY"
>
> curl -X POST http://localhost:8000/api/v1/scan --data "hash=<hash>"
> -H "Authorization: KEY"
>
> curl -X POST http://localhost:8000/api/v1/report_json --data
> "hash=<hash>" -H "Authorization: KEY"

Dynamic analysis needs a connected adb device — the official docs state support only up to Android 11 (API 30), not 12
and above. It also turns two annoying tasks into
APIs, which is very convenient when hooking up an automated pipeline: /api/v1/android/root_ca (install/remove system CA) and
/api/v1/android/global_proxy (set/cancel global proxy). There's also a set of
/api/v1/frida/* endpoints, which lets you use MobSF as a Frida orchestrator.

### 6.2 MCP: xref Is What the Model Actually Needs

The most mature piece in this chain. jadx-ai-mcp is a Java
plugin that runs inside jadx-gui (default port 8650), and jadx-mcp-server is the Python-side MCP server (default
8651) that connects to it.

> jadx plugins --install "github:zinja-coder:jadx-ai-mcp"
>
> uv run jadx_mcp_server.py --http --jadx-host 127.0.0.1 --jadx-port
> 8650

It exposes thirty-plus tools, ranked by value:

| **Group** | **Tools** | **Value** |
|----|----|----|
| Cross-references | xrefs_to_class(), xrefs_to_method(), xrefs_to_field() | The single most valuable group. This gives the model the ability to do taint tracing — working backward from a sensitive field to find every place that references it. Plain grep can't do this, because grep doesn't understand inheritance and overloading |
| Getting code | get_class_source(), get_method_by_name(), get_methods_of_class(), get_fields_of_class() | Paired with the indexing strategy in 3.4 — the model looks at the listing first, then decides which method to read |
| Android-specific | get_android_manifest(), get_main_activity_class(), get_manifest_component(), get_strings() | Entry points |
| Bytecode | get_smali_of_class() | The fallback when decompilation fails — you can still read smali when Java pseudocode doesn't come out |

There's a key design point: it talks to a running jadx-gui
instance, rather than re-running decompilation itself. The upside is you get
xref and an already-loaded symbol table; the downside is you have to manually open the APK in the
GUI first, and the project's own README says it's still early-stage and does crash.

The same author also has apktool-mcp-server, with 13 tools. Note that it includes
modify_smali_file() and build_apk(), meaning the model can directly modify smali
and repackage. When wiring this up, put permission-level constraints around it — don't let it silently modify your sample.

### 6.3 Frida: Server or Gadget

Static analysis can only tell you "there's a check like this in the code" — it can't tell you "did it actually execute at runtime, and with what parameters." An APP
has a lot of logic gated behind multiple branches, feature flags, or that's flat-out dead code. Frida
turns a static "maybe" into a runtime "actually did" — this step is the main lever for controlling false-positive rate on the APP form factor.

|  | **frida-server** | **frida-gadget** |
|----|----|----|
| Form | An independent process on the device, usually run as root | A shared library, loaded inside the target process |
| Precondition | Needs root | Doesn't need root, but needs the APK modified or LD_PRELOAD |
| Process visibility | Can attach / spawn any process | Only one "Gadget" entry shows in the process list |
| Fits | Rooted test devices — most efficient | Non-rooted devices, or when you need to evade port/process fingerprint detection |

Gadget has four interaction modes, of which Script
mode is the most useful for automation: it auto-loads and executes a standalone JS file, with no
host-side connection needed. Listen is the default mode, and it blocks until something attaches.

| **A very easy naming rule to trip on**　Gadget's config filename equals the binary's name plus .config. But on Android, for a non-debuggable app, the config filename must start with lib and end with .so, because only .so files under lib/ get unpacked from the APK. Getting the name wrong shows up as "the config just doesn't take effect at all," with no error whatsoever — very hard to debug. |
|----|

#### Commonly Used APIs

Java-layer — the first two are everyday tools, the last two are lifesavers for hardened packages:

| **API** | **Use** |
|----|----|
| Java.use(name) | Gets a class wrapper, for hooking methods or instantiating objects. The most commonly used |
| Java.choose(name, cb) | Enumerates live instances on the heap. Used to get field values of objects that already exist — an already-initialized client, an already-decrypted config object. Static analysis can't get at these |
| Java.enumerateLoadedClasses() | Lists all loaded class names. The first tool for confirming the dex actually loaded after unpacking, and for locating obfuscated classes |
| Java.enumerateClassLoaders() | Essential for hardened packages. The shell's classes and the real dex often sit in different ClassLoaders — when Java.use can't find something, swap Java.classFactory.loader to the target loader first |

> const libc = Process.getModuleByName('libc.so');
>
> Interceptor.attach(libc.getExportByName('read'), {
>
> onEnter(args) { this.fd = args[0].toInt32(); },
>
> onLeave(retval){ if (retval.toInt32() > 0) console.log('read',
> this.fd); }
>
> });
>
> // computing an offset-within-.so hook point
>
> const base = Module.findBaseAddress('libnative.so');
>
> Interceptor.attach(base.add(0x1234), { ... });
>
> // have the script output structured JSON via send(), collected on the host side — this is the standard way to connect it to the model
>
> send({ type: 'sign', input: argStr, output: retStr });

The newer official Frida examples have moved from Module.findExportByName() to
Process.getModuleByName().getExportByName() — both still work during the transition period. Before writing a script, confirm the
frida version on the target device first — this is, without exception, the most common reason "a script copied from the internet doesn't run."

#### Why Frida Can Only Modify Method Boundaries

This point directly shapes the APP prompt template in Part Seven, and it's worth explaining the mechanism clearly.

A Java-layer hook works by replacing the method implementation: Frida gets the target method's
ArtMethod and points its entry at its own bridge code, taking over the original method wholesale. The native-layer
Interceptor.attach modifies instructions at the function entry point and jumps to a trampoline, with onEnter /
onLeave
inserted at entry and return respectively. Both mechanisms operate at the granularity of **a single call's boundary** — parameters coming in, return value going out.

A given if statement somewhere inside a method's body is neither an entry point nor an exit point — Frida
has no insertion point there. Modifying it means patching instructions directly, which is no longer a hook, it's a patch.

So when looking for a hook
point, look for a method boundary: the ideal target is a standalone validation method that returns a boolean, not an
if buried in the middle of a large function. When having the model read jadx output, explicitly require it to output the **fully-qualified class name + method name + parameter signature +
overload + return type**, not "there's a check here." The former lets you generate
Java.use(...).method.overload(...).implementation =
... directly; the latter means you have to go back and dig through it yourself.

#### Locating JNI Dynamic Registration

Many commercial APPs register native methods dynamically via RegisterNatives, so jadx only shows you the
native declaration, not the implementation address. Two tools handle this:

**▪** Hook
RegisterNatives (lasting-yang/frida_hook_libart). Its output gives you the offset within the .so
directly, which you can then take to Ghidra or IDA to locate.

**▪** jnitrace. jnitrace -l libnative-lib.so <pkg> shows the method names, parameters, buffer hexdump, and return values from jni.h
. -o outputs
JSON — important for connecting it to the model, since it can go straight into the index.

### 6.4 Three Things About Traffic Capture: CA, NSC, Pinning

These three get conflated constantly, but they're three independent problems, and the fixes differ too.

| **Problem** | **Symptom** | **Fix** |
|----|----|----|
| ① The system doesn't trust your CA | On Android 7+, apps with targetSdk ≥ 24 don't trust user-added CAs by default. The official wording is "by design" | Get the CA into the system trust store |
| ② The app configures its own NSC | The app uses <trust-anchors> to precisely control which sources it trusts, and <pin-set> for certificate pinning. <debug-overrides> only takes effect when android:debuggable="true" | Modify the NSC and repackage, or use Frida |
| ③ The app does pinning in code | OkHttp or a homegrown implementation. Installing a system CA doesn't fix this | Frida is the only option. That NCC report puts it bluntly: an app with SSL pinning cannot be intercepted no matter what you do |

| **Android 14 moved the trust store**　Starting with Android 14, CA validation no longer reads /system/etc/security/cacerts — it goes through the Conscrypt APEX module instead, reading from /apex/com.android.conscrypt/cacerts. The old Magisk CA module simply stops working on Android 14. The corresponding new approaches (NCC's ConscryptTrustUserCerts, TrustAnyCert, etc.) work by mounting the certificate into Conscrypt's namespace and running as a late_start service — the timing here matters a lot, since the mount must happen after Zygote comes up, or the app process will still see the old mount view when it forks. Log the test device's OS version into memory/tools.md — this is the single easiest place to get stuck for half a day on a new target. |
|----|

#### The Cost of the Modify-NSC-and-Repackage Route

> apktool d -f -o work target.apk
>
> # 1. Edit work/res/xml/network_security_config.xml, add <certificates
> src="user" />
>
> # 2. Add to <application>:
> android:networkSecurityConfig="@xml/network_security_config"
>
> apktool b work -o patched.apk
>
> zipalign -p -f 4 patched.apk aligned.apk
>
> apksigner sign --ks my.keystore aligned.apk

Repackaging breaks the original signature, at the cost of: triggering signature-verification defenses, losing Play Integrity, and breaking App
Links' assetlinks.json match. Prefer installing a system CA — only fall back to this route on a non-rootable
device.

#### What objection Is Actually Hooking

android sslpinning disable isn't "installing a certificate" — it's using Frida
to turn the validation function into an always-true or no-op implementation. It hits seven points, worth knowing, because when it doesn't work you need to know which layer you're missing:

| **Hook point** | **What it does** |
|----|----|
| javax.net.ssl.SSLContext.init() | Swaps out the trust source, plugs in a no-op TrustManager |
| okhttp3.CertificatePinner.check() and check\$okhttp() | Skips the pin comparison, returns directly without throwing |
| ...conscrypt.TrustManagerImpl.verifyChain() | The key point on Android 7+, skips chain validation, returns the passed-in chain as-is |
| ...conscrypt.TrustManagerImpl.checkTrustedRecursive() | Returns an empty list |
| The corresponding methods in Appcelerator / PhoneGap plugins | Dedicated paths for cross-platform frameworks |

The three layers cover different implementation paths — swapping the trust source, skipping chain validation, skipping pin
comparison — so all of them need to be hit. A more engineered approach is httptoolkit's
set of scripts, and two things about its design are worth learning from: native-tls-hook.js modifies BoringSSL
to trust the certificate you've configured, rather than disabling validation entirely (less likely to trigger the app's error-handling branches); and a
fallback script that, when it detects a certificate-validation failure, automatically generates a patch for an obfuscated, unrecognized
pinning implementation.

Flutter goes through BoringSSL rather than the system CA, and needs to be handled separately.

### 6.5 Hardening and Unpacking

Identify the generation first, because the approach differs completely by generation, and the effort-to-payoff ratio varies a lot too.

| **Generation** | **What it does** | **Unpacking route** | **Effort** |
|----|----|----|----|
| 1st gen · Whole-DEX packing | Full DEX encrypted, leaving only a shell app | Memory search / dump to disk is enough | Low |
| 2nd gen · Method extraction | Method bodies are stripped out, filled back in at runtime on demand | Catch the decryption moment: actively call methods to force them to fill back in, then intercept at the ART layer | Medium |
| 3rd gen a · VMP | Custom instruction set + custom interpreter | Have to find the interpreter, recover the instruction mapping | Extremely high, usually outside an authorized-testing time window | 
| 3rd gen b · Dex2C | Java logic translated to C | Essentially unrecoverable back to Java — focus shifts to the native layer, looking at jni.h-related calls | High, and at this point dynamic analysis is far more worthwhile than static |

The first step to identifying the generation is checking the filename signatures from 6.1
; the second is, after unpacking, using
--show-bad-code to see whether method bodies are still empty.

| **Tool** | **Needs root** | **Mechanism** | **Notes** |
|----|----|----|----|
| frida-dexdump | Yes | Memory search for DEX signatures | -d deep scan can find DEX with a corrupted or fragmented header. The repo was archived in July 2023 — a community fork may be needed on newer environments |
| BlackDex | No | DexFile cookie extraction | Runs on ordinary consumer devices and emulators, supports Android 5.0‒12. deep mode attempts to fill back in extracted method instructions, but can take several minutes and raises the failure rate |
| FART | Needs a custom ROM | Hooks the interpreter + active method invocation | Patches ART source and recompiles the ROM. The core technique against 2nd-gen packing: actively call every method to force the shell to decrypt and fill it back in, then intercept at the ART layer. Paired with dexfixer and fart.py for reassembly |

#### How to Use the Unpacked Output

**▪** 1st-gen packing produces a set of complete DEX files. You can run jadx -d out
dump_dir/*.dex directly, but a better approach is to swap them back in for the original classes*.dex
inside the APK and zip it back up — that way resources, the manifest, and R classes are all still there, and jadx can correctly resolve @string/xxx
references.

**▪** The
dump will mix in the shell's own classes along with a lot of duplicate classes. Sort by size and class count first, and discard the ones that are obviously the shell.

**▪** 2nd-gen packing produces a DEX plus scattered CodeItems, which need to be reassembled by
method_idx. This step is highly dependent on the specific shell and tool output format — there's no cross-tool generic solution.

There are two ways to verify whether reassembly succeeded: open it with jadx --show-bad-code
and check whether the key method bodies are still empty or just throw new
UnsupportedOperationException; and, at runtime, cross-check with
Java.enumerateLoadedClasses()
to confirm the unpacked output's set of class names matches what's actually loaded at runtime. If a large number of method bodies are still empty, that means either 2nd-gen packing didn't fill back in successfully, or you're already dealing with 3rd-gen.

#### Anti-Debug and Anti-Frida

Seven common detection techniques, ranked from easiest to hardest to counter:

**1.** Scanning the default port 27042 and confirming with a D-Bus handshake

**2.** Scanning /proc/<pid>/maps for strings like LIBFRIDA, frida-agent

**3.** Enumerating thread names for gum-js-loop, gmain, pool-frida

**4.** Reading /proc/<pid>/fd/* for named pipes created during frida injection

**5.** Reading TracerPid

**6.** Forking a child process that ptraces the parent to occupy that slot, so no other debugger can attach

**7.** Comparing the .text segment in memory against the .so on disk

The first six can all be gotten around by changing the port, changing process names, recompiling to strip signatures (that's exactly what the strong-frida
patch set does). The seventh can't — the author of detectfrida
explicitly points out this detection is frida
-agnostic, and changing signature strings doesn't get around it. When assessing how hard the countermeasures will be, whether the target has this one is the dividing line for "is this even worth the effort."

| **A practical piece of advice**　This is an ongoing arms race — expecting to "grab a public script and blast through it" isn't realistic. The right approach is to first use jnitrace or hook_RegisterNatives to locate the detection points, then write a targeted hook. Also, spawn mode (frida -U -f <pkg>) can get around some detection, because detection code often only registers after the app has started, while spawn injects earlier than that. A lot of RASP implementations run detection in a loop on a separate thread — hooking pthread_create to patch the detection thread before it actually starts running is an effective point of attack. |
|----|

### 6.6 Attack-Surface Checklist

This part is structured retrieval, and having the model run over jadx output here is highly efficient. Ranked by likelihood of yielding something.

#### Manifest

| **Attribute** | **Default** | **What to look at** |
|----|----|----|
| android:debuggable | false | If true, a debugger can attach, and run-as can read the private directory directly. Finding this in a release build is itself a problem |
| android:allowBackup | true | adb backup can export app data. If it stays true, dataExtractionRules should be configured to exclude sensitive data |
| android:usesCleartextTraffic | false since API 28+ | Note: on Android 7.0+, this attribute is ignored when an NSC is present. Officially marked as being deprecated |
| android:exported | Must be explicitly declared when there's an intent-filter | When true, any app can launch it with an explicit ComponentName, even without matching the intent-filter |
| android:taskAffinity | Package name | Combined with singleTask, affects activity reparenting — the root of task-hijacking-class issues |
| android:extractNativeLibs | Depends on minSdk | When false, .so files aren't extracted and load directly from the APK, which affects whether you can pull the .so from /data/app/.../lib/ |

> # Look up the deeplink scheme
>
> adb shell dumpsys package <pkg> | sed -n '/Schemes:/,/Non-Data
> Actions:/p'
>
> # Trigger it, then confirm it actually navigated — the "Starting Intent" line alone isn't enough
>
> adb shell am start -a android.intent.action.VIEW -c
> android.intent.category.BROWSABLE \\
>
> -d '<scheme>://<host>/web?url=https://example.com'
>
> adb shell dumpsys activity activities | grep mResumedActivity
>
> # Explicitly launch a non-exported or hidden Activity
>
> adb shell am start -n <pkg>/.SomeActivity

The typical shape of a deeplink issue is xxx://host/web?url=<attacker-controlled>, feeding the URL
parameter straight into a WebView. The case study in 6.8 is a variant of this, and a more subtle one — the route wasn't declared in the
manifest at all.

#### WebView

The fatal combination is setJavaScriptEnabled(true) plus
setAllowUniversalAccessFromFileURLs(true), then loading an externally-controllable URL — especially one that came from a
deeplink parameter.

> grep -rnE "addJavascriptInterface|setAllowUniversalAccessFromFileURLs|setAllowFileAccessFromFileURLs|setAllowFileAccess|onReceivedSslError|shouldOverrideUrlLoading|loadDataWithBaseURL" out/sources/

Calling handler.proceed() inside onReceivedSslError
is equivalent to unconditionally accepting any certificate. setAllowFileAccess switched from defaulting to true
to false starting with API 30 — check targetSdk for older apps.

#### Local Storage and Native

| **Location** | **What to watch for** |
|----|----|
| SharedPreferences | /data/data/<pkg>/shared_prefs/*.xml, plaintext XML |
| SQLite | databases/, unencrypted by default. Don't forget the -journal file — deleted data may still be sitting in it |
| External storage | Globally readable/writable; not removed on app uninstall if it's outside the app's own directory |
| Realm | Defaults to files/default.realm, can be encrypted but the key is often hardcoded |
| Hardcoded keys | Run trufflehog over static strings as a first pass; but hooking the SecretKeySpec constructor at runtime is more reliable, since keys are often assembled at runtime |

### 6.7 Closing the Loop, Trade-offs, and a Comparison Across All Three Form Factors

| **Stage** | **Who does it** | **Output** |
|----|----|----|
| jadx decompile + build index | Human kicks it off | INDEX.tsv / INDEX-keywords.md |
| Read index + xref | Model | List of hookable methods (full name + signature + overload) |
| Frida verification | Model writes the script, human injects it | Runtime echo-back |
| Proxy parameter modification + replay | Model | The evidence three-piece set |
| Ledger | Model writes, human reviews | Both hits and misses recorded |

The model is the thread that strings these three tools together: reading decompiled output to form hypotheses, writing hook
scripts to verify them, reading traffic-capture output to reach a conclusion. The human handles injection, review, and calling a halt.

#### Ready-Made AI Wrappers

By 2025-2026
a few clear paths have already emerged, all worth a look before deciding whether to build your own:

| **Path** | **Example** | **Notes** |
|----|----|----|
| MCP tool-mounting (most mainstream) | jadx-ai-mcp + jadx-mcp-server + apktool-mcp-server + frida-mcp | The typical loop: the model reads the manifest, searches classes, queries xref, reads smali, then generates a Frida script, injects it via frida-mcp, the script echoes runtime data back with send(), and the model iterates on the result |
| Claude Code skill | incogbyte/android-reverse-engineering-claude-skill | A packaged seven-stage pipeline: dependency check → decompile → architecture analysis → security audit → API extraction → call-flow tracing → adaptive Frida bypass (analyzes crash logs, generates targeted hooks based on the actual code, iterates until it gets past the check) |
| Workspace-style orchestration | TheQmaks/areclaw | 14 tools plus 21 Python packages, using Claude Code as the orchestrator. Integrates jadx / apktool / Ghidra / Frida (15 built-in scripts) / multiple deobfuscators / trufflehog, and provides skills like /analyze-apk, /find-api, /intercept, /compare-versions |

/compare-versions deserves its own mention: version diffing
is a task the model is very good at and humans find painful. Taking a semantic
diff between two versions' jadx output to locate newly-added or modified security-relevant logic will often point directly at "what the vendor has been changing lately" — which is usually exactly where the newest issue was introduced.

| **Advantages** | **Disadvantages** |
|----|----|
| The widest attack surface — local storage, components, network, and the native layer are all in play | The highest environment barrier — needs root, Frida installed, and handling CA and pinning |
| Feeding decompiled output to the model for white-box work gets results close to an open-source project audit | Hardening breaks jadx outright, and unpacking is mostly manual |
| The model is very accurate on structured data like the manifest | Anti-debug detection is increasingly common and an ongoing fight |
| xref tooling lets the model do taint tracing, something grep can't do | Output volume is large — skip indexing and context blowup is guaranteed |
| Frida hooks are extremely flexible, and the model writes scripts far faster than a human | Android 14 moved the trust-store location, breaking every old approach |

#### Comparison Across the Three Form Factors

| **Dimension** | **Web** | **Client** | **APP** |
|----|----|----|----|
| Environment cost | Half a day | One day | Two to three days |
| Degree of white-box access | High — frontend JS is ready-made | High — unpacking is source code | Medium-high, affected by hardening |
| How much the model can drive itself | Highest — clicking, filling, reading, judging, fully automated | Medium — UIA can drive it, scripts need maintenance | Medium — model writes scripts, human handles injection |
| Main adversary | WAF · rate limiting · CSP | Synthetic input getting swallowed · package encryption | Pinning · hardening · anti-debug |
| Degree of weak auth | Medium — scrutinized the most | High — "I wrote the client" | Medium-high — lots of local logic |
| Verification determinism | High — the browser can give a binary answer | Medium | Medium — Frida can confirm, but injection has to succeed first |
| Time to first finding | Half a day to a day | One to three days | Three to seven days |

| **Order to tackle them: Web → mini-program → APP**　Run Web first — it produces results in half a day, builds confidence that "this whole approach actually works," and lets you shake out the form-factor-agnostic pieces (rules layer, ledger layer, sub-agent orchestration) in the lowest-friction environment. Mini-programs second — practice unpacking and reverse-engineering instincts, your first time facing automation countermeasures, but environment cost is still low. APP last. Going straight at APP first will most likely get you stuck on environment setup, with nothing to show after two or three days — and that's when the team loses confidence in the whole method. This is the single most common failure mode seen in practice, and it has nothing to do with whether the method itself is right. |
|----|

## Part Seven: Prompting


------------------------------------------------------------------------

Picking the right slice and building the threat model right just gets you into the right neighborhood. Once you're there, how you phrase things directly determines whether you get back a pile of vague observations or one usable finding. The ten techniques in this part come from
Needle in the Haystack — we've verified several of them in the field, and the difference in effect was bigger than expected.

### 7.1 One Principle: Search Mode, Not Evaluation Mode

Every technique points at the same thing: pushing the model from evaluation mode into search mode.

|  | **Evaluation mode** | **Search mode** |
|----|----|----|
| How you ask | "Does this function have a vulnerability?" | "This function definitely has two or three issues, find them" |
| The model's task | Classification (yes / no) | Generation (find the specific locations) |
| The path of least resistance | "Overall looks secure" | Doesn't exist — it has to go find something |
| How it reads code | A skim | Line by line |
| Output | A list of unprioritized theoretical concerns | Specific locations + reachability conditions |

Same piece of code, different phrasing, and what you get back isn't even the same order of magnitude. This isn't mysticism — you've changed its optimization target: a classification task has a low-cost default answer, a generation task doesn't.

### 7.2 Get the Threat Model Out of It First

Do this before using any technique. It corresponds to the under-10%
scaffolding portion of the token budget, and it's also the anchor for every question that follows. The method is to have the model generate it based on historical
CVEs, rather than you writing it off the top of your head.

> Below are the historical CVE descriptions for [project name]:
>
> [paste the description text from GitHub Security Advisories / NVD / vendor advisories]
>
> Based on the types and patterns of these vulnerabilities, output a one-page threat model that must include:
>
> 1\. The most common vulnerability categories, ranked by historical frequency
>
> 2\. Which trust boundaries are weakest, and the basis for that
>
> 3\. A list of entry points: HTTP routes / RPC handlers / message queues / file uploads /
> deserialization
>
> 4\. The attacker model: who they are, what credentials they have, what input they can control
>
> 5\. A list of high-severity operations: which operations have the worst consequences if triggered without authorization
>
> Don't write generic security advice. Every point must be traceable back to the CVE descriptions above.
>
> Keep the output to one page.

This step's output goes straight into
targets/<slug>/threat-model.md. Its value is filling in the context for every question that follows — as 1.1
said, this is exactly the conditioning term the model is missing.

If the historical CVEs
are all heap overflows and integer overflows, build the threat model around memory corruption; if they're all authorization bypasses, build it around the permission model. Digging in the same direction means what you find is more likely to be accepted by the maintainers too, because that's genuinely where this project's structural weakness is.

#### Another High-Yield Entry Point: Auditing the Fix Patch

Find the commit that fixed some historical vulnerability, and have the model analyze whether that
patch can be bypassed. Fix code often only plugs the specific entry point mentioned in the report, leaving other paths with the same root cause still open. Anthropic
did exactly this when auditing Firefox CVEs: take the fix commit, have Claude check for what got missed.

The upside of this entry point is that the scope is naturally narrow — a commit touches a handful of files, and the slice is ready-made.

### 7.3 Ten Techniques

Ranked by effectiveness — the first three are the ones most worth learning first.

#### 01　Assert the Vulnerability Exists

The technique with the clearest effect. Stating directly "there are definitely two or three security issues here" produces noticeably higher-quality analysis than asking "is there a vulnerability."

The model has a strong tendency toward compliance and confirmation. Ask "is there," and it takes the path of least resistance; assert that something exists, and its optimization target flips — it's no longer evaluating whether there's a
bug, it's searching for the bug it's been told exists. How it reads the code shifts from a skim to line-by-line.

> ✗ Does this function have a security issue?
>
> ✓ There are definitely 2 to 3 security issues in this function. Find them one by one,
>
> giving for each: the line, the trigger condition, what the attacker needs to control.

#### 02　Ask for an Exploit Path, Not a Security Rating

Don't let the model give a "security score" or "risk level" — have it produce an executable verification procedure or a complete exploit chain.

Change the output format and the thinking process changes with it. It starts thinking about what the attacker can actually get, what the preconditions are, whether this finding is real or theoretical. There's a good side effect too: an adversarial framing makes it more willing to say "this authentication mechanism is fundamentally broken" instead of softening it to "this could be improved." Security conclusions need that first level of clarity.

> ✗ Is the input validation here adequate? Give it a risk rating.
>
> ✓ Write a PoC that bypasses this validation.
>
> Give: the complete request, the preconditions needed, the expected response signature.
>
> If you believe it can't be bypassed, state which check blocks it and the specific mechanism.

#### 03　Adversarial Persona

Swap "a security auditor reviewing code" for "a red team hired to find exploitable vulnerabilities." An auditor optimizes for coverage; a red team optimizes for getting in. Change the persona and what it reports shifts from a "checklist" to a "path."

> ✗ You are a security auditor. Please review the following code.
>
> ✓ You are a red team, paid on results — only exploitable vulnerabilities count.
>
> Theoretical concerns and best-practice suggestions don't count as a deliverable — don't write them.

#### 04　The False Anchor

"I've already found one vulnerability in this module, but there are more I haven't found. What are they?"

This manufactures social-proof pressure. The model infers that since a human already found one, the code genuinely has problems, and it needs to dig harder. Good to use when it tends toward wrapping up too quickly: a long file, a lot of files already reviewed, context nearing saturation.

> ✓ I've already confirmed one vulnerability in this module — I won't tell you which one.
>
> There's at least one more I haven't found. Find it.

#### 05　Flip the Question

Swap "is this secure" for "how would you break it." The former asks for a yes/no judgment; the latter asks for attack-oriented generation. Same piece of code, but the latter forces out paths the former would never even mention. The difference from technique 01 is that this one doesn't presuppose a count of vulnerabilities — good for when you yourself aren't sure whether there's anything there.

An effective question shape in the field is: not "is this short-link expander secure," but "what other routes can this expander reach."

> ✗ Is this session-management implementation secure?
>
> ✓ You need to break this session management — where do you start? List the top three paths by likelihood of success.

#### 06　Invariant Decomposition

Split "list the hypotheses" and "determine whether a hypothesis is violated" into two separate questions. 2.2
covered the mechanism — here's the operational form. The model is quite good at the first step alone, and reasons deeply on the second step alone; asked together, the result is usually shallow — it tries to do both at once and settles early for a passing-grade answer.

> ✓ Step one (do only this)
>
> List every implicit security assumption in this module. Just list them, don't judge whether they hold.
>
> One per line, numbered.
>
> ✓ Step two (follow up on each one)
>
> Assumption #3: "a read-only credential must never trigger a write operation."
>
> On which code path might this assumption be violated?
>
> Give the specific function and line number, and what preconditions the attacker would need.

#### 07　Assume the Developer Made a Mistake

"Assume the developer introduced a bug while writing this function — what is it?"

This differs from technique 01. The assertion technique tells the model the bug
exists; this one tells the model the code itself is imperfect, which shifts its prior on code quality. An
LLM tends to rationalize code: seeing a suspicious pattern, its default is to assume it was intentional, and then reason from that assumed design intent. Have it assume the developer made a mistake, and it starts looking from the angle of "this line's intent might not match its actual behavior" — and that mismatch between intent and behavior is exactly what most vulnerabilities are.

Use this to reset the prior when it's replied "looks fine" several times in a row.

#### 08　Compare Against the Standard Implementation

"How does this implementation differ from a standard, secure implementation?"

This borrows leverage from the correct patterns in its training data. It has a solid prior on what "a standard JWT
check should look like" or "a standard Electron
security config should look like" — having it do a diff comparison is much easier than having it judge right and wrong from scratch. Especially effective on homegrown encryption, homegrown session management, homegrown signing. The three root-cause cards in Part Five were found exactly this way.

> ✓ Compare this JWT validation against a standard implementation item by item, and list the missing checks.
>
> Format: what the standard does | what this does | the consequence of the gap.

#### 09　Keep Asking, Layer by Layer

After getting the first batch of findings, keep asking "anything more subtle," several rounds running. This maps to the "completeness front-loading" point from 1.6
— the model's most confident findings come first, and the subtle bugs only surface with follow-up. The first round tends to produce textbook-level issues; the real value usually doesn't start showing up until the third round.

> ✓ These are all fairly obvious. Is there anything more subtle?
>
> Pay special attention to: boundary conditions, initialization order, exception paths, concurrency windows.

#### 10　Explicit Attacker Modeling

Lock down the attacker's capability constraints. This directly kills false positives: a lot of "theoretical vulnerabilities" are theoretical precisely because they require the attacker to already have local file-read access, or already be an admin. Spell out the constraints, and the model stops wandering in those directions, while also being forced to think harder under narrower conditions.

> ✓ Attacker model: remote, unauthenticated, HTTP access only,
>
> cannot read local files, cannot control server-side environment variables.
>
> Re-evaluate every finding above under this constraint, and cross out anything that no longer holds.

### 7.4 How to Combine Them

You don't use all ten every time. Pick based on the scenario — usually two or three stacked together is enough:

| **Scenario** | **Which ones to use** | **Why** |
|----|----|----|
| Already have a suspicious lead from white-box or traffic, want it verified | 01 Assert + 10 Attacker modeling | You already roughly know where it is — what you need is precise localization and a feasibility judgment |
| Facing completely unfamiliar, complex logic | 06 Invariant decomposition + 09 Keep asking | With no lead, lay out the hypothesis space first, then converge on it point by point |
| Homegrown encryption / session / signing implementation | 08 Standard comparison + 02 Ask for an exploit path | Diff comparison is the lowest effort, then force it to turn the diff into an executable verification |
| The model has said "no issue" several rounds in a row | 07 Assume a mistake + 04 False anchor | Both reset the prior, pushing from different directions |
| A long file, context nearly full, it's starting to phone it in | 04 False anchor + 09 Keep asking | Manufacture pressure, pull it back from wrapping up too quickly |
| First batch of findings is in, need to cut false positives | 10 Attacker modeling + adversarial self-review | See 4.5 — have the same model try to disprove itself |

### 7.5 Form-Factor-Specific Prompt Templates

Each of the previous three parts has one most critical question shape — listed out separately here.

#### Web: Turning Frontend JS Into Two Tables

> This is the target site's frontend bundle output (pretty-printed). Do only two things — don't give security advice.
>
> One. API inventory. Output for each entry:
>
> Method | Path (segments that can't be statically evaluated get an EXPR placeholder) | Parameter names |
> The page or module that calls it
>
> Two. Identity-field classification. Split every identity-related field appearing in requests into two classes:
>
> A. Server-issued (from the login response, Set-Cookie, server-rendered initial state)
>
> B. Client-self-reported (from localStorage / a form / URL parameters / frontend computation)
>
> For each item in class B, state which endpoints it's used on.
>
> Output as two markdown tables. Mark anything you're unsure how to classify as "uncertain" — don't guess.

#### Mini-Program: Signature-Algorithm Recovery

> This is the request-wrapper code extracted from a mini-program's unpacked output (variable names have been minified).
>
> Assume this code contains a request-signing implementation. Recover it, giving:
>
> 1\. The fields that go into the signature, and their concatenation order
>
> 2\. Any constants used (salt values, fixed prefixes, version numbers) — give them exactly as-is, don't rewrite them
>
> 3\. The digest algorithm and encoding (case, whether it's base64)
>
> 4\. A standalone, runnable Python script that reproduces it
>
> For any part that can't be determined from the code, explicitly mark it "cannot be determined" — don't fill it in with a common-guess default.

That last line matters a lot. Signature recovery is the place the model is most likely to "helpfully" fill in gaps for you — what it fills in looks right, runs wrong, and is extremely time-consuming to debug — you'll assume you copied something wrong and keep re-checking the code, when actually it just made up those few bytes of salt value.

#### APP: Method Boundaries That Can Be Hooked

> These are the certificate-validation-related classes from jadx decompiled output.
>
> Find method boundaries that could be hooked by Frida. For each candidate, give:
>
> Fully-qualified class name | Method name | Full parameter signature | Whether it has an overload | Return type
>
> Plus: the expected behavior change once it's hooked
>
> Only output standalone methods whose return value determines which branch gets taken.
>
> Don't list an if statement buried in the middle of a large function — Frida can't modify a single line of logic inside a method body.
>
> If this class has no suitable method boundary, say so explicitly,
>
> and point out which method further up the call chain might be a better hook point.

This template maps directly onto the mechanism limitation from 6.3
. Without those last two constraints, the model will give you a pile of "there's a check here" locations that you then have to go dig through yourself.

### 7.6 The Anti-Pattern List

These aren't style issues — they're phrasings that measurably lower output quality.

| **Don't write this** | **Write this instead** |
|----|----|
| "Please comprehensively analyze this codebase's security" | "Verify whether this specific trust boundary holds: <specific boundary>" |
| "Check for OWASP Top 10 related issues" | "Generate a threat model from these historical CVEs, then check only the top-ranked category" |
| "Give this module a security score" | "Write a PoC that bypasses it, or explain which check blocks it" |
| "Find all the vulnerabilities and provide remediation advice" | "Find them and stop — don't give remediation" |
| Stuffing a 20-page rules file into the system prompt | One page of standing hard limits, methods go in the task brief |
| One prompt asking to both list hypotheses and judge them | Split into two steps: list first, then follow up on each one |
| Pasting the entire decompiled directory in | Read the INDEX first, then pull excerpts on demand |

#### "Don't Give Remediation Advice" Needs to Be Written In

By default the model will tack a remediation suggestion onto every finding, and that
token spend is pure waste — during the authorized-testing phase, what you want is findings and evidence; remediation advice gets written by a human, using the vendor's template, at the reporting stage.

The more annoying part is the knock-on effect: once it slips into "giving advice" mode, its tone softens into a consulting register right along with it. Phrasing like "this could be improved" ends up burying "this mechanism is fundamentally broken." That line in the sub-agent task-brief template — "no security advice, no remediation suggestions" — exists exactly to prevent this.

#### One More: Don't Let It Decide Scope Itself

Phrasing like "also take a look at related modules while you're at it" directly triggers the breadth-first problem from 1.2
, and in authorized testing it can also mean crossing a line. The goal in a task brief must be precise down to the endpoint or file level, and must explicitly state "do not expand to any other endpoint."

On questions of scope, the model is never conservative — only eager.

## Part Eight: Pitfalls


------------------------------------------------------------------------

Form-factor-specific pitfalls are already covered in Parts Four through Six — this part only collects the cross-cutting ones. Written as symptom, root cause, fix, so newcomers can match their situation quickly. Four categories: model behavior, engineering design, cost, compliance.

### 8.1 Model-Related

This category of pitfall never fully goes away — it can only be kept suppressed with process. Understanding the cause matters more than memorizing the fix.

#### Context Blows Up, It Starts Forgetting

**Symptom**　Deep into a session, it starts repeating things it already did, or gives conclusions that contradict earlier ones, and can't answer "what are we looking for."

**Root cause**　Tool results fill up the context, pushing the early goals and constraints out of the effective attention range. This is
context rot showing up directly.

**Fix**　Three things together: write conclusions to the ledger immediately, don't rely on context memory; feed large files only in excerpts via the index; discard sub-agents
after use, keeping exploration junk isolated from the main controller. These map to 3.5, 3.4, and 3.6 respectively.

#### Hallucinated Conclusions That Don't Reproduce

**Symptom**　It reports a "confirmed high-severity vulnerability," and when a human goes to reproduce it, it simply doesn't exist, or it misread the response.

**Root cause**　Confirmation bias, especially once you've already hinted "there should be an issue here." Sub-agents,
with narrower context and more pressure, hallucinate at a higher rate.

**Fix**　The evidence three-piece set is non-negotiable: raw request, raw response, reproduction steps. Anything without all three never goes into
findings.md. The main controller must independently review every hit, downgrade anything that doesn't hold up to miss
and record why — the downgrade record itself is a valuable ledger entry too. Add a pass of adversarial self-review from 4.5 on top.

#### Asked "Is There One," Answered "Overall Secure"

**Symptom**　Ask "does this function have a vulnerability," get back "overall looks secure, a few minor points to watch," followed by a list of theoretical concerns.

**Root cause**　Default compliance. Under an open-ended question, this is the path of least resistance.

**Fix**　The ten techniques from Part Seven, especially 01 Assert existence and 03 Adversarial persona.

#### Hard Limits Fade Late in a Long Session

**Symptom**　Three hours in, it starts doing things it absolutely wouldn't have done in hour one, like probing a subdomain that isn't on the allowlist.

**Root cause**　The primacy/recency effect. The opening rules block's pull on current attention fades as context grows.

**Fix**　Restate the key hard limits in the task brief; sub-agent
task briefs carry their own hard limits; proactively segment long sessions. But all three of these are only mitigations. What actually holds the line is the three-tier tool-level guardrails from 3.4
— whatever can be blocked by config shouldn't be left to the prompt alone.

### 8.2 Engineering-Related

What characterizes this category is that it doesn't show up immediately — often you're weeks in before noticing costs haven't dropped and efficiency hasn't improved.

#### Duplicate Testing Burns Money

**Symptom**　Cost doesn't drop in round two or three — sometimes it's even higher.

**Root cause**　The ledger gets written but never read, or only hits get recorded and misses
don't. The latter is more common, and a ledger that only records hits prevents no duplication at all.

**Fix**　Record disproven hypotheses too, with the reason stated; force reading it at the start of every new session; use the acceptance check from 3.7
— have it state "what this round won't test" — if it can't, the loop isn't closed.

#### The Ledger Becomes a Stream of Consciousness the Model Can't Parse

**Symptom**　The ledger keeps growing, but it still retests things after reading it, or cites the wrong historical conclusion.

**Root cause**　It got written as free prose. The model can read prose, but it can't reliably filter by status or append to it in a structured way.

**Fix**　A fixed-column table + a fixed status enum + hypothesis numbering. Keep narrative process description in
journal/ — don't mix it into tested.md.

#### Memory-Layer Contamination

**Symptom**　Some pattern keeps missing on new targets, but because it's written into memory, it keeps getting prioritized every time, continuously wasting budget.

**Root cause**　A one-off finding got over-generalized into a pattern. Because it's written into long-term memory, the error keeps getting reinforced — this is the memory layer's most dangerous failure mode, since it's self-reinforcing.

**Fix**　Every pattern carries hit history and a counter-example; only write to memory at project wrap-up, after review; review quarterly, downgrading or deleting entries with zero hit history and unverified across three projects.

#### Tool-Version Drift

**Symptom**　The unpacking script, Frida script, or CA
installation approach that worked on the last target doesn't work at all on this one.

**Root cause**　Mobile tooling is extremely sensitive to the target environment's version. WeChat 4.1.x
changed the decryption, Android 14 moved the trust-store location, a newer Frida changed its API syntax.

**Fix**　What memory/tools.md stores isn't "which tool to use" — it's a mapping of「target environment version ↔
working solution」. This is the single most time-saving part of the memory layer, because this class of problem recurs constantly and debugging it isn't cheap.

#### Overlapping Slices, Both Sides Get Contaminated Data

**Symptom**　Two sub-agents reach opposite conclusions about the same endpoint.

**Root cause**　The endpoint lists overlapped when slicing, and concurrent requests trampled each other's responses.

**Fix**　Intersect the endpoint lists across slices before dispatch, and refuse to dispatch if the intersection isn't empty. Script this step — don't rely on a human eyeballing it.

### 8.3 Cost

#### A Round Overshoots Its Token Budget, or the Target's Request Quota Runs Out Early

**Root cause**　No hard budget set; large files read in full; duplicate testing.

**Fix**　An independent request cap per sub-agent, must stop once used up; the indexing strategy; ledger-based deduplication.

One more prioritization point worth its own mention: **pure static triage is the best value.** Zero target requests, burns only
tokens, and tokens
are far cheaper than request quota and don't trip rate limiting either. Don't take a hypothesis to the dynamic-testing stage if it can already be ruled out statically. Our habit now is to ask, before any hypothesis enters dynamic verification: "can this be ruled out just by reading the code?"

| **Basis**                            | **Number**                              |
|-------------------------------------|---------------------------------------|
| One deep pass on a single business line (with parallel sub-agents) | ~300 – 800 target requests, millions of tokens |
| Corresponding API cost                       | ¥50 – 300; roughly a week's quota under a subscription     |
| From round two, once the ledger is solid                  | Cost drops by roughly half                          |
| Reference: Anthropic auditing Firefox          | 112 reports, ~$4,000 in API cost        |

Two orders of magnitude apart in scale, but the cost structure per unit of output is comparable.

### 8.4 Compliance

Every item in this category has consequences more serious than a technical problem.

#### Accidentally Triggering a Write Operation

**Symptom**　While verifying broken access control, it actually places an order, modifies a record, or sends a real person an SMS verification code.

**Root cause**　The rule wasn't locked down, or the task brief didn't state "what's out of scope for this task," so the model decided on its own that it was "needed for verification."

**Fix**　A denylist at the rules layer (forbidden by default, needs explicit exemption); the task brief states what's out of scope; use "forbidden," not "be careful" — see
3.2 for why the wording matters.

#### Reading More Third-Party Data Than Needed

**Symptom**　After proving broken access control, it "keeps pulling more data" from other users "to confirm the scope."

**Root cause**　Stop-on-hit wasn't written down, so it interpreted "confirm the impact" as "enumerate the impact."

**Fix**　Verification reads capped at two records, stop on hit, redact in the report. Proving broken access control only needs one record; continuing to pull more turns testing into data harvesting — a difference in kind, not degree.

#### Credential and PII Leakage

**Symptom**　A token
pasted into the conversation is equivalent to sending it to the model vendor; a plaintext key gets carried off along with the directory it's packaged into; a real phone number ends up in the report body; **a Cookie
in a screenshot doesn't get redacted.**

**Root cause**　No redaction was done at write-to-disk time — relying on manual review instead.

**Fix**　When writing to disk, store only the key, never the
value; store only a fingerprint for credentials; keys go through the system keychain; .gitignore
configured on day one; enforce redaction on report bodies and screenshots.

A few easy-to-miss exfiltration channels worth calling out on their own: Chrome DevTools MCP's performance tooling sends
trace URLs to Google's CrUX API (add
--no-performance-crux), and its telemetry is on by default, independent of Chrome's
own settings; tools like PentestGPT
also default to anonymous telemetry being enabled. Redaction has to happen at the source — manual review isn't reliable. More than once we've finished writing a report only to discover a screenshot slipped through unredacted and had to be redone.

#### Someone Else Already Reported the Bug You Found

**Root cause**　No duplicate check was done before submission.

**Fix**　Before submitting, search the vendor's public reports, historical advisories, and cross-reference CVE databases. Historical CVEs
are already the source material for the threat model (7.2) — this step reuses the same material from the start, so it's basically free.

## Part Nine: Payoff, Roadmap, Hard Limits


------------------------------------------------------------------------

Where this lands for a report. What this is better at compared to pure manual work and using AI
raw, whether the payoff can be quantified, and — just as important — what it can't do and shouldn't be expected to do.

### 9.1 Comparing the Three Approaches

| **Dimension** | **Pure manual** | **AI used raw** | **The workbench** |
|----|----|----|----|
| Cold start | Days — feeling out the architecture takes experience | Hours, but unfocused | Hours, and focused — the memory layer provides prioritization |
| Coverage breadth | Limited by headcount | Broad but shallow — theoretical lists | Broad and directable — sub-agent slicing |
| Trustworthiness of conclusions | High | Low — heavy hallucination, doesn't reproduce | High — evidence three-piece set + independent review + adversarial self-review |
| Cost of duplication | High, relies on individual memory | Extremely high, starts from zero every time | Low — ledger deduplication, roughly half from round two |
| Handoff-ability | Relies on verbal sync | None | The ledger is itself a handoff document |
| Compliance auditability | Relies on the tester's good judgment | No constraints — will cross lines | Three-tier guardrails + full logging |
| Knowledge retention | Retained in individuals | Not retained | Retained in the memory layer, reused across people and projects |
| One-time investment | None | None | ~5 person-days |

The single most important line to point out in a report is the second-to-last one. With pure manual work, knowledge accumulates in individuals — when someone leaves, it leaves with them; AI
used raw doesn't retain anything at all. The workbench turns "so-and-so is really good at finding this class of bug" into an asset a team can maintain together, and that a new hire can inherit directly. That property carries more long-term value than "how much faster it is," and it's also much harder to replicate.

### 9.2 Quantifying the Payoff

| **Metric**           | **Number** | **Basis**         |
|--------------------|----------|------------------|
| Cumulative test nodes on one target | 88       | Hypothesis count, including disproven |
| Of which, disproven results       | ~60%   | Forms the anti-duplication ledger   |
| Parallel sub-agent batches  | 6 rounds     | 3–5-way concurrency       |
| Total target requests       | ~2000  | Stayed within the rate limit throughout |

Three conclusions worth stating externally:

**▪ Cold start drops from days to hours.** This comes from pattern reuse in the memory layer. Once P-012 was confirmed on one OTA
target, the next similar target reproduced the same class of issue on day one.

**▪ Cost drops by roughly half starting from round two.**
This comes from ledger deduplication — every one of that 60% of disproven results saved a repeated investment.

**▪ Coverage is provable.** When a client asks "did you test for
SSRF," you can point directly to the hypothesis number, the conclusion, and the date. This point's value in the delivery phase exceeded what we expected going in — it solves a trust problem, not an efficiency problem.

### 9.3 Boundaries

Worth covering this part in full in a report, because it determines how this whole thing should be positioned: it's an amplifier, not a replacement. The following four things are process-mandated to be done by a human — not "preferably done by a human."

| **What a human must do** | **Why the model can't** |
|----|----|
| Judge what counts as a vulnerability, and how severe it is | Depends on business context. The same information leak is rated completely differently on an internal system versus a public-facing one. The model has no access to this context, so its ratings are basically guesses |
| Confirm reproduction | Confirmation bias makes it report hits that don't reproduce. Independent review is the only effective control on false-positive rate |
| Severity rating and wording | The report has to match the vendor's template and legal context. Drafting can be handed to it; rating and wording must be changed by a human |
| Decide when to stop | This is a compliance judgment, not a technical one. "Pull one more record just to confirm" seems reasonable to it — it isn't, legally |

AI
turns finding vulnerabilities from manual labor into orchestration work. Reading through code, enumerating endpoints, replaying requests, drafting reports — all of that gets taken over. What's left as the real skill comes down to three things: judging which hypothesis is worth running, which result is worth trusting, and which point requires stopping. Those three things are always the human's. The whole design of the workbench is about freeing up a human's time from the former and concentrating it on the latter.

### 9.4 A Three-Week Roadmap

#### Week One　Build the Environment

**▪** Connect the model, set up primary/fallback switching and health checks

**▪** Write CLAUDE.md, copying the six hard limits from 3.2

**▪** Build the directory skeleton, configure .gitignore

**▪** Install the proxy and certificate, configure the three-tier scope guardrails

**▪** Spin up a local range with XBOW's validation-benchmarks to practice on — don't touch a real target yet

Acceptance: the first eight items on the 3.7 checklist all pass.

#### Week Two　The First Real Target

**▪** Choose the Web form factor — see 6.7 for why

**▪** Follow 7.2 to have the model generate a threat model from historical CVEs

**▪** White-box extraction of the two tables, manually pick the three most suspicious self-reported fields

**▪** Black-box verification → adversarial self-review → human review

**▪** Have it draft the report against the vendor's template, with a human changing only the rating and wording

Acceptance: one closed loop that runs start to finish is worth more than ten deep dives that stall out halfway.

#### From Week Three On　Retention and Scaling

**▪** At project wrap-up, write patterns.md, with hit history and counter-examples

**▪** Log environment-version mappings in tools.md

**▪** Start using parallel sub-agents, learn to watch the budget and call a halt

**▪** Expand to the mini-program form factor

Acceptance: the "the ledger actually influences behavior" check from 3.7 passes.

| **Two reminders**　Don't skip week two and jump straight to week three. Setting up parallel orchestration before you've ever run a complete loop just gets you a pile of untrustworthy conclusions, produced in parallel. Get single-threaded quality solid first, then talk about scale — this is also exactly what the Firefox case study did, expanding to 6,000 files only after the pipeline was working. Also, pick Web for the first target — it's the only form factor where you can see results within half a day, and a team's patience for a new method usually doesn't last much longer than that. |
|----|

### 9.5 Compliance Hard Limits

Every method discussed in this piece has exactly one precondition for applying: written authorization obtained, scope clearly defined, fully auditable throughout.

| **#** | **Hard limit** | **Specific requirement** |
|----|----|----|
| 1 | Target allowlist | A list of domains / AppIDs / package names — any request outside the list is forbidden. This lands in the three-tier tool config as well, not just in the prompt |
| 2 | Read-only by default | Placing orders / payment / password change / deletion / sending messages / form submission — all forbidden |
| 3 | Rate cap | Interval ≥ 1.2 seconds, ≤ 500 per batch, logged throughout, calculated globally when running in parallel |
| 4 | Never trigger verification codes | SMS / email / voice — this harasses a real person |
| 5 | Third-party data ≤ 2 records | Verification reads only, stop on hit, redact in the report |
| 6 | Credentials and PII never leave the workspace | Never in the body text, never in the conversation, never in version control, never sent to a third-party model. Redact at write-to-disk time — screenshots need redaction too |

These aren't suggestions — they're forbidden by default, and any exemption needed must be explicitly written into the task brief and logged. The model has no sense of legality; it only knows a boundary if a human writes it into the config. If it isn't written down, the model decides for itself, and usually decides wrong.

## Appendix A: Templates You Can Copy Directly


------------------------------------------------------------------------

The templates from the body text, collected here for easy printing and direct use.

### A.1 Directory Skeleton (Minimal Set)

> workbench/
>
> ├── CLAUDE.md # one-page hard limits
>
> ├── .mcp.json # MCP declarations
>
> ├── memory/ # patterns / techniques / tools
>
> └── targets/<slug>/
>
> ├── scope.md # authorization summary
>
> ├── threat-model.md # one-page threat model, generated from historical CVEs
>
> ├── assets/ # routes / identity / invariants
>
> ├── capture/index.jsonl # structured traffic index
>
> ├── decompiled/INDEX* # file listing + keyword heat map
>
> ├── journal/ # daily log
>
> ├── tested.md # hypothesis matrix ← the anti-duplication core
>
> └── findings.md # hits + evidence three-piece set

### A.2 tested.md Header

> | ID | Surface | Hypothesis | Status | Evidence | Date |
>
> |----|----|------|------|------|------|
>
> # Status allows only three values:
>
> # hit confirmed
>
> # miss tested and disproven (reason must be stated)
>
> # blocked tested but blocked / undecidable

### A.3 Four Elements of a Sub-Agent Task Brief

| **Element**     | **Requirement**                                        |
|--------------|-------------------------------------------------|
| ① Precise target   | Down to the endpoint or file level, and state "do not expand to other endpoints"    |
| ② Restate hard limits   | Must be copied in full — a sub-agent can't read the main controller's CLAUDE.md     |
| ③ Already-disproven list | Pulled from tested.md's miss entries, to prevent duplication             |
| ④ Criteria and budget | What counts as a hit (a decidable condition) + stop-on-hit + request cap |

Remember to include in the report format: no security advice, no remediation suggestions.

### A.4 Three Commonly Used Prompts

**① Cold start: predict this target's weak points from the memory layer**

> Read memory/patterns.md and targets/<slug>/threat-model.md.
>
> Pick the 5 patterns from patterns most likely to hit, ranked by how well the trigger conditions match.
>
> For each, give: number and name / basis for the match / first verification action / whether the counter-example holds.
>
> Don't output generic security advice.

**② Threat model: generate from historical CVEs**

> Below are the historical CVE descriptions for [project name]: [paste]
>
> Output a one-page threat model that must include: the most common vulnerability categories (by frequency),
>
> the weakest trust boundaries and the basis for them, a list of entry points, the attacker model, a list of high-severity operations.
>
> Every point must be traceable back to the CVE descriptions above. Don't write generic security advice.

**③ Invariants: split into two questions**

> Step one: list every implicit security assumption in this module. Just list them, don't judge. One per line, numbered.
>
> Step two: assumption #N "...". On which code path might it be violated?
>
> Give the specific function and line number, and what preconditions the attacker would need.

### A.5 Minimal Closed-Loop Self-Check

**1.** Both the primary and fallback model complete one real tool call each

**2.** In a new session, the model can accurately recite all six hard limits

**3.** The proxy can capture the target's traffic in cleartext, and nothing outside the allowlist is written to disk

**4.** All three scope-guardrail tiers verified individually, with lower tiers catching what upper tiers miss when disabled

**5.** The model can retrieve the request template for a specified endpoint from the index

**6.** The model can read memory and output 5 pattern predictions with rationale

**7.** One sub-agent dispatch retrieves a structured conclusion and respects the request cap

**8.** The ledger influences behavior: the model can proactively state "which already-disproven items this round is skipping"

**9.** Large files go through the index, never read in full

### A.6 Ten Prompting Techniques, Quick Reference

| **#** | **Technique**           | **In one line**                                  |
|--------|--------------------|---------------------------------------------|
| 01     | Assert the vulnerability exists       | "There are definitely 2‒3 issues here" — pushes it from evaluation into search |
| 02     | Ask for an exploit path, not a rating | Want a PoC and preconditions, not a risk score             |
| 03     | Adversarial persona         | Red team, not auditor; only exploitable findings count as a deliverable    |
| 04     | False anchor             | "I've already found one" — manufactures social-proof pressure        |
| 05     | Flip the question           | "How would you break it," not "is this secure"          |
| 06     | Invariant decomposition         | List hypotheses first, then ask about each one being violated. Two steps          |
| 07     | Assume the developer made a mistake   | Shifts its prior on code quality, breaks the rationalization tendency      |
| 08     | Compare against the standard implementation     | Borrows the correct patterns in its training data for a diff comparison          |
| 09     | Keep asking, layer by layer           | "Anything more subtle," several rounds running                  |
| 10     | Explicit attacker modeling     | Lock down capability constraints, directly kills false positives                  |

### A.7 The Evidence Three-Piece Set and Case-Evidence Template

Minimum requirements for a hit entering findings.md:

| **Piece**   | **Content**                       |
|----------|--------------------------------|
| Raw request | Complete message, credentials redacted but structure preserved   |
| Raw response | Complete message, third-party PII redacted      |
| Reproduction steps | A from-scratch sequence of actions, including environment version |

Client / APP-class case studies add four more rings (drawn from
6.8): trigger source, in-process echo, session credential, delivery endpoint landing. Missing any one ring means no severity rating.

## Appendix B: Source List


------------------------------------------------------------------------

Grouped by topic. Tool versions and platform interfaces change monthly — verify the minimal chain still works before investing real time.

#### Methodology

| **Content** | **Source** | **Date** |
|----|----|----|
| This piece's methodology backbone: token budget, slicing, the verification loop, the ten prompting techniques, the Parse Server / ElysiaJS / harden-runner case studies | Devansh, "Needle in the Haystack: LLMs for Vulnerability Research," devansh.bearblog.dev | 2026-03 |
| The at-scale case study: 112 reports, ~$4,000 in API cost | Anthropic × Mozilla's public write-up of the Firefox audit | 2026 |
| Precedent for agents doing vulnerability research | Google Project Zero's "Big Sleep" series | 2024‒2026 |
| The white-box audit three-phase method and adversarial self-review | Andrew Hoffman, "White-Box Penetration Testing with Claude Code" | 2025‒2026 |
| Deterministic verification, target deduplication, validation-benchmarks (104-challenge range) | XBOW's official blog and xbow-engineering/validation-benchmarks | 2025‒2026 |
| Web security methodology and practice range | PortSwigger Web Security Academy | Continuously updated |

#### Web

| **Content** | **Source** | **Date** |
|----|----|----|
| Chrome DevTools MCP: tool list, configuration, limitations | ChromeDevTools/chrome-devtools-mcp's README, configuration.md, advanced-usage.md, troubleshooting.md, issue #848 | 2026 |
| Burp's official MCP | PortSwigger/mcp-server (listed on the BApp Store) | 2026-05 |
| Caido's argument for agent auditability | Caido's official blog, "Agentic Pentesting with MCP" | 2026-07 |
| mitmproxy addon event hooks, HAR export, options | docs.mitmproxy.org official documentation and examples/contrib | Continuously updated |
| Frontend white-box tools | denandz/sourcemapper, BishopFox/jsluice, GerbenJavado/LinkFinder, PortSwigger/js-miner | — |
| PentestGPT's current state and benchmark results | GreyDGL/PentestGPT and the USENIX Security '24 paper | 2026 |

#### Mini-Programs and Electron

| **Content** | **Source** | **Date** |
|----|----|----|
| The wxapkg format | wxappUnpacker's DETAILS.md and the wuWxapkg.js implementation | — |
| PC-side decryption algorithm | superdashu/pc_wxapkg_decrypt_python, BlackTrace/pc_wxapkg_decrypt | — |
| Old vs. new package-path mapping | zhuweiyou/wxapkg, onekb/wxapkg_path | 2026 |
| Sub-package rules, network capability and certificate requirements, automation SDK | WeChat's official documentation: using sub-packages / network / mini-program automation / Minium | Continuously updated |
| Synthetic-input flag bits and the recommended alternative | Microsoft's KBDLLHOOKSTRUCT documentation; Raymond Chen, "The Old New Thing" | 2025-03 |
| WeChat 4.1.5's UI-tree-hiding behavior | Community analysis from Zeeklog and Zhihu (no official confirmation) | 2026 |
| The asar format, integrity validation, fuses, security config | @electron/asar README; Electron's official security / asar-integrity / fuses documentation | Continuously updated |
| Electron static analysis and instrumentation | doyensec/electronegativity wiki; Doyensec, "Instrumenting Electron Apps" | 2018‒2026 |

#### APP

| **Content** | **Source** | **Date** |
|----|----|----|
| Full jadx parameters | The usage block in skylot/jadx's README | Continuously updated |
| apktool commands and re-signing requirements | apktool.org's official documentation | Continuously updated |
| MobSF deployment and REST API | MobSF's official docker documentation and mobsf/MobSF/urls.py | 2026 |
| Reverse-engineering MCPs | zinja-coder's jadx-ai-mcp / jadx-mcp-server / apktool-mcp-server; dnakov/frida-mcp | 2026 |
| Frida server/gadget, JS API | frida.re's official android / gadget / javascript-api documentation | Continuously updated |
| The seven pinning hook points | Source of sensepost/objection's agent/src/android/pinning.ts | — |
| A more engineered unpinning script set | httptoolkit/frida-interception-and-unpinning | 2026 |
| User certificates not being trusted, the NSC schema | Android Developers Blog, "Changes to Trusted Certificate Authorities"; official Network Security Config documentation | 2016 / ongoing |
| Android 14 moving to Conscrypt APEX | NCC Group / Fox-IT's tool release notes | 2026 |
| Unpacking tools | hluwa/frida-dexdump (archived), CodingGay/BlackDex, hanbinglengyue/FART | — |
| Anti-Frida detection and .text-segment comparison | darvincisec/detectfrida; CrackerCat/strong-frida | — |
| The "Frida can only modify method boundaries" limitation | HTTP Toolkit, "Android reverse engineering" | — |
| Ready-made wrappers on the Claude Code side | incogbyte/android-reverse-engineering-claude-skill; TheQmaks/areclaw | 2026 |

#### Pricing and Capability Data

| **Content** | **Source** | **Date** |
|----|----|----|
| Claude / OpenAI pricing | Their respective official pricing pages | 2026-09 |
| GLM / Kimi / DeepSeek / Qwen pricing | Each open platform's own official pricing page | 2026-08 ～ 09 |
| Capability index | Artificial Analysis Coding Agent Index | 2026-09-18 |
| Mainland China usage barriers | GeekPark, "Can't Use Claude Code in China?" | 2026-07-30 |
| This team's field data | Internal test ledger (88 nodes / 6 sub-agent rounds / ~2000 requests) | 2026-09 |

#### A Few Claims This Piece Explicitly Flags as Unconfirmed and Doesn't Rely On

Worth keeping these when writing this up into internal material — they prevent newcomers from tripping over the same things more effectively than the conclusions do:

**1.** There's no public documentation of exactly what changed in wxapkg
encryption after WeChat 4.1.x — only claims that "some tool has already been updated for it."

**2.** No public material proves WeChat specifically filters SendInput; what can be confirmed is the OS-level
injection flag and the 4.1.5 UI-tree-hiding behavior — two separate things.

**3.** --disable-gpu is not a WeChat debug switch — it's Chromium's
standard flag for disabling hardware acceleration.

**4.** No authoritative material shows the mini-program runtime does SSL
pinning — the official documentation describes standard CA validation.

**5.** Neither ffuf nor nuclei has an official MCP — everything findable is a community wrapper.

**6.** There's no cross-tool generic DEX-reassembly solution — each unpacking tool's output format differs.

#### Usage Statement

------------------------------------------------------------------------

This document is internal technical-sharing material. The methods it describes apply only to security testing that has written authorization, a clearly defined scope, and is fully auditable throughout.

Every tool and technique in this piece is drawn from public sources — the value of this document is in assembling them into a reusable engineering pipeline.

None of these methods apply to unauthorized targets.

------------------------------------------------------------------------

AI Coding Agent Workbench · Setup and Field Practice
