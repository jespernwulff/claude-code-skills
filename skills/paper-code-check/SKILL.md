---
name: paper-code-check
description: >
  Cross-check an academic paper against the project's code to verify that the code
  actually implements what the paper claims. Extracts every checkable claim from the
  manuscript (equations, algorithms, parameter values, simulation budgets, sample sizes,
  table/figure numbers, software statements), traces each claim to the code and output
  files that should back it, and verifies the implementation matches — producing a
  claim-by-claim traceability report with file:line evidence. Use this whenever the user
  asks to check whether the code matches the paper, verify the paper against the code,
  audit paper-code consistency, validate that reported numbers come from the pipeline,
  or prepare a replication package — even if they just say "does the code do what the
  paper says?" or "check my numbers before submission".
---

# Paper–Code Cross-Check

Verify that a project's code implements what its paper claims. The unit of work is the
**claim**: a specific, checkable statement in the manuscript that the code either backs
or doesn't. The output is a traceability report: every claim, where it lives in the
paper, where it lives in the code, and a verdict with evidence.

This is a **read-only** review. Never modify the paper, the code, or any output file.
If the project has its own CLAUDE.md rules (e.g., never delete, use a legacy folder),
they apply to any working files you create.

## Why claims-first

A generic code review reads the code and comments on quality. That misses the failure
mode that actually sinks papers: the manuscript says "2,000 Monte Carlo replications"
but the script runs 500; the equation in Section 3 has a term the code never computes;
Table 4's numbers came from a superseded run of a script that has since changed. These
are only findable by starting from the paper's claims and demanding evidence for each
one. So: extract claims first, then verify — never the reverse.

## Step 1 — Inventory (orchestrator, inline)

1. **Locate the paper source.** Prefer editable source (.tex, .md, .Rmd, .qmd) over
   PDF — line references matter. If several candidates exist, ask the user which is
   the manuscript of record. Note approximate length.
2. **Locate the code.** Identify the active code directory, distinguishing active
   scripts from archived/legacy/superseded ones. Note the language(s).
3. **Locate outputs.** Results directories (CSVs, RDS, figures) are first-class
   evidence: a claim about a number in a table is verified against the output file
   the pipeline produced, not just against the script.
4. **Decide the claim groups** for the fan-out in Step 3 (typically: methods/API,
   each simulation study or experiment, empirical applications, tables-and-figures
   numbers, appendices). Aim for 4–8 groups of roughly balanced verification effort.

## Step 2 — Claims ledger (extraction agents)

Spawn one or two agents (split a long paper, e.g., main text vs. appendices) to read
the full manuscript and produce a **claims ledger**. Instruct them:

- Extract every claim a skeptical replicator could check against code or outputs.
  Claim types, with the paper's exact wording or value:
  - **EQ** — equations and algorithm descriptions (the math the code must implement)
  - **PARAM** — parameter values, tuning constants, defaults, thresholds
  - **BUDGET** — simulation sizes: MC reps, bootstrap draws, grid cells, seeds
  - **DATA** — sample sizes, variable definitions, data transformations, filters
  - **NUM** — specific numbers reported in text, tables, or captions
  - **SOFT** — software statements ("implemented in package X", "we use estimator Y")
  - **PROC** — procedural claims ("we drop replicates where...", "CIs are percentile")
- For each claim record: `ID | paper location (section + label/line) | claim text |
  type | expected evidence (which script/output should back it)`. The expected-evidence
  column is a guess — the verifiers will correct it.
- Prioritize: not every sentence is a claim. A 50-page paper typically yields 60–150
  checkable claims. Skip pure literature discussion; keep anything with a number, a
  formula, or a description of what the code does.
- Save the ledger as a markdown file in the working directory the orchestrator names.

## Step 3 — Verification fan-out (one agent per claim group)

Spawn the verification agents **in parallel, in a single message**. Each agent gets:
its claim group from the ledger, the paper source path, the code and output paths, and
these instructions:

- For each claim, find the code and/or output file that backs it. Read the actual
  implementation — do not trust script headers, comments, or filenames, which routinely
  go stale. Verify the substance:
  - EQ claims: line up the equation's terms against the code's math. A missing term,
    a swapped subscript, or a different normalization is a finding.
  - PARAM/BUDGET claims: read the literal values in the script. If the script was run
    with different values than it now contains (check logs/output metadata if present),
    say so.
  - NUM claims: open the output file (CSV/RDS/log) and compare the number the paper
    reports. Small rounding is fine; anything past the reported precision is a finding.
  - PROC claims: confirm the code actually does the procedure (e.g., if the paper says
    percentile CIs, find the quantile call — not a normal-approximation formula).
- Verdicts: **VERIFIED** (evidence found, matches; cite file:line or output cell),
  **MISMATCH** (evidence found, contradicts the paper; quote both sides),
  **PARTIAL** (implemented but deviates in a way the paper doesn't disclose),
  **NOT FOUND** (no producer in the active code — flag especially if outputs exist
  on disk but nothing regenerates them), **UNVERIFIABLE** (would require re-running
  long computations; say what a rerun would check).
- Never re-run expensive computations. Quick checks (loading a CSV, a sub-second
  script) are fine if a safe interpreter is available.
- Evidence discipline: every verdict cites its evidence — `file:line` for code,
  file + row/column for outputs. A verdict without a citation is worthless.
- Return the completed group table plus a short list of anything noticed outside the
  assigned claims (e.g., a bug in shared code) — flagged, not investigated.

**Cross-flag propagation (orchestrator):** when one verifier uncovers a cross-cutting
defect — a bug in a shared helper, a seeding/RNG pattern repeated across scripts, a
stale shared data file — immediately forward it to the still-running verifiers whose
scope contains the same pattern, with a concrete check to run (e.g., "count unique
result rows in your raw outputs"). One agent's local finding is often a project-wide
defect; the fan-out only catches that if the orchestrator relays it.

## Step 4 — Adversarial spot-check (optional but recommended)

Pick the 3–5 most load-bearing VERIFIED claims — the ones the paper's headline result
depends on — and spawn one skeptic agent per claim with the instruction: *"Try to
refute this verdict. Assume the first verifier was too generous."* Independent
re-derivation of the same evidence catches confirmation drift. Downgrade any claim the
skeptic breaks.

Skip this step only if the user asked for a quick pass or the paper is short.

## Step 5 — Synthesis and report

Write the report yourself (do not delegate synthesis). Save it to the project's review
folder if one exists, else the project root, named
`paper_code_check_<YYYY-MM-DD>.md`. Structure:

```markdown
# Paper–Code Cross-Check: <paper title>
*Date | paper source | code dir | N claims checked | verdict counts*

## Verdict Summary
One paragraph: is the paper backed by the code? Lead with the worst finding.

## Discrepancies (severity-ranked)
### CRITICAL — paper says X, code does Y (affects results or conclusions)
### MAJOR — claim has no producer in active code / output not regenerable
### MINOR — undisclosed deviations, stale values, precision issues
Each finding: claim ID, paper location, code evidence, why it matters, suggested fix.

## Traceability Matrix
| ID | Claim (short) | Paper loc | Code/output evidence | Verdict |
Every claim must be accounted for. If the verifiers wrote per-group tables to disk,
link those files here with per-group verdict counts instead of duplicating ~200 rows;
the group tables ARE the matrix, and the report stays readable.

## Unverifiable Claims
What a full rerun would be needed to check, and roughly what it would cost.

## Notes Outside Scope
Bugs or issues verifiers noticed that aren't paper claims.
```

Severity calibration: a MISMATCH on a headline number is CRITICAL even if small; a
MISMATCH on an appendix robustness parameter is usually MINOR. Judge by what a referee
or replicator would conclude if they found it themselves.

In your final message to the user: lead with the verdict summary and the CRITICAL/MAJOR
findings in plain prose, then link the report. Do not paste the full matrix into chat.

## Calibration notes

- **False alarms are costly.** Before reporting a MISMATCH, check whether the paper
  discloses the deviation elsewhere (a footnote, an appendix, a table note). Verify
  against the manuscript of record, not your memory of it.
- **Outputs vs. code.** "The script would produce X if run today" and "the number in
  the paper came from this output file" are different verifications. When they diverge
  (script edited after the run), report it — that is exactly the stale-pipeline failure
  this skill exists to catch.
- **Version drift.** Papers evolve. If the code matches an obviously earlier version of
  a claim (old parameter value, renamed method), report as MISMATCH with the likely
  history — that helps the author fix the right side.
