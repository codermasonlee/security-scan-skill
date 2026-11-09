---
name: security-scan
description: Single-pass security auditor for Solidity smart contracts
invocation: /security-scan
---

# Single-Pass Security Auditor

## When Invoked

Perform a comprehensive single-pass security scan of all Solidity (.sol) files in the current project.

## Prerequisites

Before running this audit skill, run the following tools on the project. Each tool adds a layer of analysis:

### 1. Static Analysis
Run at least one:
- `slither . --json .audit/slither-output.json` (install: `pip install slither-analyzer`)
- `aderyn . --output .audit/aderyn-report.md` (install: `cargo install aderyn`)

### 2. Fuzz Testing (Foundry projects)
If this is a Foundry project (foundry.toml exists):
- `forge test -vv > .audit/test-results.txt 2>&1`
- If fuzz tests exist: `forge test --match-test "Fuzz" -vv > .audit/fuzz-results.txt 2>&1`

### 3. Optional: Property Testing
- Echidna: `echidna . --config echidna.yaml > .audit/echidna-results.txt 2>&1`
- Medusa: `medusa fuzz > .audit/medusa-results.txt 2>&1`

## Tool Integration

When starting the audit, check for results from all tools:

1. **Static analysis** (`.audit/slither-output.json` or `.audit/aderyn-report.md`)
   - Extract findings as detection hints
   - Cross-reference with AI findings for confidence boosting

2. **Test results** (`.audit/test-results.txt`)
   - Check for failing tests (potential bugs)
   - Check test coverage (untested code = higher risk)

3. **Fuzz results** (`.audit/fuzz-results.txt`)
   - Failing fuzz tests are PROVEN bugs (highest confidence)
   - Include as Critical/High with test case as evidence

4. **Property testing** (`.audit/echidna-results.txt` or `.audit/medusa-results.txt`)
   - Property violations are PROVEN invariant breaks
   - Include with the exact property that was violated

### Confidence Tiers Based on Evidence
- **Tier 1 (Proven):** Fuzz test or property test failure → CRITICAL/HIGH
- **Tier 2 (Corroborated):** AI + static analysis agree → HIGH confidence
- **Tier 3 (AI-detected):** AI finding, no tool corroboration → Standard confidence
- **Tier 4 (Tool-only):** Tool finding, AI does not corroborate → Low confidence, needs review

If no tool output files are found, proceed with AI-only analysis and note which tools were missing.

## Methodology

### 1. Scope Discovery
- List all .sol files with line counts
- Identify the main contracts (not interfaces, not libraries, not tests)
- Note any imports and dependencies

## Reconnaissance Phase

Before analyzing for vulnerabilities, perform these steps:

### 1. File Inventory
List all Solidity (.sol) files in scope. For each file, report:
- File path
- Line count
- Whether it is a contract, interface, library, or abstract contract
- Number of external/public functions
- Number of state variables

### 2. Risk Scoring
For each contract (not interfaces or libraries), calculate a rough risk score:
- +5 for each external call (address.call, token.transfer, etc.)
- +4 for each state-modifying external/public function
- +4 for each payable function
- +3 for each use of assembly or unchecked blocks
- +1 for each 20 lines of code

Sort files by risk score descending. The top files get the deepest analysis.

### 3. Architecture Overview
Identify:
- Contract inheritance chains
- Which contracts call which other contracts
- Trust boundaries (who can call what, access control patterns)
- External protocol integrations (oracles, DEXes, lending pools)

### 4. Entry Point Catalog
List all external/public functions with:
- Function signature
- Whether it modifies state
- Whether it handles value (ETH or tokens)
- Access control (onlyOwner, role-based, or unrestricted)

Output this as a "Recon Summary" section BEFORE proceeding to detection.

## Exploit Context

When analyzing for vulnerabilities, reference these real-world examples to calibrate your detection:

### Reentrancy Examples
- The DAO hack (2016): $60M stolen. Recursive call in withdraw drained funds before balance update.
- Cream Finance (2021): $130M. Read-only reentrancy via price oracle manipulation during reentrant call.
- Curve (2023): $62M. Vyper compiler bug allowed reentrancy despite @nonreentrant decorator.

Pattern to detect: External call to untrusted address BEFORE state update. Look for: .call{value:}, .transfer(), token.safeTransfer() followed by state changes.

### Oracle Manipulation Examples
- Mango Markets (2022): $114M. Attacker pumped spot price of MNGO token, borrowed against inflated collateral.
- Inverse Finance (2022): $15.6M. Used flash loan to manipulate Uniswap TWAP in single block.
- Bonq (2023): $120M. Manipulated TellorFlex oracle with single staking transaction.

