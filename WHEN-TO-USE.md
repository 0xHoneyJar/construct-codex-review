# When to call FAGAN — the niche this construct fills

> Quick reference for when FAGAN is the right reviewer vs `/bridgebuilder-review`, `/bug`, or `/flatline-review`.

## The niche

FAGAN does **diff-scoped, single-pass, fix-first review**. Designed for the moments where:

- You wrote 30-200 lines of code
- You want a rigorous second pair of eyes
- `/bridgebuilder-review` is too heavyweight (PR-level, multi-pass, 4+ minutes)
- `/bug` is too slow (requires bug report, sprint creation, full triage flow)
- The code is small enough that 1 GPT pass with line-anchored fixes is the right shape

## Decision matrix

| situation | tool | why |
|---|---|---|
| "I just wrote this 50-line function — is it broken?" | **FAGAN** (`/reviewing-diffs`) | diff-scoped, single-pass, fixes-included |
| "I refactored this hot path — sanity check" | **FAGAN** | quick adversarial second look |
| "Did I introduce a security bug in this auth code?" | **FAGAN** | walks P1-P8 patterns; named CVE families |
| "Production incident — what did we ship that broke" | `/bug` | needs triage, repro, sprint, formal fix flow |
| "Reviewing a 1000-line PR before merge" | `/bridgebuilder-review` | PR-level, multi-pass, persona-rich, GitHub-trail-aware |
| "Designing a new system — adversarial design review" | `/flatline-review` or `/red-team` | planning-artifact territory |
| "Formal cross-model gate before security-touching merge" | `/bridgebuilder-review` THEN FAGAN (the dissent gate) | layer-6 multi-model adversarial |

## Why FAGAN is faster than `/bug` for ad-hoc review

`/bug` triggers:
1. Bug eligibility check (must reference observed failure or stack trace)
2. Hybrid interview (operator answers questions about repro)
3. Codebase analysis
4. Micro-sprint creation with test-first contract
5. /implement → /review-sprint → /audit-sprint cycle
6. Commit → audit gate → ship

FAGAN does:
1. `bash scripts/codex-review-api.sh review-diff /tmp/changes.diff`
2. Returns structured JSON in 30-60s with critical/major findings + current_code/fixed_code

`/bug` is a heavyweight TICKET workflow. FAGAN is a lightweight INSPECTION workflow. The difference is whether you have a confirmed bug (use `/bug`) or want to KNOW IF you have a bug (use FAGAN).

## Why FAGAN is leaner than `/bridgebuilder-review` for small diffs

`/bridgebuilder-review` runs:
1. Persona integrity check + lore load + context load
2. Pass 1 convergence review (~90s, ~$1.50)
3. Pass 2 enrichment review (~150s, ~$1.50)
4. Findings JSON + full review markdown saved to `.run/bridge-reviews/`
5. PR comment posted with persona voice + educational metaphors
6. Vision capture + lore reference scan + GitHub trail
7. Total: ~5min, ~$3-5, full PR audit trail

FAGAN runs:
1. Read diff → walk patterns → emit findings
2. Total: ~30-60s, ~$0.50-1, JSON-only output

`/bridgebuilder-review` is for PR-level shipping decisions. FAGAN is for ad-hoc "is this code OK" gut-checks. Use BOTH on security-touching merges (FAGAN as the cross-model dissent gate after Opus's bridgebuilder pass) — the cycle-032 pattern.

## Composition examples

### Solo invocation — quick sanity check

```bash
git diff > /tmp/changes.diff
bash scripts/codex-review-api.sh review-diff /tmp/changes.diff
# JSON to stdout · exit 0 = APPROVED · exit 1 = CHANGES_REQUIRED
```

### Implement-then-review loop (with codex-rescue)

```yaml
# Use the loa-compositions/compositions/delivery/code-implement-and-review.yaml
# composition. codex-rescue implements, FAGAN reviews, iterate up to 3 cycles.
```

### Cross-model dissent gate (after bridgebuilder-review)

```bash
# 1. bridgebuilder-review runs Opus pass on PR #N
gh pr review --approve  # if you choose

# 2. Pull Opus findings
gh pr view N --json reviews | jq -r '.reviews[-1].body' > /tmp/prior.md

# 3. FAGAN as cross-model dissent — find what Opus missed
bash scripts/codex-review-api.sh review-diff /tmp/diff.patch
```

This is the pattern that produced cycle-032's C1 + C2 + C3 findings — three contract-violating issues neither of two Opus passes had found.

## What FAGAN explicitly does NOT do

- **Style review** — that's craft-gate (artisan) territory
- **Design review** — that's `/flatline-review` for PRD/SDD/Sprint
- **Multi-pass enrichment** — single pass + 3-iteration cap; if you want pass-2, use bridgebuilder-review
- **GitHub PR posting** — outputs JSON; orchestrate posting yourself if needed
- **Iterative fix generation** — provides fixes in findings; doesn't apply them

## Latency / cost reference

| invocation | latency | cost | depth |
|---|---|---|---|
| FAGAN single review | 30-60s | $0.50-1 | 1 model, 1 pass |
| FAGAN 3-iteration convergence loop | 90-180s | $1.50-3 | 1 model, ≤3 passes |
| `/bridgebuilder-review` two-pass | 240-300s | $3-5 | 1 model, 2 passes (convergence + enrichment) |
| `/flatline-review` 3-model | 600s+ | $5-10 | 3 models adversarial |
| `/bug` triage + impl + review | 30min+ | $5-15 | full sprint workflow |

## TL;DR

**Reach for FAGAN when**:
- You wrote ≤200 lines and want it inspected RIGHT NOW
- `/bug` feels like overkill (no observed failure, just "is this OK?")
- `/bridgebuilder-review` feels heavy (no PR yet, or just exploring)
- You want the cross-model dissent leg of a multi-model adversarial review

**Don't reach for FAGAN when**:
- You're reviewing a planning artifact (PRD/SDD) — use `/flatline-review`
- You need style/UI review — use artisan
- You have a confirmed bug with repro — use `/bug` for the formal flow
- You want multi-iteration deep PR review — use `/bridgebuilder-review`
