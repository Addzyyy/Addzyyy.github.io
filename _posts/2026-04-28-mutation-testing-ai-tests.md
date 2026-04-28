---
layout: post
title: "How Good Are AI-Written Tests? I Mutation-Tested Five AI-Built Projects to Find Out"
date: 2026-04-28
---


When AI writes both your code and your tests, who's auditing the auditor? The tests pass. Coverage looks great. But are those tests actually catching bugs, or just exercising lines?

I ran mutation testing on five projects I'd built end-to-end with AI to find out. The results surprised me, but the most useful thing I take away from the experiment isn't a benchmark number — it's a proposal for what AI coding agents should be using as their quality feedback signal. We'll get to that.

## 🔥 The headline finding — read this if nothing else

> *(Mutation testing introduces deliberate small bugs into your code and checks whether your tests catch them. Coverage tells you which lines were touched. Mutation testing tells you whether the touch was load-bearing.)*
>
> 📊 **Caveat first, so you can calibrate the rest:** this is observational, not a controlled study. **Five projects, single sample each, single model (Claude Opus 4.6), single author.** The patterns below repeated reliably across the dataset, but five projects is not a benchmark — it's a starting point. Read everything that follows as suggestive, not proven.
>
> **Across all five projects, the AI-written test suites looked great by every traditional metric — and let roughly half the bugs through anyway.**
>
> **What looked fine:**
> - Tests **passed** — 100% green, every time.
> - Coverage was **high** — 91% on cvesieve, 92% on docklet, 76% on test-slicer Core.
> - Test counts were **substantial** — 169, 171, 162 tests on the largest projects.
>
> **What mutation testing actually showed:**
> - **No project hit 80%** (the standard "healthy" bar). Docklet came closest at 75%; the rest sat between 41% and 69%, with most clustering around 55%. Two projects reported 22–33% via a different failure mode covered later.
> - **The cvesieve result was the starkest:** 91% line coverage, 55% mutation score. A 36-point gap. In plain language: *"your tests touched every line but missed nearly half the bugs."*
> - **The pattern was consistent** — across three languages (Python, C#, TypeScript) and three different mutation testing tools (mutmut, Stryker.NET, Stryker.JS). Not a one-off.
>
> ⚠️ **If you ship something an AI wrote and tested, and you trust the green CI as your safety net: based on these five projects, your tests can look strong by coverage and still miss a large fraction of realistic faults.** They exercise the code without strongly verifying its behaviour. (To be clear: a surviving mutant doesn't necessarily mean a test is "useless" — some mutants are equivalent, unrealistic, or outside the relevant contract. But a sustained 30+ point gap between coverage and mutation kill rate, repeated across projects, is a real quality signal, not a measurement artefact.)
>
> **A behavioural pattern, stated cautiously:** I can't directly observe what the AI was optimising for. But the data is consistent with the AI taking the metric it can see (does the test pass? does it touch the line?) as a proxy for the metric it can't (does this assertion verify behaviour?). The most useful single sentence I can take from the experiment is this: **AI writes the weakest assertion the contract of the code permits, every time.** When the contract is tight, that's enough. When it's loose, half the bugs walk past.
>
> The post breaks this into two distinct failure modes, explains why one outlier project scored 75% (a near-control-group result that points the finger at code shape, not at the AI), and ends with what I think is the most useful takeaway: **mutation testing should be the feedback signal in AI coding agent loops, not coverage** — with one important caveat about overfitting that I get to at the end.

---

The rest of this post is the case for that finding — the projects, the data, and what the patterns add up to.

## The setup

All five projects were built using **[Claude Code](https://claude.com/claude-code)** with **Claude Opus 4.6** as the underlying model. Code and tests were AI-written end-to-end — no human-authored test code in any of these.

| Project | Language | What it does |
|---|---|---|
| **docklet** | Python | Minimalist Docker clone — Linux namespaces, cgroups, container management |
| **test-slicer** | C# | Test impact analysis tool with three sub-projects (Core / Cli / Roslyn Generator) |
| **cvesieve** | Python | CVE scanner output filter with NVD/EPSS/KEV enrichment |
| **open-race-telemetry** | C# | F1 telemetry pipeline: UDP → Kafka → TimescaleDB |
| **focuslock** | TypeScript / Electron | Calendar-synced website blocker desktop app |

For each, I measured:
- **Mutation score** — what percentage of injected faults the test suite catches
- **Line and branch coverage** — what percentage of code/decision points the tests touch
- **Test count, mutant count, LOC, test:code ratio**

Tools: [Stryker.NET](https://stryker-mutator.io/) for C#, [mutmut](https://github.com/boxed/mutmut) for Python, [Stryker.JS](https://stryker-mutator.io/docs/stryker-js/introduction/) for TypeScript.

## What is mutation testing, briefly

The mutation testing tool takes your source code and makes small, semantically-meaningful changes (mutations): flip `>` to `>=`, change `True` to `False`, replace a constant with a different one, drop a `not`, and so on. Then it runs your test suite. If a test fails, the mutant is "killed" — the test caught the change. If all tests still pass, the mutant "survived" — your tests didn't notice the bug. Your mutation score is the percentage of mutants killed.

**Coverage tells you whether tests *touched* a line. Mutation testing tells you whether tests would *catch a bug* on that line.** That distinction is the entire point of this post.

## The numbers

| Project | Mutation Score | Line Cov | Branch Cov | Test:Code Ratio |
|---|---:|---:|---:|---:|
| **docklet** | **74.97%** | 91.55% | 85.00% | 2.52 |
| test-slicer (Core) | 68.64% | 76.12% | 95.38% | 2.09 |
| cvesieve | 55.23% | 90.73% | 83.33% | 1.15 |
| test-slicer (Cli) | 54.59% | 54.57% | 56.10% | 1.69 |
| test-slicer (Generator) | 41.58% | 65.80% | 56.02% | 1.82 |
| focuslock ¹ | 32.97% / 85.71% | 38.86% | 31.70% | 1.10 |
| open-race-telemetry ¹ | 21.65% / 79.02% | 47.18% | 33.44% | 0.92 |

¹ Two scores: "all mutants" / "covered code only". Both projects have files the AI deliberately left out of the unit test scope. Discussed further down.

There turn out to be two distinct flavours of this — one a real problem, one partly a problem and partly the right answer. Let's look at each.

## Failure mode 1: Weak assertions on covered code

This is the cvesieve story, and it's the dominant failure mode in the data.

Cvesieve is a CVE scanner output filter — it parses Trivy JSON, enriches with NVD/EPSS/KEV data, and formats reports. Lots of data transformation, lots of fields, lots of optional structure. The kind of code where a test can get away with this:

```python
result = format_report(cves)
assert "CVE-2024-1234" in result
```

That assertion lets dozens of mutations slip past. Change the formatting, drop a field, reorder the output — the CVE ID is still in there somewhere. The test passes. Coverage rises. The mutation testing tool calmly notes that 163 mutations of `output.py` survived.

The same pattern shows up in **test-slicer Cli** (55% mutation, 55% coverage) and **test-slicer Generator** (42% mutation, 66% coverage). Wherever the AI wrote tests for code with loose contracts — text formatting, multi-field data structures, configurable output — the assertions tended to be loose too. Tests that effectively checked "did this not throw, and is the keyword somewhere in the output" rather than "is the output exactly what it should be."

Per file, the worst offenders cluster predictably:

- `cvesieve/output.py` — 91% covered, 42% mutation. **49-point gap.** Output formatter.
- `cvesieve/enrichment/nvd.py` — 90% covered, 50% mutation. HTTP enrichment.
- `test-slicer Generator` — 66% covered, 42% mutation. Roslyn syntax-tree code.

The fix for this failure mode is straightforward to describe and slow to do by hand: push back on assertion quality. Assert on full output structure, not substrings. Assert on exact mock call args, not "was this called." Assert on returned data values, not just "did this return without throwing." A reviewer doing this manually finds it tedious. A signal that catches it automatically would be more useful — that's the proposal in the back half of this post.

## Failure mode 2: Files the AI skipped entirely

The other two projects show a different shape. **focuslock** and **open-race-telemetry** report headline mutation scores of 33% and 22% — well below the rest of the field. But look at the second number in their headline rows: 86% and 79%. Those are the mutation scores on the code that *was* tested. Both are at or above the 80% bar; focuslock's is the highest covered-code score in the entire benchmark.

What's happening is that the AI made a deliberate split:

- **open-race-telemetry** is a real I/O pipeline (UDP → Kafka → TimescaleDB). The AI wrote unit tests for the pure logic (`PacketMapper`, `KafkaMessageSerializer`) and TestContainers integration tests (real Kafka + Postgres in Docker) for the I/O classes. I excluded the integration tests from this mutation run to keep the tooling consistent across projects, and that pulled the headline score down.
- **focuslock** is an Electron app. The AI wrote unit tests for the pure-logic files (blocklist loader, calendar poller, session scheduler, host blocker, app wiring) and left two files essentially untested: `main.ts` (the Electron bootstrap) and `google-calendar-api.ts` (the Google OAuth wrapper). Both need a real runtime to exercise meaningfully.

In both cases, the AI quietly classified part of the codebase as "I'm not going to unit-test this." The CI dashboard doesn't tell you that — "all my tests pass" doesn't mean "all my code is tested."

### When 0% is the right answer, and when it isn't

It's worth interrogating whether those skipped files *should* be unit-tested rather than assuming "0% is bad, write more tests." Some of those 0% scores are correct.

Take **`google-calendar-api.ts`** in focuslock. It does two jobs: (1) call `calendar.events.list()` with the right parameters, and (2) map the response items to internal `FocusEvent` objects. The API-call part **shouldn't have unit tests** — mocking `googleapis` to test that you call `googleapis` correctly is busywork that tests your mock setup, not your code. That part of the 0% is fine. But the **mapping logic is real, has edge cases (default values for missing fields, date parsing, filtering events without `dateTime`), and is absolutely testable** — if it were extracted into a pure function. The AI inlined it with the I/O and skipped the lot.

Same thing in **`main.ts`**. Most of it is genuine Electron wiring (window creation, IPC handlers, app lifecycle) — fine to leave un-unit-tested. But interleaved with that wiring are pure-ish functions: `loadSettings()`, `saveSettings()`, `loadOAuthConfig()`, `loadCredentials()`. File I/O with JSON parsing and error handling. And the project README explicitly calls out hosts-file crash recovery as the project's #1 reliability concern — that logic is in `main.ts` too, also untested.

So the right answer to "should these files be tested?" is **partly yes, partly no**, and the parts are tangled together inside the same files.

That tangle is the deeper finding, and it's worth stating clearly because it's bigger than testing:

> **AI is implicitly making architectural decisions — about what is unit-testable and what isn't — without making the corresponding refactoring decisions to support its own testing strategy.**

A human writing the same thing would notice "this file has a pure mapping function inside an Electron lifecycle handler" and pull the mapping out so it could be tested in isolation. The AI doesn't do that. It writes the inline version, classifies the whole file as "needs runtime → won't unit-test it," and moves on. The CI dashboard says all green; the test suite says high coverage; the actual code has a chunk of business logic that's untested *because of how the AI structured the file*, not because the logic is genuinely un-testable.

This is the more durable critique of AI-written code, separate from "AI tests are weak":

- **AI is doing architecture by default**, every time it decides where to draw a function boundary.
- **It's not optimising those boundaries for testability** — it's optimising for "does this work end-to-end."
- **The cost surfaces as 0% mutation scores on files that ought to have 80%+** on the half of their content that's pure logic.

The fix here isn't more tests. It's prompting the AI to **separate side-effectful code from pure logic before writing tests** — and ideally to do so as a structural step before any test code at all. That shrinks the bucket of "this can't be unit-tested" to its actual size, and stops the bucket of "tested but with weak assertions" from growing to fill the gap.

It also means **don't mechanically chase 100% mutation score.** A project where every file scores 80%+ might just mean the AI wrote pointless tests for code that didn't need them. The right target is "every file that *contains testable logic* clears 80%, and the wiring/SDK-shim files are deliberately untested with a one-line comment explaining why."

## Docklet isn't an outlier — it's a near-control-group

I keep calling docklet "the outlier," but that undersells it. Docklet vs. cvesieve is the closest thing this experiment has to a controlled comparison:

- Same model (Claude Opus 4.6).
- Same author (me).
- Same language (Python).
- Same mutation testing tool (mutmut).
- Similar test counts (171 vs. 169).
- Similar mutation density (~0.83 mutants per sloc on both).

**The thing that varied was the shape of the code.** And the mutation score moved from 55% to 75% — a 20-point swing on the same model and the same author. That's about as close to causal evidence as you get from observational data with this sample size: the code shape did the work, not the AI.

That reframes the whole experiment. The headline isn't really *"AI tests are weak."* It's *"AI tests are weak on weakly-shaped code, and acceptable on tightly-shaped code."* Which makes mutation testing not just a quality measurement, but a **detector for where your code structure is letting the AI off the hook.** The next three subsections are what specifically about docklet's code made the difference.

### Factor 1: Tight contracts force tight assertions

Docklet's code is dominated by "call this syscall with these exact arguments":

```python
os.unshare(CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET)
subprocess.run(["ip", "link", "add", veth_name, "type", "veth", ...])
write_to_cgroup_file("cpu.shares", "1024")
```

When you mock these calls in tests, your assertions *have to* be precise — the exact flag value, the exact arg list, the exact bytes. There's no looser way to verify the behaviour. So a mutation flipping `CLONE_NEWPID` to a different constant, or `"add"` to `"remove"`, dies instantly because the mock recorded it differently than the test expected.

This is the inverse of cvesieve's `output.py`. Where cvesieve's contracts let lazy assertions slip through, docklet's contracts demanded precise ones.

### Factor 2: Branch density

Docklet's code has fewer branches per line than the others. Mutants hide best in branches — comparison operator flips, boolean logic changes, condition negations. With fewer branches, less surface area for mutations to survive on.

| Project | Branches / sloc |
|---|---:|
| **docklet** | **0.144** |
| open-race-telemetry | 0.230 |
| cvesieve | 0.205 |
| test-slicer Core | 0.362 |
| test-slicer Cli | 0.668 |
| test-slicer Generator | 0.696 |

Cvesieve has 42% more branches per line than docklet. And docklet's two perfectly-scoring files — `cgroups.py` (100%) and `config.py` (100%) — literally have **zero branches**. They're declarative. The only mutations possible are on constants, return values, and arithmetic — exactly the mutations any reasonable assertion catches automatically.

### Factor 3: Mocking discipline (counterintuitively)

Docklet *had* to mock everything. You can't `clone()` into new namespaces from pytest as a non-root user, can't actually write to `/sys/fs/cgroup`, can't actually create veth pairs. The AI had no choice but to mock at the syscall/subprocess level.

Conventional wisdom says heavy mocking makes tests fragile and weak. But here, **forced mocking made tests stronger** — because mocking pushes you to specify behaviour precisely. You don't get to write `assert "ok" in result` against a mock. You have to assert on the exact `mock.call_args`.

This contradicts what open-race-telemetry implies (that the AI deferred I/O to integration tests because integration tests are "stronger"). A well-mocked unit test of a syscall is plenty strong, and you don't need Docker to make it so.

### What this isn't

It's worth being explicit about what *didn't* explain the gap between docklet and cvesieve:

- **Not language.** Same language, same tool — both Python with mutmut.
- **Not test count.** Docklet 171, cvesieve 169. Two tests apart.
- **Not mutant count.** Both have ~0.83 mutants per line of production code.

### The takeaway: AI tests are only as good as the contract forces them to be

Strip out the diplomatic framing for a moment. Three of these projects are the same author (me + Claude Opus 4.6) writing tests for very different code shapes. The mutation scores moved from the 40s to the high 90s as the *contract* of the code being tested moved from loose to tight. The AI didn't suddenly get better at testing between those projects. The code stopped letting it be sloppy.

That's the actual finding lurking under the docklet result: **AI is bad at writing tests for code that doesn't have a strong contract.** When the contract is precise — exact syscall args, exact byte-level outputs, exact transforms — the AI writes tight tests, because there's no looser assertion that would even compile and run. When the contract is loose — text formatting, JSON output with optional fields, "the response should contain a CVE somewhere" — the AI defaults to assertions like `assert "CVE-1234" in result` and stops. Coverage rises, the test passes, the loop closes, and ~half the mutations on that file survive.

This asymmetry matters because it is **not symmetric**:

- Tight contracts *force* tight tests — the floor is high regardless of who writes them.
- Loose contracts *allow* tight tests but don't require them — and the AI consistently doesn't write the tight version when a loose version satisfies the same coverage metric.

A skilled human writer of `output.py` tests would push past the easy assertion. They'd think "but I should also be checking the field order, the separator, the rounding, the empty case." The AI doesn't reliably do that — it takes the lowest-effort assertion that exercises the line and moves on. **That's the failure mode in one sentence: the AI writes the laziest assertion the contract permits, every time, and on loosely-shaped code that's not enough.**

So when you read the comparison chart, don't read it as "docklet's AI tests are good and cvesieve's AI tests are bad." Read it as "docklet's code didn't *let* the AI write bad tests; cvesieve's code did, and the AI took the offer." The right question for any AI-written test suite isn't "what's the mutation score" — it's "did the AI find a way to write loose assertions on this file, and is that hiding a coverage→mutation gap?"

That's not a reason to ignore mutation testing — it's still a brutally honest measurement of "would this test catch a bug." It's a reason to **read the per-file gap, not the headline number**, and to **prompt the AI specifically harder on the loose-contract files** (output formatters, CLI handlers, multi-field data transforms) because that's where it will silently underperform.

## Patterns that held across every project

Even with the project-shape caveat, some failure modes were consistent everywhere:

**1. Pure logic gets strong AI tests (70–100%).**
- `cvesieve/enrichment/cvss.py` — 100%
- `docklet/config.py`, `docklet/cgroups.py` — 100%
- `focuslock/blocklist-loader.ts` — 100%
- `TelemetryIngester/Kafka/KafkaMessageSerializer.cs` — 91%

These are all small, deterministic, single-input/single-output. Easy to assert structurally, hard for mutations to hide.

**2. I/O glue and entry points get weak or absent AI tests (0–10%).**
- `Program.cs` (test-slicer Cli) — 0%
- `TimescaleWriter.cs` (open-race-telemetry) — 0%
- `main.ts` (focuslock) — 0%
- `UdpListenerService.cs` — 0%

**3. Output formatters are surprisingly weak.**
- `cvesieve/output.py` — 42%
- `docklet/cli.py` — 51%
- `docklet/registry.py` — 52%

Tests assert "output contains X" instead of "output structure equals Y." Mutations that change formatting subtly survive. Consistent across projects and languages.

**4. Source generators / metaprogramming get the worst tests.**
- `TestSlicer.Generator/TestSlicerGenerator.cs` — 35%
- `TestSlicer.Generator/CategoryAttributeWriter.cs` — 26%

Roslyn syntax-tree manipulation is hard to test (you'd need golden-file snapshots), and the AI shortcuts the assertions.

These four patterns are universal AI blind spots in this dataset. Same shape, same weaknesses, regardless of language or project.

## The proposal: mutation testing as the feedback signal for AI coding agents

This is the part of the post that matters most.

A note on what I can and can't claim here: I don't directly observe the AI's optimisation process. I observe the *output* — tests that pass, coverage that's high, and a mutation score that's noticeably lower than the coverage. The behaviour is consistent with metric-driven optimisation against a leaky proxy (coverage), but I'm not in the model's head. Read the next few paragraphs as: "if the AI's behaviour is well-described by 'optimise for the visible quality signal,' then mutation testing fixes the visible signal." That's a hypothesis the data supports, not a mechanism I've proven.

When a human writes a test, they have an internal model of what the test is supposed to *prove*. They look at their assertion and ask "would this fail if the function returned wrong data, not just no data?" A skilled human pushes back on their own assertions until the answer is yes. AI agents don't do this — at least not reliably, and not without prompting. They write something that exercises the line, the line is now "covered," and the loop closes.

The failure mode in this experiment is consistent with **the agent having no good signal for whether its assertions are doing useful work.** The signals it does have are:

- **Does the test pass?** — Says nothing about whether the test would catch a bug.
- **Does coverage go up?** — The cvesieve result alone shows 91% line coverage and 55% mutation score can coexist; coverage is a leaky proxy.
- **Does it look like a real test?** — Stylistic, not behavioural.

Mutation testing, by contrast, is **the most direct possible signal of "would this test catch a bug."** It literally introduces bugs and checks. Surviving mutants are a precise, machine-readable list of behaviours your tests don't verify, with file and line numbers attached. That's exactly the kind of feedback an agent loop is designed to consume.

The mechanical version of the proposal is barely an idea — it's a script:

1. Agent writes code and tests.
2. Mutation testing runs.
3. Surviving mutants — with file, line, and the specific change that wasn't caught — get fed back to the agent: *"these 30 mutations passed your tests; write tests that fail when these mutations are applied."*
4. Iterate until the score clears whatever bar you've set.

I haven't built this end-to-end yet, but the data here suggests it would close most of the gap. The cvesieve `output.py` survivors aren't subtle — they're things like "removed the entire branch that handles malformed input" or "swapped two field separators." A human reading the diff would say "ah, my test should fail when that's removed; let me add an assertion." There's no reason an agent given the same diff and instruction wouldn't do the same.

This reframes mutation testing from "an audit tool you run manually before shipping" into **"the closed-loop quality signal an AI coding agent should be running every iteration."** Coverage was the wrong signal — it always was, but it was the only one cheap enough to run on every commit, so the industry standardised on it. Mutation testing has historically been too slow for that role. With AI agents that already spend minutes per turn, the cost calculus is different: a 5-minute mutation run between iterations is cheap relative to the agent's own thinking time, and probably worth more than another five minutes of "improve the tests" prompting.

If I were building an AI coding tool today, I'd ship this as a default. Run mutation testing in the agent loop, hand the agent the surviving mutants as targets, iterate. **The 50–60% mutation score in this experiment is what AI tests look like *without* this feedback loop. The benchmark with the loop probably looks very different.** That's the experiment I most want to see run next.

### One important caveat: don't replace one gameable metric with another

The honest objection to the proposal above is that **agents can overfit to mutants** the same way they currently over-optimise for coverage. If you point an agent at a list of surviving mutants and say "make these die," there's a real risk it learns the *shape* of the mutation operators and writes hyper-specific assertions that defeat those exact mutants without actually improving the test suite's robustness against real-world bugs. Mutation testing tools have a finite, known set of operators (boolean flips, arithmetic swaps, constant replacements, boundary changes, etc.). Anything finite and known is, at the limit, gameable.

That's a real risk and it doesn't sink the proposal, but it does shape how to build it well:

- **Use multiple mutation operators and randomise them across iterations** — don't show the agent the same operator set every loop. Stryker.NET supports `Standard`, `Advanced`, and `Complete` mutation levels; rotate them.
- **Don't show the agent the surviving *mutations* directly — show it the surviving *behaviours*.** Instead of "this `>` got changed to `>=` and your tests didn't notice," frame it as "your tests don't distinguish between these two boundary conditions; write a test that does." That keeps the agent fixing semantics, not patching mutation patterns.
- **Combine with property-based testing** ([Hypothesis](https://hypothesis.readthedocs.io/) for Python, [FsCheck](https://fscheck.github.io/FsCheck/) for .NET, [fast-check](https://fast-check.dev/) for JS). Property-based tests force the agent to reason about invariants rather than specific input/output pairs, which is much harder to overfit.
- **Hold out a fraction of mutants from the agent** as a validation set — measure the *true* mutation score on the held-out set rather than the set the agent was trained against.

Without these guards, "mutation testing in the loop" is a stronger feedback signal than coverage but still a finite metric the agent can route around. With them, it gets closer to actually measuring "would this test catch a real bug" — which was the goal in the first place.

## Practical recommendations

If you're using AI to write code and tests today:

1. **Run mutation testing on AI-written test suites.** Coverage isn't enough. Stryker.NET, mutmut, and Stryker.JS all run in 5–30 minutes for projects this size — well within "do it once before shipping" budget.

2. **Look at the coverage→mutation gap per file** — it tells you exactly which files have weak assertions. Cvesieve's `output.py` had a 49-point gap; that one file is where you'd start.

3. **Push the AI to factor code for testability before writing tests.** Extract pure functions out of files with side effects so the testable logic isn't tangled with the un-testable scaffolding. This addresses both failure modes at once: shrinks the "skipped" bucket and reduces "tested but loosely" by giving the AI well-shaped targets.

4. **Force the AI to mock at unit level for I/O-heavy code** rather than letting it defer to integration tests. Tests will be both faster and stronger. Mocking forces precise assertions on `mock.call_args` — there's no looser way. Docklet's 75% score is the proof point.

5. **Manually review tests for output formatters, entry points, and HTTP-y code** — the AI consistently undertests these regardless of project, language, or model. They're the universal blind spots in this dataset.

6. **Don't mechanically chase 100% mutation score.** Some files (Electron lifecycle, thin SDK wrappers, declarative wiring) shouldn't be unit-tested. Aim for "every file with testable logic clears 80%, and the rest is deliberately untested with a comment explaining why."

7. **For production-quality work, build the mutation-in-the-loop pattern.** This is the most important recommendation here — see the proposal section above. Don't run mutation testing once before shipping; run it between agent iterations and feed the surviving mutants back as targets.

## A loud caveat: this is observational, not a controlled study

I want to be careful not to oversell this. Everything in this post is observational — patterns across five projects I'd already built. The numbers are real, but the sample is small and confounded:

- **n=5 projects, single sample per condition.** No replicates. Session-to-session variance in AI output could easily move any individual score by 10+ points.
- **One model (Claude Opus 4.6).** I have no data on how Sonnet, Haiku, GPT-5, Gemini, or other models write tests. The patterns here might be Opus-specific, model-family-specific, or general — I can't tell from this data.
- **One author (me).** My prompting style, choice of when to push back on the AI, and willingness to accept "looks fine" are all baked into every result.
- **No prompting for stronger assertions.** I gave the AI what I wanted built and accepted the code and tests it produced — I didn't iterate on test quality, didn't ask it to strengthen assertions, didn't push back on substring matches. So these scores reflect *AI tests at default prompt effort*, not *AI tests at maximum effort*. A run with explicit "write tests that would catch off-by-one errors and field-order swaps" prompting could land very differently, and I'd want to see that experiment before generalising.
- **Different problem domains.** I argued that problem shape matters more than test-author quality. That cuts both ways — the cross-project comparison isn't apples-to-apples and the "AI tests cluster around 50–60%" headline is itself shape-dependent.

So please read this as **"here's what I saw, here's what looks like it might be a pattern, here's what would need to be tested before claiming any of it is true in general."** Not "AI tests are X% good." That number doesn't exist yet.

The mutation-as-feedback proposal in particular is a hypothesis based on the data, not a result. I haven't actually built or run the loop. The next thing I want to do is exactly that.

## What I'd want to see next

- **Build the mutation-in-the-loop system end-to-end.** Take cvesieve as the test bed. Wire up: agent writes tests → mutmut runs → survivors fed back to agent → repeat. Measure how many iterations it takes to clear 80%, and whether the resulting tests are good or merely shaped to defeat the specific mutants.
- **Across multiple models.** Run the same projects through Claude Sonnet, Haiku, GPT-5, Gemini Pro. Does the 50–60% cluster hold across model families, or is it Opus-specific? Do the *blind spots* (entry points, formatters, source generators) repeat across models, suggesting they're an "AI test" failure mode, or are they model-specific?
- **Multiple sessions per project.** Same model, same prompt, same project, ten independent builds. How much of the variance between projects above is real signal vs. session noise?
- **Same projects, human-written tests** — to isolate "AI wrote these" from "the code is what it is." Prediction: human tests would be slightly higher on the weak files (because humans push back on lazy assertions when reviewing) but not dramatically.

## Methodology and reproducibility

- All five projects had passing test suites before mutation runs (fairness baseline).
- Stryker.NET 4.14.1 used for C# projects with default mutation level "Standard".
- mutmut 2.5.1 used for Python projects (mutmut 3.5 has a path bug; downgraded).
- Stryker.JS used for TypeScript with vitest runner and TypeScript checker.
- open-race-telemetry: integration tests excluded via `test-case-filter "FullyQualifiedName!~Integration"`.
- Coverage measured separately via Coverlet (`dotnet test --collect:"XPlat Code Coverage"`), `pytest-cov --cov-branch`, and `vitest --coverage`.
- LOC counted as both total and "sloc" (non-blank, non-comment lines).
- Per-project breakdowns and raw artifacts preserved at `/experiment-docs/mutation-testing/`.

---

*If you've run mutation testing on AI-built projects — or, especially, if you've actually built the mutation-in-the-loop system I'm proposing — I'd love to compare notes.*
