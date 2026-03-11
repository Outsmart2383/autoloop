---
name: autoloop
description: >
  Autonomous optimisation loop inspired by Karpathy's autoresearch. Run iterative
  experiments that improve any measurable metric — email copy, API performance,
  prompts, configs, code, or anything else with a benchmark that outputs a number.
  Use this skill when the user says things like "optimise", "autoloop", "run experiments",
  "iterate overnight", "autoresearch", "optimisation loop", "improve my [X] automatically",
  "run an experiment loop", or wants to set up autonomous agent-driven iteration on any file
  or project.
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Agent, AskUserQuestion
---

# AutoLoop — Autonomous Optimisation Loop

> Adapts Andrej Karpathy's **autoresearch** pattern for non-ML continuous
> improvement. Same core mechanics: bounded search space, single metric,
> git ratchet, commit-before-evaluate, never-stop autonomy, simplicity bias.

AutoLoop brings the autoresearch pattern to any project. The agent edits a target
file, commits, runs a benchmark, and keeps changes that improve a metric. Changes
that don't improve get reset. The loop runs autonomously until you say stop.

The core idea: **the human is the architect, the agent is the operator.**

| | Human (architect) | Agent (operator) |
|---|---|---|
| **Decides** | WHAT to optimise and WHY | HOW to optimise |
| **Controls** | Strategy, constraints, stopping conditions | Hypotheses, edits, evaluation |
| **Interface** | Writes and revises `program.md` | Reads `program.md`, edits target file |
| **Approval** | Sets rules upfront | Runs autonomously — no per-step approval |

### Three-file model

1. **`program.md`** — strategy document. Human-written, read-only for the agent.
   This is the human's sole interface to the loop — as Karpathy says, "the human
   iterates on the prompt (.md)." All strategy, constraints, and context live here.
   The agent reads it but never modifies it. It's a living document — the human
   should revise it after seeing initial results.
2. **Target file** — the ONLY file the agent may edit.
3. **Benchmark script** — immutable evaluation. The agent runs it but MUST NEVER
   modify it. It is a read-only black box.

### Evaluation invariants

- **One metric, binary decision.** Exactly one metric, strict binary keep/discard.
  Did the metric improve? Keep. Otherwise, revert. No multi-objective weighting,
  no subjective overrides.
- **Fair comparison.** Every experiment runs the identical benchmark command under
  identical conditions — same environment, same dependencies, same resources. The
  only variable between experiments is the content of the target file. The agent
  must not alter the evaluation environment, add caching, change dependencies,
  or modify anything outside the target file between experiments. No caching,
  no shortcuts, no environment changes — the only variable is the target file.
- **Bounded search space.** One file only. No new files, no new dependencies,
  no changes outside the target file. This constraint is essential for two reasons:
  (1) **Tractability** — the agent must read and understand the entire target file
  each iteration; a single file keeps this feasible within LLM context windows.
  (2) **Clean attribution** — every metric change maps to a change in exactly one
  file, making it easy to understand what worked and why.

---

## Setup Phase

When the user invokes this skill, **walk them through an interactive wizard**
one question at a time. Use the `AskUserQuestion` tool for each question.

**The wizard is critical.** Each question is its own turn. Do NOT batch multiple
questions into one message. Do NOT skip ahead. Do NOT assume defaults. Wait for
each answer before proceeding to the next question.

### Step 1: Collect inputs

Run this as a strict step-by-step wizard. For each question below, use the
`AskUserQuestion` tool to ask it, wait for the user's response, acknowledge
their answer, then proceed to the next question. Never ask more than one
question per turn.

**Wizard flow:**

1. Ask question 1 → wait for answer → acknowledge
2. Ask question 2 → wait for answer → acknowledge
3. Ask question 3 → wait for answer → acknowledge
4. If they need a benchmark, guide them through the Benchmark Interview (Step 2)
   — this is the most important step. The benchmark defines what "better" means.
5. Ask question 4 → wait for answer → acknowledge
6. Ask question 5 → wait for answer → acknowledge
7. Ask question 6 (only if benchmark uses LLM) → wait for answer → acknowledge
8. Ask question 7 → wait for answer → acknowledge
9. Run the Program.md deep-dive interview (question 8a-8g) — multiple turns,
   one sub-question at a time. This is the second most important step after the
   benchmark interview. The quality of program.md determines the quality of experiments.
10. Display full config summary → ask for confirmation

If the user volunteers information upfront (e.g., they describe everything in
their initial message), don't re-ask what they've already told you — acknowledge
it, skip those questions, and continue with the remaining ones. But still ask
about anything they didn't cover, one question at a time.

