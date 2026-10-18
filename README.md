# Three Claude Code skills

The three skills from my TEXT Talk *Field Notes from Six Months of Claude Code*
(Center for Contemporary Cultures of Text, Aarhus University, 8 October 2026;
[event page](https://arts.au.dk/en/text/events/show/artikel/text-talk-with-jesper-wulff)).
One reviews a paper before submission, one gives a session its memory and its rules,
and one checks a paper against the code behind it.

A skill is a text file. [Claude Code](https://claude.com/claude-code) reads it when you
type `/<name>` in a session, and the file tells the model what to do, step by step, in
prose. Read a skill before you run it: it is short, and it is the whole thing.

| Skill | What it does | Written by | Where it goes |
|---|---|---|---|
| `review-paper` | Six critics write a referee report on a LaTeX paper, before you submit. | [Claes Bäckman](https://github.com/claesbackman/AI-research-feedback); one sentence changed here | `~/.claude/commands/review-paper.md` |
| `session-init` | Reads the project's `CLAUDE.md` first, rewrites `README.md`, seeds four rules, keeps a dated session log. | me, inspired by Scott Cunningham | `~/.claude/skills/session-init/SKILL.md` |
| `paper-code-check` | Lists every checkable claim in a paper, traces each to the code and its output, gives a verdict with file and line, then sends in a sceptic. | me, inspired by Claes Bäckman | `~/.claude/skills/paper-code-check/SKILL.md` |

**Before you run any of them:** the review skills send your paper and your code to the
model. Check first what you may share: unpublished work, personal data.

## Install

`~/.claude` is a hidden folder in your user folder: `C:\Users\<you>\.claude` on Windows,
`/Users/<you>/.claude` on macOS. Skills are picked up when you start a new session in the
Claude desktop app (Code tab) or in the terminal.

**Without a terminal.** [Download this repository as a zip](https://github.com/jespernwulff/claude-code-skills/archive/refs/heads/main.zip)
and unzip it. Copy `commands/review-paper.md` into `~/.claude/commands/`, and copy the two
folders inside `skills/` (`session-init`, `paper-code-check`) into `~/.claude/skills/`.
Create the `commands` and `skills` folders if they do not exist.

**macOS, Linux, WSL (bash).** All three at once:

```bash
base=https://raw.githubusercontent.com/jespernwulff/claude-code-skills/main
mkdir -p ~/.claude/commands ~/.claude/skills/session-init ~/.claude/skills/paper-code-check
curl -o ~/.claude/commands/review-paper.md          $base/commands/review-paper.md
curl -o ~/.claude/skills/session-init/SKILL.md      $base/skills/session-init/SKILL.md
curl -o ~/.claude/skills/paper-code-check/SKILL.md  $base/skills/paper-code-check/SKILL.md
```

**Windows (PowerShell).** All three at once:

```powershell
$base = "https://raw.githubusercontent.com/jespernwulff/claude-code-skills/main"
New-Item -ItemType Directory -Force "$HOME\.claude\commands", "$HOME\.claude\skills\session-init", "$HOME\.claude\skills\paper-code-check" | Out-Null
Invoke-WebRequest "$base/commands/review-paper.md"          -OutFile "$HOME\.claude\commands\review-paper.md"
Invoke-WebRequest "$base/skills/session-init/SKILL.md"      -OutFile "$HOME\.claude\skills\session-init\SKILL.md"
Invoke-WebRequest "$base/skills/paper-code-check/SKILL.md"  -OutFile "$HOME\.claude\skills\paper-code-check\SKILL.md"
```

Run the same lines again to update. To install one skill, keep only its lines.

## The skills

### `review-paper` — a referee report before you submit

Written by [Claes Bäckman](https://github.com/claesbackman/AI-research-feedback). Six
critics read the paper's own `.tex` files in parallel, each with one job: spelling and
style; internal consistency (text against tables and abstract); unsupported claims;
equations and notation; tables and figures; and a demanding associate editor who judges
the contribution and gives the recommendation. The report consolidates the six and
copies the recommendation rather than paraphrasing it.

**Usage**

```text
/review-paper
/review-paper QJE
/review-paper JF path/to/main.tex
```

The first token can be a journal (`AER`, `QJE`, `JPE`, `Econometrica`, `REStud`, `JF`,
`JFE`, `RFS`, `JFQA`, `AEJMacro`, `JME`, `RED`); the file is auto-detected if omitted.
The list of journals is a list in the file; add your own.

**Output.** `PRE_SUBMISSION_REVIEW_<YYYY-MM-DD>.md` in the current folder, with `-v2`,
`-v3` appended if the file exists.

**Provenance.** This is Claes Bäckman's six-critic version of April 2026, as installed from
his repository on 4 April 2026, with one sentence changed on 1 October 2026 (the critic
of tables and figures, line 271). Before:

> **Important**: Figure files (PDF, PNG, EPS, JPG) cannot be read directly. Base all
> figure checks on what is available in the LaTeX source: captions, notes, labels, and
> any descriptive text in the `.tex` files. [...]

After:

> **Important**: Open every figure file you can (PNG, JPG, PDF) with the Read tool and
> look at it; EPS files cannot be displayed. Judge each figure from what you see together
> with the LaTeX source: captions, notes, labels, and any descriptive text in the `.tex`
> files. [...]

The reason: the tool can read figure files, and in practice the critics were already
looking at them. His current version has eight critics and more; it is at
[claesbackman/AI-research-feedback](https://github.com/claesbackman/AI-research-feedback).
If you install both, you will have two `/review-paper` commands; keep one.

### `session-init` — memory and rules for a session

Every session starts from zero. This skill makes three files hold the memory:

1. **Read `CLAUDE.md` first.** It holds the project's hard rules, which override defaults.
2. **Rewrite `README.md`** as a snapshot of the folder: tree, file counts, key files,
   notes for the next visitor.
3. **Seed `CLAUDE.md`** if it is missing, with four rules: never delete data, never
   delete a program, reorganise by copying, never leave the project folder. Both
   "never delete" rules point to a `legacy/` folder instead.
4. **Before the session ends,** prepend a dated entry to the session log in `CLAUDE.md`
   (what changed, what is next). The log is capped at eight entries; older ones roll
   into `SESSIONLOG.md`.

New rules the user states mid-session ("from now on, X") are appended to the hard rules
at once, with a line on why.

**Usage.** It triggers at the start of a session, when you say "init" or "let's start",
or when the model enters a project that already has a `CLAUDE.md`. You can also type:

```text
/session-init
```

**Honest note.** The rules are instructions, not locks. Nothing in the software enforces
them; the model enforces them on itself, and the cap of eight entries has been known to
be ignored.

**Provenance.** My own text, inspired by Scott Cunningham.

### `paper-code-check` — does the code do what the paper says?

The errors that matter do not crash: the paper says 2,000 simulation runs, the script
says 500, and the table looks fine. This skill starts from the paper's claims, not from
the code.

1. **Inventory.** Find the paper source (`.tex` preferred, for line numbers), the active
   code, and the output files.
2. **Claims ledger.** One or two agents list every checkable claim: equations,
   parameters, simulation budgets, data definitions, reported numbers, software
   statements, procedures.
3. **Verification fan-out.** One agent per group of claims traces each claim to the code
   and the output file that backs it. Verdicts: `VERIFIED`, `MISMATCH`, `PARTIAL`,
   `NOT FOUND`, `UNVERIFIABLE`. Every verdict cites file and line, or file and cell;
   a verdict without a citation is worthless.
4. **Adversarial spot-check.** For the claims the headline result depends on, a sceptic
   is told: assume the first verifier was too generous.
5. **Report.** `paper_code_check_<YYYY-MM-DD>.md`: verdict summary, discrepancies ranked
   by severity, traceability matrix, what a full rerun would be needed to check.

It is read-only: it never modifies the paper, the code or an output file, and it never
re-runs expensive computations.

**Usage**

```text
/paper-code-check
```

or ask in plain words: "does the code do what the paper says?", "check my numbers before
submission".

**Provenance.** My own text, written in July 2026, inspired by Claes Bäckman's
[`review-paper-code`](https://github.com/claesbackman/AI-research-feedback) of April
2026, which maps a paper's main empirical claims to its code and reviews the code's
reproducibility. The two files share no text; this one starts from a complete ledger of
claims, gives five verdicts with evidence, and ends with a sceptic.

## Requirements

- [Claude Code](https://claude.com/claude-code), in the desktop app (Code tab) or the
  terminal, with subagents available. `review-paper` and `paper-code-check` launch
  several agents in parallel; `session-init` runs in the main session.
- A LaTeX paper for `review-paper`. `paper-code-check` reads any code it can open.

## Licence

MIT (see `LICENSE`) for `session-init`, `paper-code-check` and this README.
`commands/review-paper.md` is copyright Claes Bäckman, also MIT; his notice is in
`commands/LICENSE`.
