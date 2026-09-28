---
layout: post
title: "Chip Harness, Part 3: The Evolution Loop"
date: 2026-09-28 00:03:00 -0400
series: chip-harness
---

{% include chip-harness-series.html %}

[Part 2]({% post_url 2026-09-28-Chip-Harness-Design-Loop %}) covered the design loop, where agents edit Verilog and every candidate is scored by real place-and-route. This post is about the loop wrapped around it: the harness noticing it's stuck, working out why, and **rewriting its own configuration**. It's also where I made my most instructive mistakes.

![Table comparing a typical AI agent today with Chip Harness](/assets/images/chip-harness/whats_new.png)
*What changes when the harness tunes itself.*

## A harness is a config document

The first design decision was to make the whole harness data, not code. Each harness version is one document in MongoDB Atlas, and the design agents are built from it on every iteration. Here's the seed version, h1, which is deliberately weak:

```python
H1_CONFIG = {
    "reports_read": ["summary_slack"],
    "tools": [],
    "context": {"lessons": 0},
    "diagnosis_prompt": "You are optimizing a Verilog MAC array for maximum clock "
                        "frequency. You are shown the worst setup slack of the parent "
                        "design. Propose one small change to the RTL that might improve timing.",
    "fix_families": [{"name": "micro_optimization", "weight": 0.5},
                     {"name": "logic_restructure",  "weight": 0.4},
                     {"name": "pipeline_register",  "weight": 0.1}],
    "tier2_policy": {"max_tier1_regression_ns": None},
    "guardrails": ["testbench must pass", "keep module ports and parameters unchanged",
                   "no removal of MAC units"],
}
```

h1 sees only one number, has no tools and no memory, and mostly tries low-value tweaks. I made it weak on purpose, so there would be something real to fix.

Because the harness is a document, "evolving the harness" means writing a new document with a parent pointer, a trigger, a rationale and a diff. The whole lineage can be queried and audited.

## Deciding it's stuck

After each round, `check_plateau` applies a fixed policy. Once a version has at least 4 trials, any one of these triggers evolution:

| Trigger | Rule |
|---|---|
| `stuck_rejecting` | The last 4 designs were all rejected |
| `no_gain` | The best valid design is less than 1% better than before the last 3 valid designs |
| `budget_cap` | 8 trials on this version |

The limits were tighter than I'd have liked because of the hackathon deadline. The important property is that the evolution agent **can't change them**.

## The evolution agent

When a plateau fires, a second Strands agent (the same Claude Sonnet 4.5, at a lower temperature) gets the plateau reason, the version's stats, its current config, and one tool, `trial_history`. The tool queries Atlas for every trial on that version: the fix family, whether it was valid, fmax, the rejection reason, and the agent's stated goal.

Its instructions boil down to: *progress has plateaued, diagnose why from the history, then rewrite the configuration.* It's given a menu of what it may change:

- the design agent's system prompt,
- which reports the agent sees (`summary_slack`, `critical_path`, `tier1_stats`),
- which tools it has (`tier1_check`),
- the fix families and their sampling weights,
- how many past lessons to inject (0–8),
- when to skip place-and-route based on the Tier 1 estimate,
- extra guardrails.

It replies with a short rationale and a complete new config.

## Guarding against reward hacking

![What the harness can rewrite (tools, reports, memory, search priors, compute, prompt) and what is locked (scorer, testbench, plateau rule, model, output sanitized)](/assets/images/chip-harness/evolvable_vs_locked.png)
*The boundary: what the harness can rewrite, and what stays out of its reach.*

A system that rewrites its own rules has an obvious failure mode: it "improves" by changing the rules. So the boundary is enforced in code, not left to the prompt. Every proposed config goes through `sanitize_config` before it's stored:

```python
c["reports_read"] = [r for r in new["reports_read"] if r in REPORT_OPTIONS] or ["summary_slack"]
c["tools"]        = [t for t in new["tools"] if t in TOOL_OPTIONS]
c.setdefault("context", {})["lessons"] = max(0, min(8, lessons))
...
c["guardrails"] = g if "testbench must pass" in g else ["testbench must pass"] + g
c["models"] = old["models"]   # pinned: comparisons between versions stay fair
```

Anything outside the allow-list is dropped, numbers are clamped, the testbench guardrail can't be removed, and the model can't be swapped. The scorer, the testbench and the plateau policy aren't in the config at all.

## Keep or retire

A new version runs until it hits its own plateau, and then it gets a verdict:

- **Kept** if its best design beats the best fmax that existed when it was created by at least 1%. It becomes the base for the next evolution.
- **Retired** otherwise. A lesson is written ("Harness change did not help: …"), and the next evolution branches from its parent instead.

This is a simple greedy search over harness configurations, with a hard, external fitness function.

## Run 1: a rewrite that was kept