**Question phrasing:** Keep each question conversational and helpful. Include
examples and suggestions where noted below. If the user seems unsure, offer
guidance before moving on.

**1. Project directory**
Where is the project? Ask for the full path.

**2. Target file**
Which single file should the agent edit to try improvements? This is the ONLY file
the loop will modify. Everything else stays fixed.

Ask the user: "What file contains the thing you want to improve?" If they're unsure,
help them identify it:
- For copy/content optimisation: the file containing the text (JSON, markdown, txt)
- For performance optimisation: the code file handling the slow path (route, query, config)
- For prompt optimisation: the prompt template file
- For config tuning: the configuration file

**If the user wants to optimise multiple files:** Explain that AutoLoop works on one
file at a time for clean attribution (so you know which change caused each improvement).
Suggest either picking the highest-impact file first, or combining related content into
a single file if that makes sense for their project.

**If the target file is very large (>500 lines):** Warn the user that the agent needs to
read and understand the full file each iteration. Very large files slow down hypothesis
generation and increase the chance of unintended side effects. Suggest focusing on a
specific section or extracting the part to optimise into a smaller file.

**3. Benchmark command**
What command measures success? It must:
- Be runnable from the project directory (the agent will `cd` there first)
- Print a line containing the metric name and a number to stdout (output is
  redirected to run.log). Exit code 0 on success, non-zero on failure
- Complete within a reasonable time (under 5 minutes per run recommended)

**If the user does not have a benchmark yet**, help them create one. See the
"Benchmark Creation Guide" section below. This is common — most users will need help
here. Do not skip this step or start the loop without a working benchmark.

**Fair comparison — identical conditions every run:** Every experiment MUST be
evaluated under identical conditions to ensure fair comparison. This means:
- The same benchmark command runs every time — no modifications between experiments
- The same environment, dependencies, and configuration for each run
- No experiment may change how long the benchmark runs or what resources it uses
- The agent must not add caching, memoisation, or shortcuts that make the benchmark
  faster without genuinely improving the target metric

**Fixed time budget (contextual):** In autoresearch, a fixed 300-second training
budget ensures fair comparison — every experiment gets the same compute. Apply the
same principle here: if the benchmark is compute-bound (ML training, simulation,
heavy computation) where the agent might change parameters that affect runtime
(model size, batch size, iterations), require a fixed time budget. This ensures
experiments are directly comparable: "what's the best result in N minutes?" rather
than letting a slower approach win just because it ran longer. Wrap the command with `timeout`
and note the budget in program.md. For benchmarks that measure a fixed operation
(API calls, test suites, LLM scoring), a time budget is unnecessary — skip this.

**4. Metric name**
What's the metric called in the benchmark output? The agent will parse stdout for it.
Examples: `score`, `load_time_ms`, `accuracy`, `pass_rate`, `requests_per_sec`

**5. Direction**
Is higher better, or lower better?
- Higher is better: scores, pass rates, quality ratings, throughput
- Lower is better: latency, error rates, loss values, file sizes

**6. Model selection**
AutoLoop uses models for two distinct roles — offer the user a choice for each:

- **Agent model** — drives the loop: generates hypotheses, analyses results, and
  edits the target file. A stronger model produces more diverse and higher-quality
  experiments. In Claude Code this is the model running the session.
- **Scorer model** — runs the LLM-as-judge benchmark (if applicable), invoked
  via `claude -p --model {model}`. The right choice depends on what's being scored:
  - **Subjective/qualitative criteria** (persuasiveness, tone, clarity, brand
    voice, creativity) — a stronger model scores more consistently and catches
    nuances that cheaper models miss. Recommend Sonnet or above.
  - **Objective/mechanical criteria** (syntax valid, word count, contains
    required elements, passes tests) — a cheaper model is fine since the
    judgement is straightforward. Haiku works well here.

  Explain this trade-off to the user and let them choose. The cost difference
  per experiment is typically small (pennies), but it compounds over many runs.

Common pairings:
- **Quality-focused:** Opus as agent, Sonnet for scoring
- **Balanced (subjective scoring):** Sonnet as agent, Sonnet for scoring
- **Balanced (objective scoring):** Sonnet as agent, Haiku for scoring
- **Budget:** Haiku as agent, Haiku for scoring (fast but less creative)

If the user doesn't care, default to the current session model as agent.
For the scorer, recommend based on the scoring criteria: Sonnet for subjective,
Haiku for objective.

For benchmarks that don't use an LLM (patterns A and C), only the agent model
matters — skip the scorer model question.

