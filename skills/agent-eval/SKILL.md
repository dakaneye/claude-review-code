---
name: agent-eval
description: Reproducible evaluation harness for coding agents, prompts, and skills — head-to-head comparison with pass rate, cost, time, and consistency metrics captured in git worktrees. Use when comparing coding agents (Claude Code, Aider, Codex), regression-testing your own skills after changes, measuring variance across repeated runs of the same prompt, validating a model upgrade before adopting it, or A/B testing two versions of a prompt. Trigger on "compare Claude Code vs aider", "is my skill still passing", "run regression evals on this skill", "how much variance does this prompt have", "did the model upgrade regress anything".
---

# Agent Eval

**Version:** 1.0.0

A disciplined way to answer "is this agent/skill/prompt actually good?" with numbers instead of vibes. Runs tasks in isolated git worktrees, collects pass rate / cost / time / consistency, and produces a comparison report.

## What this is for

The original use case is head-to-head agent comparison — Claude Code vs Aider vs Codex on your own codebase. But the same harness covers four situations that aren't about picking a tool:

| Situation | Why the harness helps |
|-----------|----------------------|
| **Regression-test your skills** | You edited a skill and want to prove it still passes on the cases it used to. Run the skill's `evals/evals.json` under agent-eval with N=3 and compare against the previous version. |
| **Variance analysis** | A prompt works great sometimes and fails other times. Run it 10 times, see the consistency score, decide if the variance is acceptable or if the prompt needs tightening. |
| **Validate a model upgrade** | A new model just dropped (Claude 4.6, Sonnet 4.5, etc.). Re-run your regression suite against old and new, confirm nothing regressed before switching defaults. |
| **A/B test prompts** | You rewrote a system prompt. Run both variants against the same task set and compare pass rate and consistency to decide which ships. |

If what you want is a one-off "does this code work", use the normal test runner. Agent-eval is for measuring agents, prompts, and skills as *systems* — the kind of work where a single run is uninformative because the system is non-deterministic.

## Installation

Agent-eval is an external CLI. Install it from source after reviewing the code — it executes agent processes against your files, so know what you're running.

```sh
# Clone and install (or whatever the install method is for the fork you trust)
git clone https://github.com/joaquinhuigomez/agent-eval ~/dev/personal/agent-eval
cd ~/dev/personal/agent-eval && make install
```

If agent-eval is not available, the skill's workflow is still useful as a hand-run protocol — the structured task definitions and judge criteria work as a mental framework even without the automation.

## Core concepts

### Task definitions (YAML)

Every eval is a YAML file naming the task, the repo, the files under test, the prompt to hand to the agent, and the judge criteria.

```yaml
name: add-retry-logic
description: Add exponential backoff retry to the HTTP client
repo: ./my-project
commit: "abc1234"          # pin to a specific commit for reproducibility
files:
  - src/http_client.py
prompt: |
  Add retry logic with exponential backoff to all HTTP requests.
  Max 3 retries. Initial delay 1s, max delay 30s.
judge:
  - type: pytest
    command: pytest tests/test_http_client.py -v
  - type: grep
    pattern: "exponential|backoff|retry"
    files: src/http_client.py
```

**Pinning the commit** is the non-negotiable part. If you re-run this task next month and the main branch has moved on, you're measuring a different problem.

### Git worktree isolation

Each run spawns a fresh `git worktree` from the pinned commit. No Docker, no container setup — just native git isolation. The agent can't corrupt your working copy and can't see other agents' output from the same session. When the run finishes, the worktree is either kept for inspection or cleaned up.

### The four metrics

| Metric | What it measures | When it's the decisive one |
|--------|------------------|----------------------------|
| **Pass rate** | Did the agent produce code that satisfies the judge? | Always — this is the ground truth. |
| **Cost** | API spend per task (when the agent reports it) | Cheaper agent at 90% pass rate may beat a premium agent at 95%. |
| **Time** | Wall-clock seconds from prompt to judged result | Interactive coding workflows — slow agents break flow even when correct. |
| **Consistency** | Pass rate across N repeated runs of the same task | The most underrated metric. A 3/3 pass beats a 2/3 every time; non-determinism is a bug. |

## Workflow

### 1. Write tasks

```sh
mkdir tasks
# One YAML per task. Aim for 3-5 tasks that represent your real workload.
# Toy examples measure toy performance.
```

Tasks should be at the granularity you actually delegate to an agent — a feature, a refactor, a bug fix — not "rename a variable" or "write Fibonacci". Include at least one task per type of work you care about (new code, refactor, bug, test-writing).

