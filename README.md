# hallucination-check

A Claude skill that checks whether a Claude answer (or any specific claim) is likely hallucinated, and reports a probability score — without burning a lot of tokens doing it.

## Why

LLMs sound exactly as confident when they're wrong as when they're right, so tone isn't a useful signal. What's actually checkable is whether the *specific facts* in an answer — numbers, citations, named entities, computed results — hold up when you test them independently. This skill automates that check, but deliberately keeps it cheap: it checks against data you already have before it reaches for a search or a fresh generation, and it caps itself at a handful of claims per run instead of re-verifying an entire answer line by line.

## What it does

1. Picks a target — the last Claude answer in the conversation, or specific text/claims you point it at.
2. Pulls out the highest-risk checkable claims (up to 8): numbers, statistics, named entities, dates, citations, quantitative or causal statements. Skips opinions and generic explanation, since those aren't checkable.
3. Verifies each claim with the cheapest method that applies, in order:
   - Check directly against data/files/context you provide (free — no extra generation)
   - Re-run the actual computation if a number came from code, instead of trusting the prose summary
   - One targeted search per external fact or citation — never a broad re-research
   - As a last resort, a short independent re-answer of just that one claim, compared against the original
4. Scores each claim ✅ Supported / ⚠️ Unverifiable / ❌ Contradicted, and aggregates into an overall hallucination probability:
   - **0–20% Low** — proceed as normal
   - **20–50% Medium** — worth a second look
   - **50–100% High** — treat the answer as unreliable until re-verified
5. Reports back short: the score, and only the flagged claims — not a full transcript of the reasoning.

## Installation

**Claude Code / Claude Agent SDK:** drop the `hallucination-check/` folder into your skills directory (e.g. `~/.claude/skills/`) so `SKILL.md` sits at `hallucination-check/SKILL.md`.

**Claude.ai / Cowork:** upload `SKILL.md` (or the whole folder) through the skill upload flow in your account's skill settings.

**Any other Claude surface that supports custom skills:** follow that surface's normal skill-install steps — the only required file is `SKILL.md`; this repo just adds a README and license around it.

## Usage

Once installed, just ask normally — you don't need to name the skill explicitly:

> "Can you double-check that last answer for hallucination?"

> "Verify the gene IDs and fold-change numbers in your last response against this CSV." *(attach the file — the skill will check directly against it instead of re-deriving anything)*

> "Fact-check this paragraph: [paste text]"

Example output shape:

```
Hallucination check: 15% (Low)
6/7 claims supported.

⚠️ "Gene XYZ1 shows 3.2-fold upregulation" — no source data provided to confirm
   the exact figure; self-consistency re-check gave 2.8–3.4x, roughly consistent
   but not exact.
```

If you have a specific dataset, document, or question you want the claim checked against, mention it up front (or attach the file) — the skill will use it directly instead of asking, and it's the cheapest and most reliable check available.

## Design notes

- **Token budget is a first-class constraint, not an afterthought.** The skill is ordered cheapest-check-first on purpose, caps claim count at 8 by default, and explicitly avoids spawning subagents or doing broad research unless asked.
- **It doesn't re-verify everything.** Opinions, hedged language, and generic explanation are skipped — they're not falsifiable, so checking them just burns tokens for no signal.
- **It's most reliable with real data.** Given a source file or dataset to check against, this is close to a deterministic check. Without one, it falls back to self-consistency probing, which is a weaker (but still useful) signal.

## License

[PolyForm Noncommercial License 1.0.0](LICENSE) — free for personal, academic, research, hobby, and nonprofit/government use. Any commercial use (a company using this, including internally, to make or save money) is **not** licensed by default and requires a separate commercial agreement.

### Commercial licensing

If your company wants to use this skill, contact **dhruvaditya88@gmail.com** to arrange a commercial license. (Swap in whatever contact you prefer before publishing.)

Note: this isn't legal advice, and this license doesn't collect payment automatically — it just makes unlicensed commercial use a breach, which is what gives you standing to ask a company to pay. You (or a lawyer, if it ever gets contentious) still have to notice the use and follow up.