**7. Stopping behaviour**
Ask the user explicitly: "Should the loop run until you say stop, or do you want
a fixed experiment limit?"

- **Default is unlimited** — the loop runs until the user says "stop". This is
  the intended behaviour and should be clearly presented as the recommended
  option. Frame it positively: "The loop will keep running and improving
  autonomously — you can check in any time and say stop when you're happy."
- If the user specifically asks for a limit, accept it (suggest 30-50 for
  overnight, 10-15 for a quick test).
- Optionally ask if there's a target score to stop at (e.g., "stop if you hit
  9.0"). Most users should skip this.

**Do NOT default to a fixed limit.** Do NOT invent a cap unless the user
explicitly requests one.

**8. Program.md interview — deep elicitation**

program.md is the human's primary interface to the loop — Karpathy says "the human
iterates on the prompt (.md)." A shallow program.md produces shallow experiments.
This interview elicits the deep context that makes program.md genuinely useful.

**Do NOT just ask "describe your project."** Use the structured flow below. Each
sub-question is its own turn using `AskUserQuestion`.

**Adapt to expertise level.** Gauge from early answers whether the user is a beginner
or expert. For beginners: use simpler language, offer more examples, skip Goodhart's
law warning (frame it instead as "things that could go wrong"). For experts: go
deeper on constraints, push harder on metric validity, ask about edge cases.

**Flow: Orient → Ground → Constrain → Direct → Validate**

**8a. Orient** — Use story-based questions, not abstract ones:
- "Tell me about the last time you tried to improve this. What did you try, and
  what happened?" (grounds in real experience)
- If first time: "What made you decide this needs improving? What's the pain point?"

**8b. Ground in specifics** — When users say vague things ("better quality"), ladder:
- "Can you show me an example of good vs. bad?"
- "Why does that matter?" (repeat — dig from surface to root goals)
- "If you had two versions, how would YOU decide which is better?"
- Push for ONE primary goal. If multiple: "When these conflict, which wins?"

**8c. Surface hidden constraints and warn about Goodhart's law**
- "What would make you reject a result even if the metric improved?"
- "Are there things about the current version you definitely want to keep?"
- "How could the agent game this metric without actually achieving what you want?"
  (e.g., an email scoring high on 'persuasiveness' by being manipulative)
- Use answers to add anti-gaming constraints to program.md.

**8d. Define direction and domain knowledge**
- Hard constraints: "What must the agent NEVER change or violate?"
- Directions: "Any approaches you'd like it to explore first?"
- Domain: "What does the agent need to know about your industry or audience?"

**8e. Validate understanding — reflect back before generating program.md**
Summarise your understanding back: "Here's what I'll put in program.md: [summary].
Does this capture what matters? Anything missing?" Revise until confirmed.

**8f. Set revision expectations**
- "program.md is a living document — revise it after the first 10-15 results.
  The first version is a starting point, not a final answer."

### Step 2: Benchmark creation (if needed)

**Note:** Since autoloop runs inside Claude Code, LLM-as-judge benchmarks use
the `claude` CLI which inherits the user's existing auth. No separate API key
setup is needed.

If the user doesn't have a benchmark, help them build one. Three common patterns:

**Pattern A: Script-based measurement** — Run the thing, output a number. Best for
latency, file sizes, pass rates, throughput.

```bash
#!/bin/bash
SECONDS_TOTAL=$(curl -s -o /dev/null -w "%{time_total}" http://localhost:3000)
MS=$(awk "BEGIN {printf \"%.0f\", $SECONDS_TOTAL * 1000}")
echo "response_time_ms: $MS"
```

**Pattern B: LLM-as-judge** — Use the `claude` CLI to score content against
criteria. Best for copy, documentation, prompts — anything where "better" is
subjective. The script builds a scoring prompt in a temp file, pipes it to
`claude -p`, and parses the JSON response.

Since autoloop runs inside Claude Code, use the `claude` CLI for scoring — it
inherits the user's existing auth (OAuth or API key). No separate API key setup
is needed. The benchmark script MUST include `unset CLAUDECODE` at the top to
allow nested `claude` CLI invocations from within a Claude Code session.

**Use the scorer model** (from Step 1, question 6) via the `--model` flag
(e.g., `claude -p --model haiku --no-session-persistence`).

The scoring prompt IS the benchmark — it defines what "better" means. This is the
most important artefact in the entire system. Treat benchmark creation as a
collaborative design process, not a form to fill in. Walk the user through it:

**Criteria discovery (for LLM-as-judge benchmarks):**

Start by understanding what the user actually cares about. Ask: "What does
'better' look like to you? Describe a great version of this versus a mediocre
one." Listen for implicit criteria they haven't articulated.