### 2. Run

```sh
# Single task, single agent, 3 trials
agent-eval run --task tasks/add-retry.yaml --agent claude-code --runs 3

# Head-to-head
agent-eval run --task tasks/add-retry.yaml \
  --agent claude-code \
  --agent aider \
  --runs 3
```

Always run at least 3 trials. Agents are non-deterministic — a single pass or fail tells you nothing about consistency.

### 3. Report

```sh
agent-eval report --format table
```

```
Task: add-retry-logic (3 runs each)
┌──────────────┬───────────┬────────┬────────┬─────────────┐
│ Agent        │ Pass Rate │ Cost   │ Time   │ Consistency │
├──────────────┼───────────┼────────┼────────┼─────────────┤
│ claude-code  │ 3/3       │ $0.12  │ 45s    │ 100%        │
│ aider        │ 2/3       │ $0.08  │ 38s    │  67%        │
└──────────────┴───────────┴────────┴────────┴─────────────┘
```

Pass rate is the first column but not the only one. A 95% agent at 10x the cost is often the wrong answer.

## Judge types

### Code-based (deterministic)

The most trustworthy judges are tests and builds. They don't lie and they don't add variance.

```yaml
judge:
  - type: pytest
    command: pytest tests/ -v
  - type: command
    command: npm run build
```

### Pattern-based

Use these as a *secondary* signal — to catch cases where the agent passed tests by hardcoding or faking.

```yaml
judge:
  - type: grep
    pattern: "class.*Retry|exponential_backoff"
    files: src/**/*.py
```

### Model-based (LLM-as-judge)

Use only when deterministic judges aren't available — e.g. judging explanation quality or documentation clarity. LLM judges add noise; never rely on them alone.

```yaml
judge:
  - type: llm
    prompt: |
      Does this implementation correctly handle exponential backoff?
      Check for: max retries, increasing delays, jitter.
```

Rule of thumb: every task should have **at least one deterministic judge**, even if it also has an LLM judge. If you can't write a deterministic judge for a task, the task is probably too fuzzy to evaluate.

## Evaluating your own skills

Claude skills ship with `evals/evals.json` (reasoning cases) and `evals/trigger-evals.json` (activation discrimination). Agent-eval can run the reasoning cases through the skill under test and measure pass rate across N trials.

Pattern for regression-testing a skill you just edited:

```yaml
name: skill-regression-prompt-v2
description: Verify /prompt still passes its own reasoning evals after iteration 2 changes
repo: ~/.claude
files:
  - skills/prompt/SKILL.md
prompt: |
  Load the skill at ~/.claude/skills/prompt/SKILL.md.
  Run each case in ~/.claude/skills/prompt/evals/evals.json.
  For each case, invoke the skill with the prompt and judge whether the
  response matches the expected_output description.
  Return a pass/fail per case and a summary.
judge:
  - type: llm
    prompt: |
      Check the skill achieved >= 8/10 pass on its reasoning evals.
      Any regression from the previous version is a failure.
```

The previous result is the baseline. If pass rate drops or consistency drops, the edit regressed something.

## Best practices

- **3–5 tasks minimum** drawn from real workload, not synthetic examples. Toy tasks measure toy performance.
- **Pin commits** in every task YAML. Otherwise "reproducible" is a lie.
- **At least one deterministic judge per task.** LLM-only judges are too noisy to gate decisions on.
- **Run at least N=3 trials** to capture variance. N=1 tells you nothing about consistency — the most important long-term metric.
- **Track cost alongside pass rate.** A correct-but-expensive agent may still be the wrong choice for your budget.
- **Version your task YAML.** They are test fixtures; commit them to the repo they test.
- **Re-run after every model/tool upgrade.** Model drift is real; baselines go stale in weeks, not months.

## When NOT to use agent-eval

- **Single deterministic tests** — use the normal test runner. Agent-eval's overhead isn't worth it for code you wrote by hand.
- **One-off comparison** that doesn't need to be reproducible — just try both tools informally and decide.
- **Tasks that can't be judged without a human** — skill at explanation, code review subtleties, UX judgment. Don't force an LLM judge where a human is required.

## Related

- Skill evaluation bar: every shipped skill under `~/.claude/skills/` has `evals/evals.json` and `evals/trigger-evals.json`. Agent-eval operationalizes those.
- `skill-stocktake`: periodic quality audit of skills (uses subagent judgment). Agent-eval is the quantitative counterpart.
- Upstream reference: [github.com/joaquinhuigomez/agent-eval](https://github.com/joaquinhuigomez/agent-eval)
