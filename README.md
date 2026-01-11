```markdown
# Ralph

Ralph is an autonomous AI agent loop that runs **Amp** on your codebase until every item in your Product Requirements Document (PRD) is complete [web:28][web:36]. Each iteration launches a fresh Amp instance with a clean context, while long-term memory is stored in your git history, `progress.txt`, and `prd.json` [web:33][web:36]. This pattern is inspired by Geoffrey Huntley’s Ralph workflow for continuous, agent-driven software development [web:32][web:33][web:36].

---

## Why Ralph?

Traditional coding agents are powerful for one-off tasks but struggle with:
- Large features that exceed a single context window
- Maintaining consistency across many small changes
- Capturing and reusing “tribal knowledge” about a codebase

Ralph turns Amp into a **persistent teammate** that:
- Breaks work into small, shippable stories
- Iterates until all stories pass their acceptance criteria
- Persists learnings so future runs (and humans) get smarter over time [web:33][web:36].

---

## How It Works (High Level)

Ralph runs in a loop:

1. **Reads the PRD** from `prd.json`  
2. **Chooses the highest‑priority story** where `passes: false`  
3. **Spawns a fresh Amp instance** with `prompt.md` as the system instructions  
4. **Implements exactly one story** (code changes, tests, docs)  
5. **Runs quality checks** (typecheck, tests, and any custom commands)  
6. **Commits if green** and updates `prd.json` to `passes: true`  
7. **Appends learnings** to `progress.txt` and updates `AGENTS.md`  
8. Repeats until all stories pass or `max_iterations` is reached [web:33][web:36].

This loop is orchestrated by `ralph.sh`, a simple Bash script that acts as the control plane for your agents [web:36].

---

## Key Concepts

### Fresh Context, Persistent Memory

Every iteration is a brand-new Amp process with a clean context window. The only cross-iteration memory comes from:
- **Git history** – the ground truth of code changes
- **`prd.json`** – which stories exist and which are done
- **`progress.txt`** – append-only log of what each iteration learned
- **`AGENTS.md`** – durable hints and patterns that Amp reads on future runs [web:33][web:36].

This keeps each run focused and prevents context bloat, while still accumulating durable knowledge over time.

### Right-Sized Stories

Ralph works best when each PRD item fits into a single agent context. Stories should be narrow, such as:
- “Add a new column and migration”
- “Add a filter dropdown to the users list”
- “Update the billing API to include field X”

Broad tasks like “build the entire dashboard” or “implement authentication” should be split into smaller stories before running Ralph [web:33][web:36].

### Feedback Loops

Ralph assumes your project has strong feedback signals:
- Typechecking for fast failure
- Tests for behavioral correctness
- CI that must stay green

These loops prevent broken changes from compounding over many autonomous iterations [web:35][web:36].

---

## Files and Layout

| File / Folder       | Purpose                                                                 |
|---------------------|-------------------------------------------------------------------------|
| `ralph.sh`          | Main loop script that orchestrates Amp iterations                      |
| `prompt.md`         | System prompt given to each Amp instance                               |
| `prd.json`          | Machine-readable PRD with `userStories` and `passes` flags             |
| `prd.json.example`  | Sample PRD to copy and adapt                                           |
| `progress.txt`      | Append-only log of iteration learnings and outcomes                    |
| `skills/prd/`       | Amp skill for generating PRDs from natural language                    |
| `skills/ralph/`     | Amp skill for converting PRDs to `prd.json`                            |
| `flowchart/`        | Interactive visualization of the Ralph loop and data flow             | [web:33][web:36]

---

## Prerequisites

- **Amp CLI** installed and authenticated (`@sourcegraph/amp`) [web:34].
- **`jq`** installed for JSON processing.
- A **git repository** for your project (Ralph relies heavily on git history).
- Optional but recommended:
  - CI pipeline that runs tests and typechecks
  - `AGENTS.md` files in relevant directories for agent-readable context [web:34][web:36].

---

## Setup

### Option 1: Add Ralph to a Single Project

From your project root:

```bash
mkdir -p scripts/ralph
cp /path/to/ralph/ralph.sh scripts/ralph/
cp /path/to/ralph/prompt.md scripts/ralph/
chmod +x scripts/ralph/ralph.sh
```

Place `prd.json` and `progress.txt` at the project root (or wherever your `ralph.sh` expects them).

### Option 2: Install Skills Globally

To make the PRD and Ralph skills available across all Amp projects:

```bash
cp -r skills/prd ~/.config/amp/skills/
cp -r skills/ralph ~/.config/amp/skills/
```

### Recommended: Amp Auto-Handoff

Enable automatic handoff in `~/.config/amp/settings.json`:

```json
{
  "amp.experimental.autoHandoff": { "context": 90 }
}
```

This lets Amp chain itself when context fills up, which is useful for larger stories [web:30][web:34].

---

## Workflow

### 1. Create a PRD

Use the PRD skill inside Amp:

> Load the prd skill and create a PRD for \[your feature description\]

Answer clarifying questions. The skill writes a markdown PRD to `tasks/prd-[feature-name].md` [web:33][web:36].

### 2. Convert the PRD to JSON

Turn the markdown PRD into `prd.json`:

> Load the ralph skill and convert `tasks/prd-[feature-name].md` to `prd.json`

This produces machine-readable user stories with `id`, `title`, `priority`, and `passes` fields [web:33][web:36].

### 3. Run Ralph

```bash
./scripts/ralph/ralph.sh [max_iterations]
```

- Default: 10 iterations
- Ralph will branch, implement, test, commit, and update `prd.json` and `progress.txt` on each pass [web:33][web:36].

When every story has `passes: true`, Ralph prints:

```txt
<promise>COMPLETE</promise>
```

and exits [web:33][web:36].

---

## Flowchart

Ralph includes an interactive flowchart that shows the full loop:

- **Online**: [Interactive Flowchart](https://snarktank.github.io/ralph/)  
- **Local**:

```bash
cd flowchart
npm install
npm run dev
```

This visualization walks through each step with animations and is useful when onboarding teammates or explaining Ralph to stakeholders [web:36].

---

## Best Practices

### Keep Stories Small and Independent

- Aim for tasks that fit comfortably inside one context window.
- Prefer vertical slices that go “end-to-end” for one behavior over horizontal “do everything in the backend” spans [web:33][web:35].

### Invest in AGENTS.md

After each iteration, Ralph appends learnings to the relevant `AGENTS.md` files. Amp reads these automatically, so:
- Document patterns (“use X library for Y”)
- Call out gotchas (“remember to update Z when changing W”)
- Capture domain context (“billing lives in `apps/billing` and uses feature flags”) [web:33][web:36].

### Maintain Strong Feedback Loops

- Keep tests fast and reliable.
- Fail early on type errors and lint issues.
- Treat a red CI build as a blocker before running more iterations [web:35][web:36].

### Browser Verification for UI Work

For frontend stories, include acceptance criteria like:

> Verify in browser using dev-browser skill.

Ralph will then use Amp’s browser tools to open the app, navigate, and confirm behavior for UI changes [web:30][web:33].

---

## Debugging & Introspection

Quick ways to inspect Ralph’s current state:

```bash
# Which stories are done?
cat prd.json | jq '.userStories[] | {id, title, passes}'

