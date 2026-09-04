---
name: change-brief
description: |-
  High-level design brief for a pull request, branch, or commit range, written so the author can read it unaccompanied. Establishes what it delivers versus what it claims, maps it against the systems it touches, and interrogates the design: whether each new durable thing (table, job, endpoint) needs to exist, whether an existing mechanism already covers it, whether what accumulates is quantified, and whether a narrow abstraction was built where a general one is implied. Ends with an options matrix and the questions that decide it.
  TRIGGER when: user says "brief me on this PR/change", "understand this at a high level", "no time to go line by line", "align with the author before reviewing", "does this deviate from our patterns", or "what's the blast radius".
  SKIP when: the user wants line-level defect hunting, severity-ranked findings, or an approve/reject verdict; the change is mechanical (version bump, typo, revert); or the design is settled and one specific question is asked.
argument-hint: "<PR-URL-or-number-or-range> [audience-note] [dimensions-to-emphasize]"
model: opus
---

# Change Brief

Explain what a change set commits the system to, and surface the design decisions worth settling **before** anyone audits implementation soundness.

Two things define the output:

- **The reader is fluent in the codebase.** They can read code; they don't have time to. Skip background, skip explaining the domain, skip narrating the diff.
- **The author will read this without you present.** It is asynchronous written feedback, not notes for a meeting. Every claim carries its own evidence, because there is no conversation in which to fill gaps.

**Not in scope:** line-level defect hunting, severity buckets, style, or an approve/reject verdict. If you find a real bug, state it once where it affects a design decision and move on.

## Operating principles

Apply these throughout, not as a final check.

**Question the design, not just the conformance.** The most common failure of this kind of review is to accept the author's design as the frame and only look for deviations *inside* it. Doing that produces a brief that is accurate and still misses the biggest point. For every change set, ask separately: *is this the right thing to build*, and *is it built consistently*. Step 4 exists entirely for the first question and must not be skipped or compressed.

**Evidence or silence.** Never assert a file, type, endpoint, config key, sibling implementation, consumer, or constraint you have not read. "Not verified" is a useful sentence; a plausible-sounding invention destroys the artifact's credibility with the one person who *can* check it. Mark inference as inference.

**Quantify anything that accumulates or repeats.** Do not describe a growth or cost concern qualitatively when the numbers are derivable from the code. See Step 4d.

**Find the binding constraint.** When something looks expensive, identify what actually breaks first — throughput limits, quota, index size, connection pool, cost, lock contention — rather than naming the most obvious resource. A concern attached to the wrong constraint is easy for an author to dismiss correctly.

**Separate the symptom from the shape.** A reviewer's stated worry is often a real signal pointing at a different underlying problem. Report both: confirm what is true, correct what is not, and name the deeper cause when the surface complaint is a proxy for it.

**Attribute to the code, not the person.** See Tone.

## Inputs

| Input | Default | Notes |
|---|---|---|
| **Change set** | required | PR URL, PR number, branch, or commit range. |
| **Audience** | engineer fluent in this codebase and domain | Users usually state this ("assume I know what X is"). Honor it — omit everything they claim to know. |
| **Emphasis** | all dimensions | If specific dimensions are named, lead with those; still sweep the rest, compressed. |
| **Recipient** | the change author | If the brief is for the reader's own use only, tone rules relax but structure does not. |

State your interpretation in one sentence, then start. Do not interview the user first.

## Getting the change set