At a 1 GHz target, h1's agents tried to pipeline the design and broke it 4 out of 4 times. `stuck_rejecting` fired, and 17 seconds later the evolution agent produced h2 with this diagnosis (trimmed):

> *All 4 trials failed testbench because the agent proposed functional changes without ability to test them first. The agent needs: (1) tier1_check tool to validate drafts before submission, (2) critical_path report to see actual bottlenecks not just slack numbers, (3) … explicit pipelining families with higher weight …*

That's the right diagnosis. With the tool and the critical-path report, h2's agents pipelined the design correctly: **909.5 → 1,332.2 MHz, +46.5%, kept.**

Then things got interesting. h3 and h4 each diagnosed that the agent was "repeatedly attempting nearly identical pipeline_register fixes" and rebalanced the fix-family weights toward unexplored strategies. That's a sensible read of the history, but neither found any gain, and both were retired. The real cause was the metric saturating at the clock target (see Part 2), which the evolution agent had no way to see. The run ended when the hackathon credits ran out, partway through h5.

## Run 2: the harness fixed its agents, and retired every fix

For the fully traced run I raised the target to 1.5 GHz. h1 got a lucky first design (1,529.8 MHz), then broke the design 7 times in a row. **1 of 8 designs valid.** The plateau rule flagged it 4 minutes 19 seconds in.

Over the next 90 minutes the evolution agent wrote **8 rewrites, each in 16–24 seconds.** All 8 gave the design agents `tier1_check` and the critical-path report. The effect on the agents was immediate and consistent:

- **32 of 32 designs valid** under h2–h9, with zero testbench failures.
- Agents took about 1½ minutes per round instead of about 10 seconds, because they were now testing their drafts before submitting them.

And **all 8 were retired**. The best, h4, reached 1,545.4 MHz, 0.9% above h1's lucky first design and just short of the 1% bar.

## What I got wrong

Looking at the logs, two design flaws explain almost everything.

**1. The keep rule measured the wrong thing.** I only scored peak fmax. By that measure the rewrites didn't help. By any reasonable engineering measure they were far better: 100% valid designs instead of 13%, and no wasted place-and-route runs. The harness did exactly what I told it to, and that's the lesson. A self-improving system optimizes your definition of "better", so the definition has to include reliability and cost, not just the headline number. Next time the keep rule will weigh pass rate and cost per valid design.

**2. The evolution agent had no memory of its own attempts.** When a version is retired, the next evolution branches from its parent, and in run 2 that was always h1. The `trial_history` tool looks up the history of the version being evolved *from*, so every evolution call saw the same h1 history, reached the same (correct) diagnosis, and wrote essentially the same fix. The "harness change did not help" lessons were stored in Atlas, but nothing ever showed them to the evolution agent. Eight independent rewrites agreeing is reassuring, but it's also wasted search. The fix is to give the evolution agent the lineage: which rewrites were already tried, and how they did. Vector search over past versions and lessons is the natural way to do that in Atlas.

## The audit trail

![Dashboard: fmax per trial colored by harness version, with markers where each version was evolved](/assets/images/chip-harness/dashboard.png)
*Run 2 on the dashboard. The dotted markers show where each new harness version took over.*

One part worked better than I expected: every change is explainable. Each version document holds its trigger, the trials that triggered it, the rationale, a unified diff of the config and the verdict, and the dashboard renders all of it. When the harness retired h4, I could see exactly what it had tried and why it didn't count. For anything you'd want to run unattended, that audit trail matters as much as the results.

## Tools in this part

The full toolbox is in [Part 1]({% post_url 2026-09-28-Self-Evolving-Chip-Design-Harness %}#the-toolbox). The evolution loop uses:

- **Strands Agents** with **Claude Sonnet 4.5 via OpenRouter**: the evolution agent and its `trial_history` tool.
- **MongoDB Atlas**: harness versions (config, parent, trigger, rationale, diff, verdict), lessons, and the trial history the evolution agent queries.
- **LangGraph**: the plateau check and the conditional hand-off from the design loop to evolution and back.
- **LangSmith**: traces of every evolution call, so each rewrite's reasoning can be inspected afterwards.
- **Next.js on Vercel**: the dashboard that shows the lineage and diffs.

## Next

- A keep rule that counts pass rate and cost per valid design, not just fmax
- Evolution with memory: the lineage and lessons from retired versions in its context
- A clock target the design can't reach (or a slack-margin score), so the fitness signal doesn't saturate
- Letting evolution choose models per role (cheap models for proposals, strong ones for diagnosis)
- Bigger designs, and place-and-route spread across more machines

This was a one-day prototype, and my first real attempt at building an agent harness. What I'm taking away isn't that the harness made a chip faster. It's that a hard verifier plus an auditable, self-modifying config is a workable pattern, and that the hard part is still deciding what "better" means.

Thanks for reading! The code is at [github.com/chmartin/chip-harness](https://github.com/chmartin/chip-harness).
