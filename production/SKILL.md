---
name: security-scan
description: >-
  Solidity smart contract security audit pipeline (recon → detection → kill-gate
  verification → JSON+Markdown report). Use when the user asks to audit, scan,
  review, or find vulnerabilities in smart contracts / Solidity / DeFi protocol
  code; when preparing Code4rena / Sherlock / Cantina / Immunefi contest or
  bounty submissions; or says "security-scan", "审一下这个合约", "查漏洞",
  "跑一次安全扫描", "找找这个协议有没有洞". Also use for re-scanning a codebase
  after new commits.
---

# Security Scan — Multi-Phase AI Auditor

Architecture: multi-phase pipeline (Krait-style). Detection: checklist + domain
primers. Verification: 8 deterministic kill gates. Tools: Slither/Aderyn when
available, graceful AI-only degradation. Output: Arena-compatible JSON + Markdown.

Optimized for **precision over recall**: a report with 45 false positives trains
the reader to stop reading — and that is when they miss the real critical. Every
phase below exists to protect the signal.

## Operating principles

1. **Understand before hunting.** Never emit a finding about code you have not
   mapped. Findings without recon context are how "missing slippage check" ends
   up reported in a governance contract.
2. **Every finding names WHO (attacker), WHAT (entrypoint), HOW MUCH (quantified
   impact).** A finding that cannot answer all three dies at the gates.
3. **Concrete exploit or it does not count.** If you cannot write the step-by-step
   attack, you do not have a finding; you have a hunch. Hunches go to
   `killed_candidates`, not `findings`.
4. **Tool output is leads, not findings.** Slither High/Medium still passes
   through the gates like everything else.
5. **Degrade, never fake.** If Slither/Aderyn are unavailable, record that in
   report metadata and continue AI-only. Do not silently skip.
6. Serial execution, read-only analysis: do not install dependencies, do not
   execute project scripts, do not broadcast transactions.

## Phase 0 — Setup & tool discovery

1. Confirm scope with the user if ambiguous: which directories/files are in
   scope, and (for contest mode) the bounty scope rules.
2. Detect build system: `foundry.toml`, `hardhat.config.*`, `remappings.txt`,
   `solc` version pragma distribution.
3. Check tools (availability only; never install without asking):
   ```bash
   command -v slither; command -v aderyn
   ls ~/.solc-select/artifacts/ 2>/dev/null | head
   ```
   Known-good local setup: slither 0.11.x + solc-select binaries at
   `~/.solc-select/artifacts/solc-<v>/solc-<v>`; invoke with
   `--solc <that-binary-path>` (the `SOLC_VERSION` env var does NOT propagate
   through slither→solc-select).
4. Record tool availability in the report `tools` field.

## Phase 1 — Recon (mandatory; recon section ships before any finding)

Produce a RECON section containing:

1. **Inventory**: per file — contracts, LOC, external call count.
2. **Value map**: every function that moves user funds in or out, ranked by
   amount at risk. These are the detection entry points.
3. **Trust map**: who can call what. Admin/owner/guardian/keeper powers,
   whether behind timelocks, what admin can drain or brick.
4. **Call graph** (textual): `ContractA.f() → ContractB.g() → Token.transfer`,
   cross-contract edges especially those with user-supplied targets.
5. **External dependencies**: tokens (fee-on-transfer? rebasing? odd decimals?
   blacklists?), oracles (which feeds, update cadence, decimals), AMMs, bridges,
   precompiles, L2 sequencer assumptions.
6. **Domain classification**: which primers apply (see
   `references/domain-primers.md`). Multi-domain protocols get multiple primers.
7. **Protocol intent**: one paragraph — what is this supposed to do
   economically? You cannot spot "wrong" until you know "right".

## Phase 2 — Detection

**Always read `references/checklist.md` first** — it is the full vulnerability
taxonomy; sweep every applicable item. Then:

1. **Value-flow first**: walk each entry point from the value map end to end.
2. **Math**: share/price/rate computations; rounding direction (who wins the
   dust); division-before-multiply; precision loss across 1e6/1e8/1e18 mixing.
3. **State machines**: claim/settle/liquidate/withdraw orderings; what state is
   promised before vs after external calls; reentrancy surface per call.
4. **Access**: every state-changing external function — what happens if the
   attacker calls it, including parameter-level abuse of permitted callers.
