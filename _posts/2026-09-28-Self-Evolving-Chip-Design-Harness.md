---
layout: post
title: "Chip Harness, Part 1: A Harness That Rewrites Itself"
date: 2026-09-28 00:01:00 -0400
series: chip-harness
series_title: a self-evolving chip-design harness
---

{% include series.html %}

It has been a while! The posts just before this one look back at two earlier projects: [GamerVapor]({% post_url 2019-08-01-GamerVapor-Predicting-Churn-on-Steam %}), a churn model from 2019, and a [2023 hackathon chatbot]({% post_url 2023-06-01-Natural-Language-SQL-Chatbot %}) built in the first months of ChatGPT. Read in order, they show how much the tools have changed. One thing hasn't: I still like problems where the data can't lie to you.

On Saturday, Sep 26, I competed solo in **The Harness Engineering & Model Wrangling Hackathon**, hosted by MongoDB and Cerebral Valley in New York. I didn't win, but I'm proud of what I built in one day, and I learned something I think is worth sharing.

The short version: I built a system where AI agents redesign a small piece of a chip to make it faster, and when the agents get stuck, **the system rewrites its own instructions, tools and memory**, then keeps only the changes that measurably help.

- Code: [github.com/chmartin/chip-harness](https://github.com/chmartin/chip-harness)
- Live dashboard (while the hackathon's database lasts): [chip-harness.vercel.app](https://chip-harness.vercel.app/)

## First, what is a "harness"?

When people talk about AI agents, they usually talk about the model. But an agent is only as good as what surrounds the model: what it's told, what it's allowed to see, which tools it can call, what it remembers from last time, and when it should stop. That surrounding scaffolding is the **harness**.

Today, humans tune harnesses by hand. You watch the agent fail, you tweak the prompt, you add a tool, you try again. The hackathon's challenge (Problem Statement 1, "Recursive Harnessing") asked: can the harness improve itself?

![Table comparing a typical AI agent today with Chip Harness](/assets/images/chip-harness/whats_new.png)
*How this differs from a typical agent setup today.*

I was inspired by a write-up on a [meta-harness used to design a chip for Kimi K3](https://www.luoluo.ai/blog/kimi-k3), and wanted to try the idea on a problem small enough to run on my laptop in an afternoon.

## Why chip design is a great test bed

![Timing closure loop: edit RTL, simulate, synthesize, place and route, read timing](/assets/images/chip-harness/timing_closure.png)

Getting a chip to hit its target clock speed ("timing closure") is a slow loop: edit the hardware description (Verilog), check it still works, synthesize it into gates, place and route it into real wires, and only then read out how fast it can run. Every pass is expensive, and the true answer arrives at the end.

That makes it a nearly perfect test bed for self-improving agents, for one reason: **the score can't be faked.** There is no "LLM as a judge" here. A design either passes a randomized testbench and routes cleanly at a given frequency, or it doesn't. If the harness is going to evolve itself, you want it evolving against something that honest.

My test block was a 4×4 array of INT4 multiply-accumulate units, the basic building block of AI accelerators. Everything ran on open-source tools: Yosys for synthesis and OpenROAD for place-and-route on the Nangate45 open cell library.

## Two loops

The system runs two loops, one inside the other, sharing one database as memory.

![Architecture: the evolution loop above the design loop, both writing to MongoDB Atlas](/assets/images/chip-harness/architecture.png)
*The architecture. The design loop (blue) edits, verifies and measures every design. The evolution loop (orange) rewrites the harness when progress stalls. Everything is stored in MongoDB Atlas (green).*

**The design loop.** Four AI agents each propose a change to the Verilog to raise the maximum clock frequency (fmax). Each proposal goes through a gate:

1. A randomized, self-checking testbench (seconds). Broken designs stop here.
2. A fast synthesis estimate (seconds).
3. Full place-and-route (about 5 minutes). This produces the real score.

A design only counts if it passes the testbench, completes the flow, and has zero design-rule violations.

**The evolution loop.** After every round, a plateau check asks: is this harness stuck? (Four rejections in a row, less than 1% gain over the last three valid designs, or eight trials on one version.) If so, an **evolution agent** reads the full trial history, diagnoses *why* the harness is stuck, and writes a new harness version. It can change:

- the agents' instructions,
- which reports they get to read (e.g. the critical-path report),
- which tools they can call,
- which kinds of fixes they try, and how often,
- how many past lessons they're shown,
- when to skip the expensive place-and-route step.

What it **cannot** change is the scorer, the testbench or the plateau rule. Those sit outside its reach, which is what keeps it from "improving" by gaming the metric.

Every new version must beat its parent by at least 1% or it is retired, and the failure is stored as a lesson for the future.

## What happened

### Run 1: the rewrite made the chip 46.5% faster

<div style="display:flex; gap:8px; flex-wrap:wrap;">
  <figure style="flex:1; min-width:240px; margin:0;"><img src="/assets/images/chip-harness/run1_before.jpg" alt="Worst timing path of the baseline design"><figcaption><em>Before: baseline, 909.5 MHz</em></figcaption></figure>
  <figure style="flex:1; min-width:240px; margin:0;"><img src="/assets/images/chip-harness/run1_after.jpg" alt="Worst timing path after the evolved harness's design"><figcaption><em>After: evolved harness, 1,332.2 MHz</em></figcaption></figure>
</div>

The first harness (h1) was deliberately weak: a summary number, no tools, no memory. Its agents tried the textbook fix, pipelining, but broke the design every time (for example, registering the multiplier output without delaying the matching enable signal). The testbench rejected all four attempts.

That tripped the plateau rule. The evolution agent read the failures and wrote h2, whose main change was a **tool**: agents could now run the testbench and a quick synthesis on their own draft *before* submitting it. It also showed them the critical-path report.

Under h2, the agents pipelined correctly and fmax went from **909.5 MHz to 1,332.2 MHz (+46.5%)**, measured after full place-and-route. h2 was kept.

### Run 2: the harness fixed its own agents

![Demo: the loop running in a terminal next to the live dashboard](/assets/images/chip-harness/demo_poster.jpg)
*The demo setup: the loop running in the terminal (left) and the dashboard filling in as the harness evolves (right).*

![Dashboard: fmax per trial, colored by harness version](/assets/images/chip-harness/dashboard.png)
*Run 2 on the live dashboard. Every dot is a design that went through full place-and-route; ticks under the axis are designs the testbench rejected. (The +70% tile is best design vs. baseline, and most of that came from h1's lucky first try, which is why I quote run 1's +46.5% as the result.)*

For the second, fully traced run I raised the target to 1.5 GHz. This time h1 got lucky: its very first design landed at about 1.53 GHz. Then its agents broke the design seven times in a row trying to build on it. **Only 1 of its 8 designs passed verification.**

About four minutes in, the harness flagged itself as stuck. The evolution agent wrote h2 in **17 seconds**. From then on, across eight evolved versions:

- **32 of 32 designs passed verification**, with zero testbench failures.
- Broken ideas were caught by the agents themselves in seconds, instead of after minutes of routing.

All 40 trials took 96 minutes and 4.6 CPU-hours on my laptop, for about $13 of model calls. The whole day cost $21.10.

![Run 2 metrics: 13% to 100% valid designs, 7 to 0 testbench failures, 4 min 19 s to self-diagnose, 8 of 8 rewrites added a self-test tool](/assets/images/chip-harness/run2_metrics.png)
*Run 2 in numbers.*

## The most useful result was a "failure"

Here's the twist: **all eight evolved harnesses in run 2 were retired.** None beat h1's lucky first design by the 1% bar (the best, h4, came in at +0.9%).

My first reaction was disappointment. My second was that this is exactly what I want a self-improving system to do: it refused to count noise as progress. But it also exposed a flaw in *my* rule. I only scored speed, so the harness retired versions that were clearly better agents: 100% valid designs instead of 13%, and far less wasted compute.

That's the lesson I keep coming back to: **a self-improving system is only as good as the definition of "better" you give it.** The next version should keep a harness if it improves pass rate or cost per valid design, not just peak speed.

A few other things I'd do next:

- Give the evolution agent memory of which rewrites were already tried and retired (all eight independently reached the same fix, which is reassuring but wasteful).
- Use a clock target the design can't reach, so the score doesn't saturate once it's met.
- Try bigger designs, and spread place-and-route across more machines.

## The toolbox

The next two posts go deeper: [Part 2]({% post_url 2026-09-28-Chip-Harness-Design-Loop %}) on the design loop, and [Part 3]({% post_url 2026-09-28-Chip-Harness-Evolution-Loop %}) on the evolution loop. Here is every product the harness uses, what it is, and what I used it for. Several of these were new to me, so treat this as a one-day prototype, not a reference design.

**AI and orchestration**

- **[LangGraph](https://langchain-ai.github.io/langgraph/)** (LangChain) is a framework for building long-running, stateful agent workflows as graphs of steps. *I used it for* both loops: each step (pick a parent, propose, evaluate, check for a plateau, evolve) is a node, and the edges decide what runs next.
- **[Strands Agents](https://strandsagents.com)** (AWS) is an open-source SDK for building agents: a model, a system prompt and a set of tools, with the agent deciding when to call them. *I used it for* the design agents and the evolution agent. Each agent's tools are built from the harness config, so evolution can change what they're allowed to do.
- **[OpenRouter](https://openrouter.ai)** is a single API in front of many model providers. *I used it for* every model call, through one key. The model is a field in the harness config, so it could be swapped per role.
- **Claude Sonnet 4.5** (Anthropic) is a large language model. *It was* the model for every agent, pinned across all harness versions so comparisons between versions stayed fair.
- **[LangSmith](https://smith.langchain.com)** (LangChain) is a tracing and observability platform for LLM apps. *I used it for* tracing every LangGraph node, plus the Strands agents' model and tool calls (exported over OpenTelemetry), so I could see what each agent did and why.

**Data**

- **[MongoDB Atlas](https://www.mongodb.com/atlas)** is MongoDB's managed cloud document database. *I used it as* the harness's single memory: every trial, every harness version (with its rationale and diff), the lessons, and the LangGraph checkpoints that let a killed run resume. Documents were a natural fit: the schema changed three times during the day with no migrations.

**Chip design (all open source)**

- **[Icarus Verilog](https://steveicarus.github.io/iverilog/)** is a Verilog simulator. *I used it to* run the randomized testbench, the first gate every design must pass.
- **[Yosys](https://yosyshq.net/yosys/)** and **[ABC](https://github.com/berkeley-abc/abc)** are a logic synthesis tool and the logic optimizer it uses. *I used them for* the seconds-long synthesis and rough timing estimate, and inside the agents' `tier1_check` self-test tool.
- **[OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build)** is a prebuilt bundle of open-source hardware tools. *It gave me* Icarus Verilog and Yosys in one download.
- **[OpenROAD](https://theopenroadproject.org)** and **[OpenROAD-flow-scripts](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)** are an open-source toolchain that takes a design from synthesized gates to a finished chip layout, along with scripts that run the whole flow. *I used them for* full place-and-route, the source of the real score: fmax after routing, with zero design-rule violations.
- **Nangate45** is an open-source 45 nm standard-cell library (the building blocks a layout is made of). *It was* the target technology for every design.
- **[Docker](https://www.docker.com)** runs software in isolated containers. *I used it to* run OpenROAD from its official image, four place-and-route jobs in parallel on my laptop.

**Dashboard and development**

- **[Next.js](https://nextjs.org)** is a React framework for web apps. *I used it for* the live dashboard: the fmax-per-trial chart, harness versions and their config diffs.
- **[Vercel](https://vercel.com)** is a hosting platform for web apps. *I used it to* publish the dashboard, which reads Atlas through a read-only user.
- **[GitHub](https://github.com)** hosts the [code](https://github.com/chmartin/chip-harness), and I tracked the day's tasks in **[Linear](https://linear.app)**, an issue tracker.
- **[Claude](https://claude.ai)** (Anthropic's assistant) was my pair-programmer and planning partner for the day.

## Why this matters beyond chips

The pattern isn't specific to hardware. Anywhere there's a hard, trustworthy check (compilers, GPU kernels, database query plans) you can let a harness tune itself, keep only the changes that pass, and get a full audit trail of what changed and why.

Thanks to MongoDB and Cerebral Valley for hosting, OpenRouter for the credits that powered every agent call, and the OpenROAD and Yosys communities for making open-source chip design possible.

As always, the data doesn't lie. Sometimes it just tells you that you asked the wrong question.