Then help them convert that into 3-6 specific, measurable, independent scoring
criteria. For each candidate criterion, validate it:
- **Specific enough?** "Good writing" is too vague. "Clear topic sentence in
  each paragraph" is scorable. Push the user to be concrete.
- **Independent?** Criteria that move together waste scoring budget. If
  "conciseness" and "brevity" are both listed, they'll always score the same
  — merge them. Each criterion should be able to improve while others stay flat.
- **Actually matters?** Ask: "If this criterion scored 10 but the result felt
  wrong, what would be missing?" That missing thing is a better criterion.
- **Measurable by an LLM?** The scorer needs to judge it from the text alone.
  "Converts well" requires real-world data. "Has a clear call to action" is
  scorable.

**Help users who don't know what to measure.** If they say "I just want it
better" or seem unsure, use diagnostic questions:
- "Who is the audience? What do they need from this?"
- "What's wrong with the current version? What bothers you about it?"
- "If you could only fix one thing, what would it be?"
- "Show me an example you admire — what makes it good?"

These questions surface implicit quality dimensions the user hasn't named yet.

**Warn about common pitfalls:**
- **Vague criteria** ("quality", "professionalism") — the scorer will interpret
  these inconsistently. Always push for specifics.
- **Correlated criteria** — if two criteria always move together, the benchmark
  double-counts that dimension. Merge or drop one.
- **Missing dimensions** — ask "Is there anything important about quality that
  none of these criteria capture?" before finalising.
- **Too many criteria** — more than 6 dilutes signal. Each criterion should
  earn its place.
- **Metrics that don't reflect real quality** — a high score should mean the
  output is genuinely better, not just technically compliant.

**Scenario validation before finalising:** Test the criteria against a concrete
scenario. Ask: "Imagine the agent produces a version that scores 10 on all these
criteria. Would that actually be the result you want? What could still be wrong?"
If the user identifies a gap, add a criterion for it.

**Confirm before finalising:** Show the user the complete list of criteria with
one-sentence descriptions. Ask: "If the agent optimised for exactly these
criteria and nothing else, would the result be what you want?" Revise until
they say yes.

**Pattern C: Test suite** — Run existing tests and extract pass rate. Best for code,
configs, schemas — anything with testable output. Adapt the grep patterns below to
match your test runner's output format (this example is for mocha/jest):

```bash
#!/bin/bash
OUTPUT=$(npm test 2>&1)
PASSED=$(echo "$OUTPUT" | grep -o '[0-9]* passing' | grep -o '[0-9]*')
FAILED=$(echo "$OUTPUT" | grep -o '[0-9]* failing' | grep -o '[0-9]*')
FAILED=${FAILED:-0}
TOTAL=$((PASSED + FAILED))
[ "$TOTAL" -eq 0 ] && echo "ERROR: no test results parsed — check grep patterns" && exit 1
RATE=$(awk "BEGIN {printf \"%.4f\", $PASSED / $TOTAL}")
echo "pass_rate: $RATE"
```

After creating the benchmark: save it in the project directory, make it executable
(`chmod +x`), and test it. Do NOT commit yet — it gets committed in Step 3.

### Step 3: Verify setup

Before starting the loop, run ALL of these checks in order:

1. `cd` to the project directory — confirm it exists
2. Confirm the target file exists and is readable
3. Confirm git is initialised (`git rev-parse --git-dir`). If not, ask the user
   if they want to run `git init` and make an initial commit of all project files
4. Confirm the working tree is clean (`git status --porcelain`), ignoring any
   benchmark file created in Step 2. If other files are dirty, ask the user to
   commit or stash changes first — the loop needs a clean starting point
5. Note the current branch. If the user is not on `main` or `master`, confirm they
   want the autoloop branch to fork from their current branch
6. **Dry-run the benchmark:** Run the command once. If it fails, help the user
   debug (common issues: missing dependency, service not running, permission
   denied, wrong file path)
7. Parse the output for the metric — confirm the value is a valid number
8. Run the benchmark a second time — check if the metric is stable (within 10%
   of the first run). If not, warn the user that noisy benchmarks cause false
   positives and suggest using averaged runs (see Variance Handling in Loop Phase)
9. Record the baseline metric value (average of the two verification runs)

Display a summary of all configuration and the baseline metric, then ask for
explicit confirmation before proceeding.

### Step 4: Create branch and program.md

Create the autoloop branch FIRST, then write and commit program.md to it.
This keeps the user's main branch clean.

```bash
git checkout -b autoloop/run-$(date +%Y%m%d-%H%M%S)
```

