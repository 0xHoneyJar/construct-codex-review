# construct-codex-review

> Lean adversarial code review for diffs and implementations. Single GPT pass via codex CLI, structured JSON findings, convergence loop.

A construct in the [Loa Constructs](https://github.com/0xHoneyJar/loa-constructs) ecosystem. Designed to occupy a clean responsibility seam **below** Flatline Protocol: Flatline reviews planning artifacts (PRD/SDD/Sprint), codex-review reviews **code diffs**.

## Quick links

- **[WHEN-TO-USE.md](./WHEN-TO-USE.md)** — niche-fit: FAGAN vs `/bug` vs `/bridgebuilder-review` vs `/flatline-review`
- **[resources/patterns.md](./resources/patterns.md)** — 17 patterned-finding shapes (cycle-032 distillation), each with named CVE family and surface signal
- **[identity/persona.yaml](./identity/persona.yaml)** — FAGAN's full voice + boundaries

## Persona

**FAGAN** — after Michael Fagan, who invented formal code inspection at IBM in 1976. Line-anchored. Evidence-based. Provides actual code fixes, not descriptions. Converges toward approval on re-review. Walks the 17 patterns in `resources/patterns.md` against every diff.

## Install (via constructs-cli)

```bash
npx constructs-cli install codex-review
```

Or directly via git:

```bash
git clone https://github.com/0xHoneyJar/construct-codex-review.git ~/.claude/constructs/packs/codex-review
```

## Requirements

- `codex` CLI installed (defaults to `gpt-5.5`; override via `CODEX_REVIEW_MODEL`)
- `OPENAI_API_KEY` in environment
- `jq`, `bash >= 4`

## Usage

### Standalone — review a diff

```bash
git diff main..HEAD > /tmp/changes.diff
bash scripts/codex-review-api.sh review-diff /tmp/changes.diff
# stdout: structured JSON
# exit 0 = APPROVED, 1 = CHANGES_REQUIRED
```

### Standalone — review specific files

```bash
bash scripts/codex-review-api.sh review-files src/auth.ts src/session.ts
```

### Composition — implement → review loop

Use the [`code-implement-and-review`](https://github.com/0xHoneyJar/loa-compositions/blob/main/compositions/delivery/code-implement-and-review.yaml) composition (in `loa-compositions`) to pair this construct with `codex-rescue` (or any implementer) and iterate until APPROVED or the cap is reached.

## Output

JSON conforming to [`schemas/codex-review-finding.schema.json`](./schemas/codex-review-finding.schema.json):

```json
{
  "verdict": "APPROVED" | "CHANGES_REQUIRED",
  "summary": "...",
  "findings": [
    {
      "severity": "critical" | "major",
      "file": "src/foo.ts",
      "line": 42,
      "description": "...",
      "current_code": "```...```",
      "fixed_code": "```...```",
      "explanation": "..."
    }
  ],
  "fabrication_check": { "passed": true, "concerns": [] },
  "iteration": 1
}
```

## Configuration

Environment variables:

| Var | Default | Description |
|---|---|---|
| `CODEX_REVIEW_MODEL` | `gpt-5.5` | Model id for codex CLI |
| `CODEX_REVIEW_TIMEOUT` | `300` | Seconds per invocation |
| `CODEX_REVIEW_MAX_ITERATIONS` | `3` | Iteration cap (auto-approves past this) |
| `CODEX_REVIEW_MAX_TOKENS` | `30000` | Token budget for prepared content |

Optional `.loa.config.yaml` keys (project-level):

```yaml
codex_review:
  secret_patterns:           # additional regex patterns for redaction
    - 'my-internal-pattern-[A-Z0-9]{20}'
```

## Convergence

- Iteration cap defaults to 3.
- Past the cap, the API auto-approves at the wrapper level (no model invocation), returning `{auto_approved: true, note: "iteration-cap-reached"}`.
- The re-review prompt is the load-bearing convergence asset:
  > **VERIFY. DON'T REINVENT. CONVERGE.**

## Boundaries

| Surface | Tool | When |
|---|---|---|
| PRD / SDD / Sprint planning | **Flatline Protocol** (Opus + GPT-5.3-codex + Gemini, 4-persona) | High-stakes, slow, multi-model dissent |
| Code diff after implementation | **codex-review** (single GPT pass via codex CLI, single persona) | Lean, fast, composable as a stage |
| UI feel / animation curves | **artisan** | Design surface, not bugs |
| Architecture audit | **audit-* compositions** | Cross-cutting, multi-pass |

## Layout

```
construct-codex-review/
├── construct.yaml                       ← source of truth (schema_version 3)
├── identity/
│   ├── persona.yaml                     ← FAGAN
│   └── expertise.yaml
├── skills/
│   ├── reviewing-diffs/                 ← primary
│   └── reviewing-files/                 ← secondary
├── scripts/
│   ├── codex-review-api.sh              ← lean wrapper (~190 lines)
│   ├── lib/                             ← VENDORED — see VENDOR.md
│   │   ├── lib-codex-exec.sh
│   │   ├── lib-security.sh
│   │   └── lib-content.sh
│   └── tests/
│       └── *.bats
├── prompts/
│   ├── code-review.md                   ← first-review (FAGAN)
│   └── re-review.md                     ← convergence prompt
├── schemas/
│   └── codex-review-finding.schema.json ← draft-07, permissive
└── VENDOR.md                            ← vendor pin + adaptations
```

## Lineage

This construct exists because PR #523 in `loa-constructs` deprecated the in-tree `/gpt-review` for honest reasons (orphan code, broken tests, silent hooks, Flatline absorbed its primary value). Rather than resurrect it, this construct **occupies a different seam**: lean, single-persona, no hooks, composable as a stage in larger workflows.

What it learned from `/gpt-review`'s deprecation:
- **No silent hooks** — explicit invocation only
- **Tests reflect reality** — happy-path-first, no aspirational asserts
- **Clear scope from Flatline** — code diffs only; planning is Flatline's territory
- **Convergence discipline** — re-review prompt explicitly forbids new findings on iteration 2+

See [`VENDOR.md`](./VENDOR.md) for the vendored libraries' provenance and re-vendor procedure.

## License

MIT