5. **Integrations**: token transfer assumptions vs reality; oracle freshness,
   deviation and manipulation cost; callback surfaces (ERC-777/721/1155 hooks,
   uniswapV3SwapCallback, onReceive style hooks).

Cross-check with static tools when available:

```bash
# foundry project:
slither . --solc ~/.solc-select/artifacts/solc-<v>/solc-<v>
# standalone file(s):
slither src/Foo.sol --solc <binary>
```

Merge Slither output as *candidates*. Dedupe against checklist findings by
root cause (not by tool label). When a candidate smells like a known incident
class, read `references/exploit-patterns.md` for the matching detection heuristic.

## Phase 3 — Verification: the 8 kill gates

Apply all gates to every candidate. **Any fail = kill** (move to
`killed_candidates` with the gate and one-line reason). Nothing reaches the
report ungated.

| # | Gate | Question |
|---|------|----------|
| G1 | Generic best practice | Is this style/best-practice with no concrete loss path? |
| G2 | Theoretical exploit | Can you write the concrete step-by-step exploit? Precondition chains deeper than 2 invented steps = KILL. |
| G3 | Intentional design | Do code comments/README/docs show this is a documented, intentional tradeoff? |
| G4 | WHO/WHAT/HOW MUCH | Missing attacker profile, entrypoint, or quantified impact? |
| G5 | Admin trust | Only reachable by trusted roles whose broad power is accepted design? (Centralization as such gets ONE Info finding per report, not per-function noise.) |
| G6 | Dust impact | Quantified impact < $100 or < 0.01% of TVL? |
| G7 | Out of scope | Outside the declared audit/contest scope? |
| G8 | Already documented | Already in known-issues, public audits, or past contest reports? (If no internet access, mark confidence accordingly and say so.) |

Severity scale:

- **Critical** — unauthenticated fund loss, or solvency break with a concrete path.
- **High** — fund loss with realistic conditions, or permanent invariant break.
- **Medium** — bounded loss, griefing with cost-asymmetry in attacker's favor,
  temporary DoS of critical functions.
- **Low** — minor loss / repairable accounting drift.
- **Info** — centralization notes, observations.

Confidence: High (exploit steps verified against code paths), Medium (logic
sound, one link unverified), Low (survived gates but untested edge).

## Phase 4 — Report (JSON + Markdown)

Emit BOTH, exactly these shapes.

JSON (machine-parseable, Arena-compatible):

```json
{
  "protocol": "<name>",
  "scan_date": "YYYY-MM-DD",
  "scope": ["<files/dirs>"],
  "tools": {"slither": "<version|missing>", "aderyn": "<version|missing>", "solc": "<version>"},
  "domain": ["<primer names>"],
  "findings": [
    {
      "id": "F-001",
      "title": "<imperative one-liner>",
      "severity": "Critical|High|Medium|Low|Info",
      "confidence": "High|Medium|Low",
      "file": "src/X.sol",
      "lines": "120-135",
      "who": "<attacker profile>",
      "entrypoint": "<function + call path>",
      "impact": "<quantified: USD, share of TVL, or broken invariant>",
      "exploit_steps": ["1. ...", "2. ...", "3. ..."],
      "poc_sketch": "<optional foundry/solidity sketch>",
      "gates_passed": ["G1","G2","G3","G4","G5","G6","G7","G8"],
      "recommendation": "<fix direction>"
    }
  ],
  "killed_candidates": [
    {"candidate": "<one-liner>", "gate": "G#", "reason": "<one line>"}
  ],
  "stats": {"files": 0, "loc": 0, "candidates": 0, "reported": 0, "killed": 0}
}
```

Markdown: findings ordered by severity; each with code excerpt, attack-path
table (who/entrypoint/impact), exploit steps, fix. Close with the
killed-candidates table — showing what the gates killed IS the product's
trust story.

**Contest mode** (target is C4/Sherlock/Cantina/Immunefi): additionally, per
finding, add a `discovery_path` (how it was found: which recon observation led
where). Also enumerate which invariants hold (what you checked and found safe) —
judges and leads reward demonstrated coverage, and it feeds the workspace
exclusion-list discipline (see [[user-sandbox-rules]]: 漏洞分类学全覆盖).

## References (read on demand)

- `references/checklist.md` — full vulnerability taxonomy A–N (read in Phase 2, always)
- `references/domain-primers.md` — per-protocol-type primers (read after domain classification)
- `references/exploit-patterns.md` — real incidents → code smells → heuristics