Pattern to detect: Any use of spot price (balanceOf / totalSupply, or single DEX read) for value calculations. Look for: missing TWAP, missing price deviation checks, single-source oracles.

### Access Control Examples
- Ronin Bridge (2022): $624M. Compromised 5 of 9 multisig validators. Threshold too low.
- Wormhole (2022): $326M. Uninitialized proxy allowed attacker to call guardian set upgrade.
- Wintermute (2022): $162M. Vanity address generator vulnerability gave attacker private key access.

Pattern to detect: Missing access modifiers on functions that change critical state. Unprotected initialization. Single-point-of-failure admin patterns.

### Flash Loan Examples
- bZx (2020): $8M. Flash loan + oracle manipulation + leveraged trade in single transaction.
- Pancake Bunny (2021): $45M. Flash loan to manipulate LP token price, then deposit at inflated value.
- Euler Finance (2023): $197M. Flash loan + donate function to create bad debt.

Pattern to detect: Functions where large temporary capital could create profit. Price calculations that use current reserves. Donation attacks on share-based systems.

When reporting findings, reference the most similar real exploit to justify the severity and demonstrate the attack is practical, not theoretical.

## Detection Phase: Multi-Mindset Analysis

After completing reconnaissance, analyze each contract using 4 independent mindsets. Process the highest-risk files from recon first.

For EACH external/public function in scope:

### Mindset 1: The Attacker
Think like someone trying to steal funds or break the protocol.
- How would I exploit this function?
- Can I manipulate inputs to extract more than I deposited?
- Can I reenter during execution?
- Can I front-run or sandwich other users?
- Can I use flash loans to amplify an attack?
- What is the maximum damage I can cause?
Tag findings as [ATTACKER].

### Mindset 2: The Accountant
Think like someone auditing financial correctness.
- Do the numbers add up across all paths?
- Is fee calculation correct? (rounding direction, precision)
- Are shares/tokens minted and burned proportionally?
- Could precision loss accumulate over many operations?
- Are balances consistent before and after the function?
- Could someone profit from rounding errors at scale?
Tag findings as [ACCOUNTANT].

### Mindset 3: The Spec Auditor
Think like someone verifying code matches documentation.
- Does this function do what its name implies?
- Are documented invariants actually enforced in code?
- Does error handling match the expected behavior?
- Are events emitted for all state changes?
- Does the access control match the documented roles?
- Are edge cases in the specification handled?
Tag findings as [SPEC].

### Mindset 4: The Edge Case Hunter
Think like someone testing boundary conditions.
- What happens with amount = 0?
- What happens at uint256 max value?
- What happens with empty arrays or zero-length inputs?
- What happens if an external call reverts?
- What happens on first use (empty state)?
- What happens on last use (final withdrawal, last NFT)?
- What happens if called twice in the same transaction?
Tag findings as [EDGE].

### Merging (Important: How This Works in Practice)

Claude processes each mindset one at a time in a single response. It tags findings as it goes. At the end, it reviews all tags and merges duplicates. This is automatic within one prompt execution.

1. **Tagging**: Each finding is tagged with its source mindset [ATTACKER], [ACCOUNTANT], etc.
2. **After all 4 mindsets complete**, review all tagged findings
3. **Group by root cause**: If [ATTACKER] and [EDGE] both flagged the withdraw() function for reentrancy, that is ONE finding with two perspectives
4. **Note consensus**: "Found by: ATTACKER, EDGE" (2 mindsets = higher confidence)
5. **Resolve severity conflicts**: If mindsets disagree on severity, use the HIGHER severity with a note

### 2. Vulnerability Scan
For each contract, check for ALL of the following categories:

**Access Control**
- Missing access modifiers on sensitive functions
- Use of tx.origin instead of msg.sender
- Unprotected initialization functions
- Missing ownership transfer patterns

**Reentrancy**
- External calls before state updates (CEI violation)
- Missing reentrancy guards on functions that transfer value
- Callback patterns without protection

**Integer Issues**
- Unchecked arithmetic in financial calculations
- Division before multiplication (precision loss)
- Unsafe casting between types

**Oracle & Price Manipulation**
- Use of spot prices (single-block readings)
- Missing TWAP or time-weighted validation
- Chainlink stale price checks missing

**Token Handling**
- Missing return value checks on ERC-20 transfers
- Fee-on-transfer token compatibility
- Rebasing token handling

**Protocol-Specific**
- Flash loan attack vectors
- Sandwich attack vectors
- Front-running vulnerabilities
- Missing slippage protection
- Missing deadline checks

## Verification Phase: Confidence Scoring

After detection, assign a confidence score to each finding based on multiple factors.

### Scoring Factors (each 0-25 points, total 0-100):