# What has Ralph learned so far?
cat progress.txt

# Recent commits from Ralph
git log --oneline -10
```

If Ralph is “stuck”:
- Check for failing tests or type errors.
- Confirm that stories are small enough.
- Tighten or clarify acceptance criteria in `prd.json` [web:33][web:36].

---

## Customization

You can adapt Ralph to your stack by editing `prompt.md`:

- Add project-specific quality gates (e.g., `pnpm lint`, `cargo test`).
- Specify directory structure, naming conventions, and patterns.
- Include links or short summaries of critical docs (design system, domain models, etc.) [web:33][web:36].

You can also extend `ralph.sh` to:
- Run different commands per-language or per-package.
- Integrate with your CI or deployment scripts.
- Archive logs or artifacts for observability.

---

## Archiving Runs

When a new feature (with a different `branchName`) starts, Ralph automatically archives previous runs into:

```txt
archive/YYYY-MM-DD-feature-name/
```

This keeps `prd.json` and `progress.txt` focused on the current feature while preserving past runs for audit and analysis [web:33][web:36].

---

## References

- [Geoffrey Huntley’s Ralph pattern](https://ghuntley.com/ralph/) [web:32]  
- [Amp CLI manual](https://ampcode.com/manual) [web:30][web:34]  
- [Original Ralph repo](https://github.com/snarktank/ralph) [web:36]  
- Background: autonomous coding agents and continuous development loops [web:28][web:35].
```

[1](https://ampcode.com)
[2](https://sourcegraph.com/amp)
[3](https://www.siddharthbharath.com/amp-code-guide/)
[4](https://ampcode.com/manual)
[5](https://zoltanbourne.substack.com/p/early-preview-of-amp-the-new-ai-coding)
[6](https://ghuntley.com/ralph/)
[7](https://docsmith.aigne.io/discuss/docs/ralph/en)
[8](https://www.npmjs.com/package/@sourcegraph/amp)
[9](https://ai-daily.news/articles/ralph-the-autonomous-coding-agent-that-never-stops)
[10](https://github.com/snarktank/ralph)