Then write a `program.md` in the project directory capturing the configuration:

```markdown
# AutoLoop Program

## Configuration
- **Target file:** `{target_file}`
- **Benchmark:** `{benchmark_command}`
- **Metric:** `{metric_name}` ({direction} is better)
- **Stopping:** {stopping_behaviour}
- **Baseline:** {baseline_value}
- **Agent model:** {agent_model} (hypotheses, analysis, and edits)
- **Scorer model:** {scorer_model or "N/A — benchmark doesn't use LLM"}
- **Time budget:** {time_budget or "none (benchmark runs to completion)"}
- **Variance handling:** {single_run or averaged}
- **Started:** {ISO_timestamp}

## Goal
{primary_goal — one sentence, specific, from laddering interview}

## Context
{user_provided_context — grounded in real experience from story-based elicitation}

## What "better" looks like
{concrete examples of good vs. bad from the interview, in the user's own words}

## Anti-gaming constraints
{from Goodhart's law discussion — how could the metric be gamed? What to avoid}

## Optimisation directions
{user_provided_directions or "Agent will explore autonomously"}

## Constraints
{user_provided_constraints — includes both explicit and surfaced implicit constraints}

## Revision notes
This is version 1 of program.md. Review and revise after the first 10-15 experiments
based on what the agent actually tries and what the results reveal.
```

Commit program.md (and benchmark if created in Step 2):

```bash
git add program.md && git commit -m "autoloop: initialise program"
git add {benchmark_file} && git commit -m "autoloop: add benchmark"  # if applicable
```

Create the experiment log and gitignore it (results.tsv is never committed):

```bash
printf "commit\tvalue\tstatus\tcost\tdescription\n" > results.tsv
echo "results.tsv" >> .gitignore
echo "run.log" >> .gitignore
git add .gitignore && git commit -m "autoloop: gitignore results.tsv and run.log"
```

results.tsv tracks every experiment — kept, discarded, and crashed. The git log
tracks only kept experiments (commits that advance the branch).

---

## Loop Phase

### 1. Record the baseline

Run the benchmark (using variance handling if noisy — see below), parse the metric,
and store it as `current_best`.

### 2. Enter the experiment loop

LOOP FOREVER (or until max_experiments if configured):

**Context management — minimise token usage by design:**
- Read `program.md` **once** at the start of the batch, not every iteration.
- Read the target file each iteration (you need the current state to edit it).
- Read only the **last 15 lines** of `results.tsv` each iteration (not the
  whole file). This tells you recent history without ballooning context.
- **Always redirect output to files** (`> run.log 2>&1`) and parse only the
  metric line — never let raw benchmark output flood the context window.
- Skip `git log` — `results.tsv` already tracks kept/discarded/crashed status.
- Sub-agent prompts include only config, instructions, and current best — the
  sub-agent reads history from `results.tsv` on disk, not from the prompt.

#### a) Form a hypothesis

Read the target file and the last 15 lines of results.tsv. Review what has
been tried recently and what worked.