Use whatever host tooling is available (`gh`, the host's CLI, a code-history tool, or plain `git` for a local range) to collect: metadata, full diff, file list, commit sequence, linked issues, and existing comments. Then **check out or fetch the head revision locally** — every step after this reads whole files, not diff hunks. A diff hides constructors, base classes, imports, registrations, and config, which is where most of this analysis lives.

Two things to extract that are easy to skip:

- **Linked issues and design docs** carry the intent the diff cannot. Where intent and diff disagree, that gap is a headline finding, not a footnote.
- **Existing comments** are read for *author rationale only* — "the author already explained why X" changes what is worth asking. You are not adjudicating other reviewers' findings.

## Step 1 — Intent versus delivered, decomposed

Write two statements and hold them side by side:

1. **Stated** — from title, body, linked issue, commits.
2. **Delivered** — from the diff alone, as if the description did not exist.

Then **decompose the delivery into separable changes**. A PR titled as one feature routinely ships three or four independently-shippable changes, and naming them separately is often the single most clarifying thing in the brief. For each, note its own blast radius:

| # | Change | Blast radius |
|---|---|---|
| 1 | {…} | {what it can affect} |

Flag any component that has **nothing to do with the stated purpose** — a global default, a shared fallback, a preference-resolution rule, a migration. Riders like these affect people who will never read this PR, and they are the most under-reported category in high-level review.

## Step 2 — Change map

Group changed files by layer or component (API surface, service, data, pipeline, UI, config, infra, tests): **component → what changed → net-new vs. modification**.

Extract two signals:

- **Center of gravity** — where the real design decision lives. Usually not the file with the most changed lines.
- **Surprising members** — a shared base class, DI module, serializer, or global config that has no obvious business being in this change set. These are leads for Step 3, not footnotes.

## Step 3 — Neighborhood sweep

**The change alone is not the story.** A brief built only from the diff is one the reader could have produced from the PR page. Run these as parallel explorations, each with enough standalone context to stand on its own; then fold the results back together.

1. **Sibling implementations** — find two or three existing places that solve the *same shape* of problem: another endpoint of the same family, another job of the same cadence, another integration, another surface of the same kind. Search by concept first, then by name. Read how they are structured. This thread feeds both Step 3's conformance verdicts and Step 4's alternatives, and it is the thread most often under-run.
2. **Framework and extension-point reach** — for every shared type, base class, interface, middleware, registry, generic helper, or schema the change *modifies* (not merely calls): who else derives from, implements, or consumes it, and does the change alter their behavior? A modified shared default is a change to every caller and appears nowhere in the diff.
3. **Downstream consumers** — for every contract touched (route, DTO, schema, event, column, message, flag), find the readers on the other side, including outside this service. Are they versioned, tolerant, or brittle?
4. **Prior art and history** — has this area been designed, changed, or reverted before? Commit history on the center-of-gravity files, plus nearby design docs, often reveals a constraint the change is silently re-litigating.
5. **Docs shipped inside the change set** — when a PR edits documentation describing its own behavior, compare those claims against the code in the same PR. Disagreement between the two is high-signal: it usually marks where the design changed mid-flight and something was not re-checked. It is also the cheapest way to confirm original intent.

Record each divergence with a verdict — *conforms* / *deliberate and better* / *deliberate and debatable* / *apparently unaware* — and a **named comparator path**. A divergence claim without a real comparator is an opinion, not a finding.

## Step 4 — Interrogate the design

This is the step that separates a useful brief from a well-organized summary. Steps 1-3 describe the change as designed; this one asks whether the design is right. Run all five sub-steps.

### 4a — Needs decomposition

Restate the change as the **user-facing needs** it serves, then map each need independently:

| Need | Existing mechanism that already covers it | What this change built | Real gap? |
|---|---|---|---|
| {…} | {path, or "none found — searched X, Y, Z"} | {…} | yes / no |

Typically only a subset of the needs actually require something new. Isolating which ones do is often the most actionable output of the entire brief, because it converts "I have concerns about this design" into "these two parts have an existing home; this third one is the genuinely new thing, so let's decide it on its own merits."

Do not fill "existing mechanism" with a guess. If you searched and found nothing, say what you searched.

### 4b — The why-new test

For **every new durable thing** the change introduces — table, job, scheduled activity, endpoint, service, queue, component, orchestration step, config surface, storage location — ask:

- What existing mechanism performs this **shape** of work today?
- Why was it not used? Is the reason stated, inferable, or absent?
- If the existing mechanism is *nearly* sufficient, what is the smallest extension that closes the gap — and would that extension benefit everything else that already uses it?

Extending a shared mechanism usually beats adding a parallel one, because the extension accrues to every existing user while the parallel path fragments the system. When you recommend it, state the cost honestly too — a framework extension has a wider blast radius than a local addition.

### 4c — The why-specific test

For every new schema, abstraction, table, or interface, ask whether it is **hard-coded to one instance of a concept that plainly has siblings** — one metric among many metrics, one platform among many platforms, one surface among many surfaces, one tenant type among many.

Then complete the thought in both directions, because both failure modes are real:

- **Cost of staying specific:** what does the second instance require — a parallel table, a copied job, a forked component?
- **Cost of generalizing now:** a generic store landing without the governance that makes it safe (registry, catalog, ownership, units, validation, lifecycle) can be worse than a specific one, because it becomes the path of least resistance for everyone who follows. Generalizing storage without generalizing the framework around it is the worst of both.

Present the trade, and say which side the evidence favors here.

### 4d — Quantify what accumulates

Anything written repeatedly per entity, per event, or per period gets arithmetic — not adjectives.

- **Rows or objects per period** = qualifying population × frequency. Derive the *qualifying* population from the code, not the total population. If the reader has estimated it from the total, correct the estimate and show why the filter narrows it — but do not stop there, because the volume is frequently not the real problem.
- **Projection** at 30 days and 1 year, and the growth curve (linear, per-entity linear, super-linear).
- **The binding constraint.** Look for the limit that is actually hit first, and prefer limits the codebase documents about itself over your own assumptions. Common ones the change may collide with: throughput or capacity ceilings on the target store, write pattern (row-by-row versus set-based), concurrency multipliers (a job that runs N times per day multiplies everything), quota on an upstream source, index or partition growth, and cost.
- **Lifecycle.** Is there a retention, cleanup, compaction, or archival path — anywhere in the system, not just in this change? If none exists, say so plainly; "no retention path exists" is a concrete, checkable statement.

Do the same for anything read repeatedly: calls per request, fan-out, N+1 shapes, and repeated scans.

### 4e — Complexity and shape

Count the moving parts the change adds (new components, cross-component ordering requirements, new failure modes, new places to look during an incident). Then ask whether the same user-visible outcome is reachable with fewer.

Pay attention to **cross-step read dependencies** — where one step writes what another step reads, especially across activity, job, or deploy boundaries. These are the most common source of ordering bugs, and they are usually a symptom of the design shape rather than an isolated defect: a shape that keeps the computation in one place cannot have the bug at all.

When a simpler alternative exists, state whether it is **larger or smaller than the current change**. "This alternative is strictly smaller than what is here now" is far more persuasive than an appeal to elegance, and it is often true.

## Step 5 — Dimensional sweep

Cover every dimension. Mark one **N/A** rather than padding it; a short honest table beats a long speculative one.

| Dimension | Ask |
|---|---|
| **Contracts & API surface** | New or changed routes, DTOs, schemas, events, signatures. Additive or breaking? Versioned? Who must change in lockstep? |
| **Data model & storage** | New tables, columns, indexes, objects. Migration, backfill, retention, cardinality, growth (per 4d). |
| **Data flow & lifecycle** | Where data enters, is transformed, persisted, read. Sync vs. async, push vs. pull, freshness and ordering guarantees. |
| **Framework & extension points** | Shared types, interfaces, middleware, registries, DI, config touched — and the consequences for *other* scenarios. |
| **Component boundaries & ownership** | Does responsibility move across a service or team boundary? New dependency edge or cycle? |
| **Permissions, auth & tenancy** | Who can call or see this? Is tenant scoping enforced on every new path? Any new data-exposure surface? State explicitly when there is no deviation. |
| **Config & feature flags** | New settings, defaults, per-environment behavior; what happens when the flag is off or config is absent. |
| **UX surface** | What a user actually sees or does differently. Empty, loading, partial, and error states. Does anything bypass the shared component model? |
| **Operability** | New failure modes, telemetry, how you would debug this during an incident, which alerts would not fire. |
| **Performance & cost** | Per 4d. |
| **Compatibility & rollout** | Back/forward compatibility, deploy ordering, rollback safety, in-flight data during rollout. |
| **Testing posture** | What kinds of tests came with it, and which of the above are untested. Report the shape; do not design the test plan. |

## Step 6 — Diagrams

Use Mermaid, at most three, only where structure is load-bearing. Highest value first:

1. **Component or data-flow, before → after** — usually worth it; distinguish new nodes and edges.
2. **Sequence diagram** — when a path crosses three or more participants, or when ordering between steps is itself the issue.
3. **ER or state diagram** — when a schema or a lifecycle is introduced.

Every node and edge must correspond to something you read. A diagram of an imagined architecture is the worst possible output of this skill.

## Step 7 — Options and the questions that decide them

Do not end on a list of questions alone. A reader who is not present to argue needs the alternatives laid out.

**Options matrix** — including the change as written:

| Option | What it looks like | Gains | Costs | Bigger or smaller than current diff |
|---|---|---|---|---|

Then a short **decision tree**: the two or three factual answers that select among the options — typically whether a capability is a committed requirement or a nice-to-have, whether an existing mechanism can be extended, and who owns the result.

**Open questions: three to seven, ranked** by *cost of being wrong × cost of changing later*. Each is a decision the author can confirm or correct in a sentence, and each carries its evidence:

- Effective: "The daily rows are written by the request path rather than by the existing scheduled job that already runs per entity — deliberate freshness trade, or should it move to the existing schedule?"
- Not effective: "Have you considered performance?"

Anything cheap to change after merge goes in a separate non-blocking list, not here.

## Step 8 — The artifact

Write **one self-contained markdown file** that stands alone when forwarded. Default name `PR-<id>-change-brief.md` (for a range, a slug of the range). Always write it, even when the conclusion is "conforming and low-risk" — the reader must be able to see what was examined.

**Number the sections** (§1, §2, …) so specific parts can be cited in follow-up discussion.

Render identifiers, paths, and issue numbers verbatim — no truncation. In chat, give the headline, the top few findings, and a pointer to the file; do not paste the artifact.

````markdown
# Change Brief: #{id} — {title}

**Repo:** {…} • **Branch:** `{head}` → `{base}` • **Author:** {…} • **Size:** +{a}/-{d} across {n} files
**Assumed known by the reader:** {…}

## §1 Summary
{3-5 sentences: what this delivers, the one design decision that matters most, and the single thing to settle first.}

## §2 What ships here
- **Stated:** {…}
- **Delivered:** {…}
- **Gap:** matches | more than stated | less than stated | different

| # | Separable change | Blast radius |
|---|---|---|

**Center of gravity:** {where the real decision lives}

## §3 How it works
{Prose plus diagram. Name the moving parts and how they connect; do not walk the code.}

## §4 Design questions
### §4.1 Needs versus existing mechanisms
| Need | Existing mechanism | What was built | Real gap? |
|---|---|---|---|
### §4.2 New durable things, and why each exists
### §4.3 Specific versus general
### §4.4 Volume, cost, and the binding constraint
{Arithmetic, projection, constraint, retention.}
### §4.5 Complexity and shape
{Moving parts, cross-step read dependencies, simpler alternative and its relative size.}

## §5 Pattern conformance
| Area | This change | Existing pattern (path) | Verdict |
|---|---|---|---|

## §6 Dimensional read
| Dimension | Finding | Worth settling? |
|---|---|---|

## §7 Blast radius
{What outside the diff is affected. State "none found; checked X, Y, Z" when that is the answer.}

## §8 Options
| Option | Shape | Gains | Costs | Size vs. current |
|---|---|---|---|---|

**What decides it:** {2-3 factual questions and what each answer implies.}

## §9 Questions for the author
1. **{Question}** — *why it matters:* {consequence} — *evidence:* {path or comparator}

## §10 Non-blocking
- {Cheap-to-change items, small suspected bugs, test gaps — one line each.}

## §11 What works as-is
{Plain statements of decisions worth keeping, so they survive any restructuring.}

## §12 Coverage
- **Read fully:** {…}
- **Skimmed:** {…}
- **Not examined:** {…, and why}
- **Uncertain or inferred:** {claims not fully verified, marked as such}
````

## Tone

The artifact is read by the person who wrote the code, usually without the reviewer present. Write it so it reads as analysis, not assessment.

- **Plain and factual.** Short sentences. State the fact, then the consequence.
- **Attribute to the code, not the author.** "The table has no retention path" — not "you forgot retention." Avoid "you did X," "unfortunately," "concerning," "this is wrong."
- **No blame and no absolution.** Skip both "this is a mistake" and "you couldn't have known this" — reassurance is as distracting as criticism, and both invite argument about framing instead of the decision.
- **No emotional framing or emphasis words.** No "surprisingly," "worryingly," "just," "simply," "obviously." If something matters, the consequence shows it.
- **Note what works as plain statement, not as cushioning.** §11 exists so the author knows which decisions to preserve through a restructure — not to soften §4.
- **Recommend without prescribing.** Give the options and what decides between them. The author owns the call.
- **Concede accurately.** Where the change is right and a concern of yours does not hold, say so directly. Selective accuracy is what makes the rest credible.

## Guardrails

- **Stay at the level of design.** If a paragraph could only be written by someone reading one method body, it usually belongs in an implementation review instead.
- **Do not invent.** No unverified paths, types, endpoints, config keys, siblings, or consumers. Say "not verified."
- **Cite comparators.** Every conformance and alternative claim names a real path.
- **Do not stop at description.** A brief that only explains the change, without Step 4, has not done the work.
- **Do not post anywhere** unless explicitly asked.