#### Factor 1: Evidence Strength (0-25)
- 25: Complete exploit scenario with exact function calls, parameters, and profit calculation
- 20: Exploit scenario with some assumptions
- 15: Clear vulnerability pattern but no complete exploit
- 10: Suspicious code pattern, theoretical risk
- 5: Minor concern, stylistic issue
- 0: Generic observation, no security relevance

#### Factor 2: Code Context (0-25)
- 25: Vulnerability in a function that handles significant value (>$10K typical flow)
- 20: Vulnerability in access-controlled function with bypass
- 15: Vulnerability in frequently called function
- 10: Vulnerability in edge-case path
- 5: Vulnerability in rarely triggered code
- 0: Vulnerability in unreachable or dead code

#### Factor 3: Detection Consensus (0-25)
If you used multi-mindset detection:
- 25: Found by 3+ mindsets independently
- 20: Found by 2 mindsets
- 15: Found by 1 mindset with strong evidence
- 10: Found by 1 mindset with moderate evidence
- 5: Found by 1 mindset with weak evidence
If single-pass: score based on pattern match strength instead.

#### Factor 4: Historical Precedent (0-25)
- 25: Identical pattern caused a known exploit (cite specific incident)
- 20: Very similar pattern caused a known exploit
- 15: Same vulnerability class has caused exploits in other protocols
- 10: Vulnerability class is well-documented but no close match
- 5: Theoretical vulnerability, no known precedent
- 0: No historical evidence

### Confidence Classification:
- **70-100**: HIGH confidence → Report as-is with full severity
- **50-69**: MEDIUM confidence → Report with severity, note "verify manually"
- **30-49**: LOW confidence → Report only if HIGH/CRITICAL severity
- **0-29**: INSUFFICIENT → Do not report (log for analysis)

### How to Read the Score:
A clear reentrancy with a complete exploit scenario in a value-handling function should score 70+. A vague "consider adding a check" observation should score under 30. If most of your real findings score below 50, adjust the weights to better reflect your detection quality.

For each finding, show: [SEVERITY] Title (Confidence: XX/100)
Break down the 4 scoring factors so the reader understands WHY the confidence score is what it is.

### 3. Output Format
For each finding, report:

**[SEVERITY] Title**
- **File**: contract.sol:lineNumber
- **Description**: Clear explanation of the vulnerability
- **Impact**: What an attacker could do
- **Recommendation**: How to fix it

Severity levels: CRITICAL, HIGH, MEDIUM, LOW

Only report findings you are confident about. Do not report generic best practices or theoretical issues without a concrete exploit scenario.

## Output Format

Generate a comprehensive audit report with the following sections:

### 1. JSON Report (extended format)
Write to `.audit/findings.json` with the JSON + Markdown format from above, PLUS these additional fields per finding:

{
  "findings": [
    {
      "id": "[SKILL-001]",
      "title": "[title]",
      "severity": "critical|high|medium|low",
      "confidence": "high|medium|low",
      "confidenceScore": [0-100],
      "file": "[path]",
      "line": [number],
      "endLine": [number],
      "description": "[description]",
      "exploitScenario": {
        "attacker": "[who]",
        "steps": ["step 1", "step 2", "..."],
        "profit": "[estimated profit]",
        "cost": "[attack cost]"
      },
      "impact": "[impact]",
      "remediation": "[fix]",
      "codeSnippet": "[relevant code]",
      "category": "[category]",
      "detectedBy": ["mindset1", "mindset2"],
      "historicalReference": "[similar real exploit if known]",
      "discoveryPath": "[how this finding was discovered]"
    }
  ]
}

### 2. Markdown Report
Write to `.audit/report.md` with full details:

# Security Audit Report

## Executive Summary
One paragraph overview: what was audited, key risks found, overall assessment.

## Methodology
Describe your architecture, detection strategy, and verification approach.

## Findings
For each finding (sorted by severity):

### [SEVERITY-ID] Title
**Confidence**: XX/100 (or High/Medium/Low)
**File**: path:line-endLine
**Category**: category

**Description**
Full explanation of the vulnerability.

**Exploit Scenario**
1. Attacker does X
2. Contract responds with Y
3. Attacker extracts Z

**Vulnerable Code**
(code snippet)

**Impact**
Quantified damage assessment.

**Recommendation**
Specific fix with before/after code.

**Historical Reference**
Similar exploit: [name, year, amount lost]

---

## Appendix
- Scope: X files, Y LOC
- Duration: Z seconds
- Tools used: [list]
- Detection method: [your choice from Step 5]
- Verification method: [your choice from Step 7]

### 3. Console Summary
Print brief summary to terminal after writing files.
