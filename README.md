# security-scan — AI Security Auditor Skill

An AI auditor skill for Solidity smart contracts, built following the
[Zealynx Academy — AI Auditor Builder](https://academy.zealynx.io/ai-agents/security/build-ai-auditor)
course (12 steps) and benchmarked against the course's official test targets.

## What's in this repo

| Path | What it is |
|---|---|
| [`skills/security-scan.md`](skills/security-scan.md) | **The course artifact** (Step 12 deliverable) — single-pass auditor with recon risk-scoring, 4-mindset detection, exploit-context calibration, 4-factor confidence scoring, full-stack tool integration, and full reporting suite. Drop into `.claude/skills/` and invoke `/security-scan`. |
| [`production/`](production/SKILL.md) | **The production variant** — multi-phase pipeline (recon → detection → kill-gate verification → report) with a 90-item A–N vulnerability taxonomy, 7 domain primers, 13 real-incident exploit patterns, and 8 deterministic kill gates. Lives in `.agents/skills/security-scan/` with its `references/` directory. |
| [`BENCHMARKS.md`](BENCHMARKS.md) | Scored runs against the course's official test project and a C4 contest repo (Basin). |

## The five decisions

| Decision | Course artifact | Production variant |
|---|---|---|
| Architecture | Single-pass | Multi-phase (Krait-style) |
| Detection | Multi-mindset (ATTACKER / ACCOUNTANT / SPEC / EDGE) + 12 real-incident exploit context + recon risk scoring | Checklist-based (taxonomy A–N, 90 items) + 7 domain primers |
| Verification | Confidence scoring (4 factors, 0–100, statistical weighting) | Kill gates (8 deterministic rules — any fail = kill) |
| Tool integration | Full stack (Slither/Aderyn → tests → fuzz → property testing, Tier 1–4 evidence) | Static analysis (Slither) with graceful AI-only degradation |
| Output | Full suite (`.audit/findings.json` extended schema + `.audit/report.md` + console summary) | JSON + Markdown (Arena-compatible shape) |

The two variants are complementary: the course artifact optimizes per-finding evidence
quality (exploit scenarios, profit/cost, historical references, confidence breakdowns);
the production variant optimizes precision-per-report (nothing reaches the reader
ungated) with mandatory recon and invariant-coverage statements.

## Install

**Course artifact (Claude Code):**
```bash
git clone <this-repo>
mkdir -p ~/.claude/skills && cp security-scan-skill/skills/security-scan.md ~/.claude/skills/
# in any Solidity project:
/security-scan
```

**Production variant (ZCode or any agent runtime reading `.agents/skills/`):**
```bash
mkdir -p .agents/skills/security-scan
cp security-scan-skill/production/SKILL.md .agents/skills/security-scan/SKILL.md
cp -r security-scan-skill/production/references .agents/skills/security-scan/references
```

## Requirements

- Optional but recommended: [`slither`](https://github.com/crytic/slither) (`pip install slither-analyzer`) + a solc binary per pinned version. The skill degrades to AI-only and records missing tools when unavailable.
- Foundry projects: `forge test` results are consumed as Tier-1 evidence when present.

## Benchmark results

Scored blind/semi-blind against official targets — see [`BENCHMARKS.md`](BENCHMARKS.md):

| Target | Size | Precision | Recall |
|---|---|---|---|
| Zealynx official test project (`ai-auditor-test-project`) | 3 contracts, 5 planted bugs | 50% | 40% |
| Basin (C4 2023-07, Arena target) | 10 files, 1,145 SLOC, 13 official findings | **100%** | **92%** |

## Credits

Built following the Zealynx Academy "Build an AI Auditor" course framework
(five decision points, step-by-step skill assembly). All skill text, taxonomy,
primers, and incident patterns authored independently. Educational use.