**Track what you've tried.** Use results.tsv (status column) to see which
approaches were kept, discarded, or crashed. Never repeat an approach that
was already discarded. If a direction has failed 3+ times, move on. If 5+
consecutive experiments were discarded, try something structurally different
— change the approach entirely, not just parameters. Systematically vary across
these dimensions (cycle through them — don't get stuck on one):
- **Wording:** rephrase, clarify, tighten prose
- **Structure:** reorder sections, merge or split content, change hierarchy
- **Deletion:** remove paragraphs, sections, or features (less is often more)
- **Addition:** add missing concepts or strengthen weak areas
- **Reframing:** change the conceptual angle, emphasis, or mental model
- **Combination:** bundle multiple small ideas into one bold change

**Prefer simplicity.** Actively seek experiments that simplify the target file.
Removing something and getting equal or better results is a great outcome — it
proves the removed part was unnecessary.

**Be bold.** Small tweaks get lost in scorer variance. Prefer combined changes
that test a clear, substantial hypothesis. If a combined change fails, try the
individual ideas separately next time.

If you run out of ideas: think harder. Re-read the target file from scratch.
Look at it from a completely different angle. Try the opposite of what you've
been doing. Try removing things instead of adding. You are autonomous — do not
stop to ask for direction.

#### b) Edit the target file

Each experiment should test a clear hypothesis, describable in a sentence for the
log. Everything is fair game: small tweaks, large structural changes, architectural
rewrites. If a combined change fails, try the ideas separately next time.

#### c) Commit and benchmark

Before running the benchmark, do a quick sanity check on the target file. If it's
a structured format (JSON, YAML, TOML, XML, config), validate syntax first. If
invalid, revert immediately — log as "crash" in results.tsv.

Commit the change (every experiment gets a commit, even if it will be reset later):

```bash
git add "{target_file}"
git commit -m "autoloop #{N}: {description}"
```

Record the short commit hash for results.tsv.

Run the benchmark, redirecting output to a log file. Do NOT let benchmark output
flood your context window — always redirect and read only what you need:

```bash
cd {project_directory} && {benchmark_command} > run.log 2>&1
```

**Timeout protection:** Wrap the command:
`timeout 600 {benchmark_command} > run.log 2>&1`.

**Metric parsing:** Read the metric from run.log. Search for these patterns:
1. `{metric_name}: {number}`
2. `{metric_name}={number}`
3. `{metric_name} = {number}`
4. `"{metric_name}": {number}` (JSON-style)

Where `{number}` matches: integers (`42`), decimals (`3.14`), negative values (`-0.5`),
and scientific notation (`1.5e-3`). Strip trailing non-numeric characters (`%`, `ms`,
`s`, `MB`, etc.).

If multiple lines match, use the LAST one. If no match, treat as benchmark failure.

**Variance handling:** If setup showed >10% variance, run 3 times and use the median.

#### d) Evaluate the result

Compare the parsed metric to `current_best`:

**Cost tracking:** After each benchmark run, parse the cost line from the
benchmark output. If the benchmark doesn't report cost, log `-` in the cost column.

**Benchmark failed** (non-zero exit, no metric found, NaN, killed by timeout):
- Reset: `git reset --soft HEAD~1 && git checkout -- "{target_file}"`
- Log to results.tsv: `{hash}\tNA\tcrash\t{cost}\t{description}`

**Suspiciously large change** (>10x improvement or degradation):
- Run the benchmark a second time to check if reproducible
- If both runs agree (within 20%), use the second value
- If they disagree, treat as unreliable — reset the commit

**Metric improved** (strictly better according to configured direction):

The decision is strictly binary: did the metric improve? If yes, keep. If no,
discard. There is exactly ONE metric and exactly ONE decision rule. No
multi-objective weighting, no subjective judgment calls, no probabilistic
acceptance. The metric is the sole arbiter.

If the benchmark has known variance (>10% during setup), require the improvement
to exceed the observed variance to count as real. Otherwise, any strict
improvement = keep.

If the improvement is real:
1. Keep the commit (already committed in step c)
2. Log to results.tsv: `{hash}\t{new_value}\tkept\t{cost}\t{description}`
3. Update `current_best = new_value`
4. Tell the user: `Experiment {N} — {metric} {old} -> {new} ({delta}) — {description} [{hash}] (cost: {cost})`

**Metric unchanged or worsened:**
- Reset: `git reset --soft HEAD~1 && git checkout -- "{target_file}"`
- Log to results.tsv: `{hash}\t{new_value}\tdiscarded\t{cost}\t{description}`
- Batch notifications: send an update every 3-5 consecutive resets summarising
  what was tried and the current best.

#### e) Check stopping conditions and signals

**Check the signal file** before starting the next experiment:

```bash
if [ -f .autoloop-signal ]; then
  SIGNAL=$(cat .autoloop-signal)
  rm .autoloop-signal
fi
```

Handle signals:
- `stop` — finish the current experiment, then go to step 3 (summarise and end)
- `focus:{direction}` — prioritise this direction for upcoming experiments
- `avoid:{approach}` — blocklist this approach for the rest of the run
- `try:{suggestion}` — use this as the next hypothesis, then resume normal flow

**Stop the loop** ONLY if:
- Signal file contains `stop`
- Reached the batch experiment limit (the main session will spawn the next batch)
- Reached a user-configured total limit (experiment count or target score) — but ONLY
  if the user explicitly set one during setup. Never invent a stopping condition.

If 15+ consecutive experiments were discarded, note it in your hypothesis
reasoning and try something structurally different — but do NOT stop. If
improvements are shrinking, try bolder changes — but do NOT stop. You are
autonomous. The user might be away.

### 3. Finish and summarise

When the loop ends for any reason:

1. Send the final summary (use directional language — "76% faster" for lower-is-better,
   "39% higher" for higher-is-better — always phrased as a positive outcome):

```
AutoLoop complete — {total} experiments

Results
  Baseline:  {metric_name} = {baseline}
  Best:      {metric_name} = {best} ({improvement_pct}% {better_word})
  Kept:      {num_commits} improvements
  Discarded: {num_discarded}
  Failed:    {num_failed}
  Total cost: ${total_cost} (scorer: ${scorer_cost})

Top improvements (by impact)
  1. #{N} {hash} — {description} ({delta})
  2. #{N} {hash} — {description} ({delta})
  3. #{N} {hash} — {description} ({delta})

Full experiment log in results.tsv

Next steps:
  Review:  git log --oneline {branch}
  Inspect: git show {hash}
  Apply:   git checkout main && git merge {branch}
  Discard: git branch -D {branch}
```

If the user asks for help reviewing, guide them: `git diff main..{branch}` for the
overall change, `git show {hash}` for individual experiments, `git cherry-pick` to
apply specific improvements, or "resume" to start another run from the branch tip.

---

## Resuming a previous run

If the user asks to "resume", "continue", or "pick up where we left off":

1. List existing autoloop branches: `git branch --list 'autoloop/*'`
2. If multiple exist, ask the user which to resume (show dates from branch names)
3. Check out that branch
4. **Check for dirty working tree** (`git status --porcelain`). If the target file
   has uncommitted changes, the previous run likely crashed mid-experiment. Reset:
   `git reset --soft HEAD~1 && git checkout -- "{target_file}"`
5. Read program.md to restore all configuration
6. Determine the next experiment number by counting rows in results.tsv (this
   includes kept, discarded, and crashed experiments)
7. Run the benchmark on the current branch tip to establish `current_best`
8. Resume the loop from the next experiment number

---

## Mid-loop interaction

The loop runs in a background sub-agent, so the main session stays free. The user
can message the main session at any time. Handle commands by writing a **signal
file** that the sub-agent checks between experiments:

**Main session responsibilities** (when the user sends a command):

- **"stop"/"pause"/"finish"** → Write the signal file and tell the user:
  ```bash
  echo "stop" > {project_dir}/.autoloop-signal
  ```
  "Stop signal sent — the current experiment will finish, then the batch will end."

- **"status"** → Read `results.tsv` and `git log --oneline` directly from the
  project directory. Report current best, experiments run, and recent activity
  to the user. No need to contact the sub-agent.

- **"focus on {X}"** → `echo "focus:{X}" > {project_dir}/.autoloop-signal`
- **"avoid {X}"** → `echo "avoid:{X}" > {project_dir}/.autoloop-signal`
- **"try {X}"** → `echo "try:{X}" > {project_dir}/.autoloop-signal`

---

## Error handling and crash judgment

The agent must distinguish between trivial bugs and fundamental failures to
maintain experimental cadence:

- **Trivial bugs** (typo, syntax error, missing import in the target file): fix
  and re-run the benchmark. This doesn't count as a new experiment. Don't waste
  an experiment slot on a typo.
- **Fundamental failures** (OOM, impossible constraint, the hypothesis is
  structurally broken): revert immediately, log as "crash" in results.tsv, and
  move on to the next hypothesis. Don't get stuck trying to rescue a bad idea.
- **Git operation fails:** Stop immediately. Report the error. Never continue
  without clean version control.
- **System errors** (disk full, permissions, benchmark script itself broken): Stop
  and report. These are the ONLY reasons to pause and ask the user.

The key principle: **cadence over perfection.** Fix what's trivially fixable,
abandon what's fundamentally broken, and never let a single failed experiment
stall the loop. A fast iteration cycle with some crashes is better than a slow
cycle that tries to rescue every bad idea.

---

## Core principles

1. **One file only — the benchmark is immutable.** Never edit anything except the
   designated target file. The benchmark script, program.md, and all other project
   files are strictly off-limits during experiments. The agent MUST NOT modify,
   overwrite, replace, or recreate the benchmark under any circumstances. The
   benchmark is the immutable evaluation function — if the agent could change it,
   the entire optimisation loop would be meaningless. Treat the benchmark as a
   read-only black box: run it, parse its output, never touch it.

2. **Simpler is better.** All else being equal, prefer simplicity. Removing
   something and getting equal or better results is a great outcome — that's a
   simplification win. Use this decision matrix for complexity tradeoffs:
   - Metric improved + simpler → always keep
   - Metric improved + same complexity → keep
   - Metric improved + more complex → keep only if improvement is substantial
   - Metric unchanged + simpler → keep (simplification is an improvement)
   - Metric unchanged or worse + more complex → always discard

   Actively seek deletion experiments: if you can remove
   a section, paragraph, or feature and the metric holds or improves, that
   proves the removed part was unnecessary weight.

3. **Git ratchet — the branch only advances on improvements.** Every experiment
   is committed BEFORE evaluation. Kept experiments advance the branch.
   Discarded experiments get `git reset --soft HEAD~1`. The git log is the
   monotonically-improving record; results.tsv is the complete exploration
   record (kept + discarded + crashed). The branch IS the output — each
   commit on it represents a verified improvement over the previous state.

4. **Never stop voluntarily.** Once the loop begins, you are fully autonomous.
   The loop runs indefinitely without human approval. If you run out of ideas,
   think harder — re-read the file, try the opposite, try removing things.
   Only stop when explicitly told to, or when max_experiments is reached (if
   configured). Do NOT invent stopping conditions. Do NOT ask for guidance.

5. **Respect constraints absolutely.** If program.md says "don't do X", never do X.

6. **Pursue diversity — rotate across experiment dimensions.** Track what you've
   tried. Never repeat a discarded approach. Systematically vary across dimensions:
   wording, structure, deletion, addition, reframing, combination. If 5+
   consecutive experiments were discarded, try something structurally different
   — change the entire angle. The best experiments are often the unexpected ones.
   Be bold: try radical deletions, structural inversions, or complete rewrites.

7. **The user is in charge.** They can redirect, pause, or stop at any time.
   Respond to mid-loop messages promptly. The loop serves the user.

---

## Execution model

**Spawn a background sub-agent.** The autoloop MUST run as a sub-agent via the
Agent tool with `run_in_background: true`, not inline in the main session. This
keeps the main session free for other conversations and prevents the context
window from growing unboundedly with experiment data.

### Setup (main session)

Complete the full setup interview (Steps 1-4) in the main session — collect
inputs, create the benchmark, verify setup, create the branch and program.md.

### Launch (main session → background sub-agent)

Once everything is confirmed, spawn the loop as a background sub-agent using
the Agent tool:

```
Agent tool call:
  description: "autoloop: {project-name} batch 1"
  run_in_background: true
  prompt: <full batch instructions — see below>
```

The `prompt` must include everything the sub-agent needs to run autonomously:
- The project directory path
- The branch name (already created and checked out)
- All configuration from program.md (target file, benchmark command, metric,
  direction, constraints, scoring criteria)
- The full Loop Phase instructions from this skill document
- The signal file path (`.autoloop-signal` in the project directory)
- The batch experiment range (e.g., "Run experiments 1 to 15")
- The current best score

Tell the user: "AutoLoop is running in the background. You can keep using this
session — I'll update you when each batch finishes. To interact with the loop,
just tell me: stop, status, focus, avoid, or try."

### Batch spawning (context optimisation)

Each experiment adds ~5-6 tool calls to the sub-agent's context. Over a long
run (50+ experiments), accumulated context becomes expensive and slows down
reasoning.

**Spawn in batches of 10-15 experiments.** After each batch completes, the
main session spawns a fresh sub-agent for the next batch. Each spawn starts with
a clean context window. Continuity is preserved via `results.tsv` and the git
log on disk.

**Main session batch loop:**

1. Spawn sub-agent in background with: "Run experiments {N} to {N+batch_size}.
   Read results.tsv for history. Current best is {score}. Stop after
   {batch_size} experiments or if you receive a stop signal."
2. When notified that the sub-agent has completed, read `results.tsv` to get
   the updated best score and experiment count.
3. Report batch progress to the user (brief summary: experiments run, current
   best, number kept/discarded).
4. If the user hasn't said stop and no experiment limit is reached, spawn the
   next batch.
5. If the user said stop, or the limit was hit, send the final summary.

**Set a per-batch experiment limit** in the prompt (e.g., "run 15 experiments
then stop and summarise"). This ensures the sub-agent exits cleanly and the
main session can respawn with fresh context.

**Stopping behaviour with batches:** If the user configured "run until stopped",
the main session keeps spawning batches indefinitely. The user's stop command
goes to the main session, which either writes the signal file (stops the current
batch mid-run) or simply doesn't spawn the next batch (cleaner — waits for
current batch to finish).

### Sub-agent responsibilities

The sub-agent runs the Loop Phase (section above) for its assigned batch of
experiments. It:

1. `cd` to the project directory
2. Reads program.md for configuration
3. Reads the last 15 lines of results.tsv for recent history
4. Runs the benchmark to establish/confirm `current_best`
5. Executes experiments in the assigned range
6. Checks `.autoloop-signal` between each experiment
7. When the batch is complete (or stop signal received), returns a summary
   including: experiments run, current best, number kept/discarded/crashed

### Completion

When the last batch ends (user stop, experiment limit, or target score reached),
the main session sends the final summary to the user. The results persist on
disk in `results.tsv` and the git branch — the user can resume later with
`/autoloop` and asking to resume.
