---
name: hallucination-check
description: Checks whether a Claude answer or a specific claim is likely hallucinated by testing its checkable facts against provided data/sources and reporting a probability score. Use when asked to verify, fact-check, or double-check an answer for hallucination.
---

# Hallucination Check

Verifies whether a Claude response (or a specific claim) is factually grounded rather than hallucinated, and reports a probability score. Built to run cheap: it prefers checking against context that's already available over new searches or long re-generations, and it caps how much work it does per run.

## When to run

Invoke when the user asks to verify, fact-check, "double check," or check an answer for hallucination — their own, Claude's, or a pasted claim.

## Step 1 — Pick the target

- If the user names or pastes specific text/claims, verify only those.
- Otherwise, default to the most recent substantive Claude answer in this conversation.
- If the user supplied a verification question, dataset, file, or "ground truth" reference along with the request, treat that as authoritative context for Step 4 — skip straight to using it, don't re-ask for it.

## Step 2 — Offer context only if it's missing and would matter

- Skip this step entirely if the user already gave a question, dataset, or source to check against, or if the claims are easily self-checkable (pure logic, arithmetic, something answerable in one lookup).
- Otherwise, ask ONE short question (AskUserQuestion if available, else plain text): do they have a specific question, dataset, or source to check the answer against? Offer "no, just self-check" as a valid default so this never blocks progress. If the session is unattended or there's no reply, proceed straight to self-verification.

## Step 3 — Extract the claims worth checking

List the atomic, checkable factual assertions in the target: numbers, statistics, named entities (people, papers, genes, APIs, products), dates, citations, and causal/quantitative claims. Skip opinions, hedges, and generic explanation — those aren't checkable and aren't worth tokens.

Cap at the **8 highest-risk claims**, prioritizing specific/low-frequency facts (a citation, a computed number, a named entity) over generic statements, since those are where models actually hallucinate. If more exist, say so and offer to check further only if asked.

## Step 4 — Verify each claim, cheapest method first

For each claim, use the first method that applies — don't reach for an expensive one if a cheap one works:

1. **Direct check against supplied context/data/files.** Read or grep the actual source. No generation needed — cheapest and most reliable.
2. **Recompute.** If the claim states a number or result derived from code or data, re-run the actual computation and diff it against the claim, rather than trusting the prose summary of it.
3. **One targeted lookup.** For an external fact or citation, do exactly one focused search for that specific fact. Never re-research the whole topic to check one line.
4. **Self-consistency probe.** If nothing above applies, independently answer a narrow question that isolates just this claim — without re-reading the original phrasing first — then compare the two answers. Keep this to one short, targeted generation, not a full redo of the original work.

## Step 5 — Score

Per claim: ✅ Supported, ⚠️ Unverifiable (no way to check it), or ❌ Contradicted.

Aggregate into a hallucination probability:

`score = (contradicted × 1.0 + unverifiable × 0.4) / claims_checked`

Report it as a band, not just a raw number:
- **0–20% Low** — proceed as normal
- **20–50% Medium** — worth a second look at the flagged claims before relying on them
- **50–100% High** — treat the answer as unreliable until re-verified

## Step 6 — Report (short)

- Lead with the overall score and band.
- List only the flagged (⚠️/❌) claims, one line each: the claim, what it was checked against, and why it's flagged.
- State in one line how many claims were fully supported, without re-listing them individually.
- Do not reprint the original answer or the full verification reasoning — the point is a fast, cheap read, not a transcript.

## Token-budget rules (the whole point of this skill)

- Max 8 claims per run unless the user explicitly asks for an exhaustive pass.
- Prefer data/files the user already provided over new web searches; at most one search per external claim, never broad research.
- Don't spawn a subagent by default — only propose one for an unusually large or high-stakes verification job, and ask first, since a subagent's cold-start context costs more than it saves for a normal check.
- Keep the final report to a score line plus a short flagged-claims list. No long explanations, no restating supported claims in detail.
