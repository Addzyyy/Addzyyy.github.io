---
layout: post
title: "How Good Are AI-Written Tests? I Mutation-Tested Five AI-Built Projects to Find Out"
date: 2026-04-28
---

When AI writes both your code and your tests, who's auditing the auditor? The tests pass. Coverage looks great. But are those tests actually catching bugs, or just exercising lines?

I ran mutation testing on five projects I'd built with Claude Opus 4.6 to find out. The results surprised me, but the most useful takeaway isn't a benchmark number — it's a proposal for what AI coding agents should be using as their quality feedback signal.

## 🔥 The headline finding — read this if nothing else

> *(Mutation testing introduces deliberate small bugs into your code and checks whether your tests catch them. Coverage tells you which lines were touched. Mutation testing tells you whether the touch was load-bearing.)*
>
> 📊 **Caveat first, so you can calibrate the rest:** observational, not controlled. Five projects, single sample each, single model, single author. The patterns repeated reliably, but n=5 is a starting point, not a benchmark. Read everything as suggestive, not proven.
>
> **Across all five projects, the AI-written test suites looked great by every traditional metric — and let roughly half the bugs through anyway.**
>
> **What looked fine:** 100% green tests, 76–92% line coverage, 162–171 tests on the largest projects.
>
> **What mutation testing actually showed:**
> - **No project hit 80%** (the standard "healthy" bar). Docklet came closest at 75%; the rest sat between 41% and 69%, most clustering around 55%.
> - **The cvesieve result was the starkest:** 91% line coverage, 55% mutation score. A 36-point gap. Plain language: *"your tests touched every line but missed nearly half the bugs."*
> - **The pattern was consistent** across three languages (Python, C#, TypeScript) and three different mutation testing tools. Not a one-off.
>
> ⚠️ **If you trust green CI as your safety net for AI-written code: based on these five projects, your tests can look strong by coverage and still miss a large fraction of realistic faults.** They exercise the code without strongly verifying its behaviour. (A surviving mutant doesn't necessarily mean a test is "useless" — some are equivalent or unrealistic. But a sustained 30+ point gap repeated across projects is a real quality signal, not a measurement artefact.)
>
> **Two findings underpin the rest of this post.** First, on covered code: the data is consistent with the AI taking the metric it can see (does the test pass? does it touch the line?) as a proxy for the metric it can't (does this assertion verify behaviour?). The shortest version: **AI writes the weakest assertion the contract of the code permits, by default.** When the contract is tight, that's enough. When it's loose, half the bugs walk past.
>
> Second, on uncovered code: **AI is implicitly making architectural decisions — about what is unit-testable and what isn't — without making the corresponding refactoring decisions to support its own testing strategy.** It inlines pure logic with side effects, then classifies the whole file as "needs runtime → won't test." That's arguably the more durable critique, and I unpack it below.
>
> The post ends with what I think is the most useful proposal: **mutation testing should be the feedback signal in AI coding agent loops, not coverage** — with a real overfitting caveat I get to at the end.

---

## The setup

All five projects were built with **[Claude Code](https://claude.com/claude-code)** + **Claude Opus 4.6**. Code and tests were AI-written end-to-end — by which I mean: I described what I wanted, accepted the AI's code and tests as produced, and didn't iterate on test quality. No human-authored test code, and **no prompting for stronger assertions, exact-match output, or off-by-one tests.** This is "AI tests at default prompt effort."

| Project | Language | What it does |
|---|---|---|
| **docklet** | Python | Minimalist Docker clone — Linux namespaces, cgroups, container management |
| **test-slicer** | C# | Test impact analysis tool with three sub-projects (Core / Cli / Roslyn Generator) |
| **cvesieve** | Python | CVE scanner output filter with NVD/EPSS/KEV enrichment |
| **open-race-telemetry** | C# | F1 telemetry pipeline: UDP → Kafka → TimescaleDB |
| **focuslock** | TypeScript / Electron | Calendar-synced website blocker desktop app |

Tools: [Stryker.NET](https://stryker-mutator.io/) for C#, [mutmut](https://github.com/boxed/mutmut) for Python, [Stryker.JS](https://stryker-mutator.io/docs/stryker-js/introduction/) for TypeScript. All tools were run with their **default mutation operator sets** — no customisation across tools, which means cross-tool comparisons should be read with that caveat.

## What is mutation testing, briefly

The tool takes your source code and makes small, semantically-meaningful changes: flip `>` to `>=`, change `True` to `False`, replace a constant, drop a `not`. Then it runs your test suite. If a test fails, the mutant is "killed." If all tests still pass, the mutant "survived." Mutation score = % killed.

**Coverage tells you whether tests *touched* a line. Mutation testing tells you whether tests would *catch a bug* on that line.** That distinction is the entire point.

## The numbers

| Project | Mutation Score | Line Cov | Cov→Mut Gap | Test:Code |
|---|---:|---:|---:|---:|
| **docklet** | **74.97%** | 91.55% | 17 pts | 2.52 |
| test-slicer (Core) | 68.64% | 76.12% | 8 pts | 2.09 |
| cvesieve | 55.23% | 90.73% | **36 pts** | 1.15 |
| test-slicer (Cli) | 54.59% | 54.57% | 0 pts | 1.69 |
| test-slicer (Generator) | 41.58% | 65.80% | 24 pts | 1.82 |
| focuslock ¹ | 32.97% / **85.71%** | 38.86% | — | 1.10 |
| open-race-telemetry ¹ | 21.65% / **79.02%** | 47.18% | — | 0.92 |

> ¹ **Important:** these two projects have *two scores* — "all mutants" / "covered code only." Both have files the AI deliberately left out of the unit test scope, which drags the headline number down. The bolded second number is what those test suites actually achieve on the code they tested. This is the central tension of failure mode 2 below.

There turn out to be two distinct flavours of this — one a real problem, one partly a problem and partly the right answer.

## Failure mode 1: Weak assertions on covered code

This is the cvesieve story, and it's the dominant failure mode in the data. Cvesieve parses Trivy JSON, enriches with NVD/EPSS/KEV data, and formats reports — lots of data transformation with optional fields. The kind of code where a test can get away with this:

```python
# format_severity returns "CRITICAL"/"HIGH"/"MEDIUM"/"LOW" based on CVSS score
result = format_report(cves)  # cves contains a high-severity entry
assert "HIGH" in result
```

That assertion lets dozens of mutations slip past. A mutation that flips `score >= 7.0` to `score > 7.0` changes the boundary — a CVE scoring exactly 7.0 now formats as "MEDIUM" instead of "HIGH." But because the test batch contains *other* high-severity CVEs, "HIGH" is still in the output somewhere. Test passes. Coverage rises. Mutation survives. cvesieve had **163 mutations on `output.py` survive** by patterns very similar to this.

The same shape appears in **test-slicer Cli** (55% / 55%) and **test-slicer Generator** (42% / 66%). Wherever the AI wrote tests for code with loose contracts — text formatting, multi-field structures, configurable output — the assertions tended to be loose. Tests that effectively check "did this not throw, and is the keyword somewhere in the output" rather than "is the output exactly what it should be."

The fix is straightforward to describe and tedious to do by hand: assert on full output structure, not substrings; on exact mock call args, not "was this called"; on returned data values, not just "did this return." A signal that catches lazy assertions automatically would be more useful than reviewer effort — that's the proposal in the back half.

## Failure mode 2: Files the AI skipped entirely

**focuslock** and **open-race-telemetry** report headline mutation scores of 33% and 22%. But look at the second number: 86% and 79% — the score *on the code that was tested*. Both clear the 80% bar; focuslock's is the highest covered-code score in the dataset.

What's happening: the AI made a deliberate split. open-race-telemetry's I/O classes got TestContainers integration tests (real Kafka + Postgres in Docker) that I excluded from this run; focuslock's `main.ts` (Electron bootstrap) and `google-calendar-api.ts` (OAuth wrapper) were left essentially untested as needing-a-real-runtime. The CI dashboard doesn't tell you that — "all my tests pass" doesn't mean "all my code is tested."

### When 0% is the right answer, and when it isn't

Some of those 0% scores are correct. Mocking `googleapis` to test that you call `googleapis` correctly is busywork. Most of `main.ts` is genuine Electron wiring. **But** interleaved with that wiring in `google-calendar-api.ts` is response-mapping logic with real edge cases (default values, date parsing, filtering). And `main.ts` contains `loadSettings()`, `saveSettings()`, file I/O with JSON parsing and error handling — the project's #1 stated reliability concern is hosts-file crash recovery, which lives there too. All untested.

The AI inlined the testable parts with the I/O and skipped the lot.

## The deeper finding: AI is doing architecture by default

> **AI is implicitly making architectural decisions — about what is unit-testable and what isn't — without making the corresponding refactoring decisions to support its own testing strategy.**

A human would notice "this file has a pure mapping function inside an Electron lifecycle handler" and pull the mapping out so it could be tested in isolation. The AI doesn't. It writes the inline version, classifies the whole file as un-unit-testable, and moves on.

- **AI is doing architecture by default**, every time it picks a function boundary.
- **It's not optimising those boundaries for testability** — it's optimising for "does this work end-to-end."
- **The cost surfaces as 0% mutation scores on files that ought to have 80%+** on the half of their content that's pure logic.

The fix isn't more tests. It's prompting the AI to **separate side-effectful code from pure logic before writing tests** — as a structural step before any test code at all. Don't mechanically chase 100%, either. The right target is "every file with testable logic clears 80%; wiring and SDK-shim files are deliberately untested with a one-line comment explaining why."

## Docklet isn't an outlier — it's a near-control-group

Docklet vs. cvesieve is the closest thing this experiment has to a controlled comparison: same model, same author, same language (Python), same tool (mutmut), 171 vs. 169 tests, ~0.83 mutants per sloc on both. **Test count and mutant density are the right controls here because they hold *test investment* and *mutation surface area* fixed** — meaning the score difference can't be explained by "the AI wrote more tests" or "one project had more places to inject bugs."

**The thing that varied was the shape of the code.** The mutation score moved from 55% to 75% — a 20-point swing on the same model and the same author. That's about as close to causal evidence as observational data with this sample size gets.

The headline isn't really *"AI tests are weak."* It's *"AI tests are weak on weakly-shaped code."* I see three plausible factors. **Honest caveat up front: these are very likely correlated manifestations of the same underlying property — code that talks to the kernel via narrow, exact-args APIs.** I can't separate their independent contributions from this data; treat the three as one finding presented from three angles.

### Factor 1: Tight contracts force tight assertions

Docklet's code is dominated by "call this syscall with these exact arguments":

```python
os.unshare(CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET)
subprocess.run(["ip", "link", "add", veth_name, "type", "veth", ...])
```

When you mock these, your assertions *have to* be precise — exact flag value, exact arg list. There's no looser way. So a mutation flipping `CLONE_NEWPID` or `"add"` to `"remove"` dies instantly. The inverse of cvesieve's `output.py`.

### Factor 2: Branch density

Mutants hide best in branches. Docklet has fewer per line:

| Project | Branches / sloc |
|---|---:|
| **docklet** | **0.144** |
| open-race-telemetry | 0.230 |
| cvesieve | 0.205 |
| test-slicer Core | 0.362 |
| test-slicer Cli | 0.668 |
| test-slicer Generator | 0.696 |

**Caveat:** test-slicer Cli has **3× cvesieve's branch density at virtually identical mutation score (54.59% vs. 55.23%).** So branch density contributes but doesn't determine — it's not a clean monotonic predictor.

### Factor 3: Mocking discipline (counterintuitively)

Docklet *had* to mock everything — you can't `clone()` into namespaces from pytest as non-root. Conventional wisdom says heavy mocking weakens tests, but here, **forced mocking made tests stronger**: you don't get to write `assert "ok" in result` against a mock. You assert on exact `mock.call_args`.

This contradicts what open-race-telemetry implies (defer I/O to integration tests because they're "stronger"). A well-mocked unit test of a syscall is plenty strong — and you don't need Docker to make it so.

### The takeaway

The AI didn't suddenly get better between projects. The code stopped letting it be sloppy. Tight contracts *force* tight tests; loose contracts *allow* tight tests but don't require them, and the AI consistently doesn't write the tight version when a loose version satisfies the same coverage metric.

**Read the comparison chart not as "docklet's AI tests are good and cvesieve's are bad" but as "docklet's code didn't *let* the AI write bad tests; cvesieve's did, and the AI took the offer."** The right question for any AI-written test suite isn't the headline mutation score — it's "did the AI find loose assertions to write here, and is that hiding a coverage→mutation gap?"

## Patterns that held across every project

Some failure modes were consistent everywhere:

- **Pure logic gets strong AI tests (70–100%).** `cvss.py`, `config.py`, `cgroups.py`, `blocklist-loader.ts`, `KafkaMessageSerializer.cs`. Small, deterministic, single-input/single-output. Hard for mutations to hide.
- **I/O glue and entry points get weak or absent AI tests (0–10%).** `Program.cs`, `TimescaleWriter.cs`, `main.ts`, `UdpListenerService.cs`.
- **Output formatters are surprisingly weak.** `cvesieve/output.py` — 42%. `docklet/cli.py` — 51%. `docklet/registry.py` — 52%. Tests assert "output contains X" instead of "output structure equals Y."
- **Source generators / metaprogramming get the worst tests.** `TestSlicerGenerator.cs` — 35%. `CategoryAttributeWriter.cs` — 26%. Roslyn syntax-tree manipulation needs golden-file snapshots; the AI shortcuts it.

These are universal blind spots in this dataset, regardless of language or project.

## The proposal: mutation testing as the feedback signal for AI coding agents

This is the part of the post that matters most.

Caveat on what I can and can't claim: I don't observe the AI's optimisation process. I observe the *output* — tests that pass, high coverage, lower mutation score. The behaviour is consistent with metric-driven optimisation against a leaky proxy (coverage), but I'm not in the model's head. Read this as: *"if the AI's behaviour is well-described by 'optimise for the visible quality signal,' then mutation testing fixes the visible signal."* A hypothesis the data supports, not a mechanism I've proven.

When a human writes a test, they have an internal model of what the test is supposed to *prove* and push back on their own assertions. AI agents don't reliably produce output consistent with this — at least not without prompting. They write something that exercises the line, the line is now "covered," and the loop closes.

The signals an agent actually has are:

- **Does the test pass?** — Says nothing about whether it would catch a bug.
- **Does coverage go up?** — Coverage is a leaky proxy (cvesieve: 91% / 55%).
- **Does it look like a real test?** — Stylistic, not behavioural.

Mutation testing, by contrast, is **the most direct possible signal of "would this test catch a bug."** Surviving mutants are a precise, machine-readable list of behaviours your tests don't verify, with file and line numbers attached. Exactly what an agent loop is designed to consume.

The mechanical proposal is barely an idea — it's a script:

1. Agent writes code and tests.
2. Mutation testing runs.
3. Surviving mutants get fed back: *"these 30 mutations passed your tests; write tests that fail when these mutations are applied."*
4. Iterate until the score clears your bar.

The cvesieve `output.py` survivors aren't subtle — they're things like "removed the entire branch that handles malformed input" or "swapped two field separators." A human reading the diff would say "ah, my test should fail when that's removed." There's no reason an agent given the same diff and instruction wouldn't do the same.

Coverage was the wrong signal — but it was the only one cheap enough to run on every commit, so the industry standardised on it. With AI agents that already spend minutes per turn, a 5-minute mutation run between iterations is cheap relative to the agent's own thinking time. **The 50–60% score is what AI tests look like *without* this loop. The benchmark with the loop probably looks very different.**

### One important caveat: don't replace one gameable metric with another

The honest objection: **agents can overfit to mutants** the way they currently over-optimise for coverage. Mutation operators are a finite, known set; anything finite and known is, at the limit, gameable.

Guards that shape the proposal:

- **Rotate operator sets across iterations** — Stryker.NET supports `Standard` / `Advanced` / `Complete`; don't show the agent the same set every loop.
- **Don't show *mutations*; show *behaviours*.** Not "this `>` got changed to `>=` and your tests didn't notice" — instead "your tests don't distinguish between these two boundary conditions; write a test that does." Keeps the agent fixing semantics, not patching mutation patterns.
- **Combine with property-based testing** ([Hypothesis](https://hypothesis.readthedocs.io/), [FsCheck](https://fscheck.github.io/FsCheck/), [fast-check](https://fast-check.dev/)). Forces reasoning about invariants, much harder to overfit.
- **Hold out a fraction of mutants from the agent** as a validation set.

Without these guards, "mutation testing in the loop" is a stronger signal than coverage but still routable-around. With them, it's closer to actually measuring "would this test catch a real bug."

## Practical recommendations

If you're using AI to write code and tests today, the two that move the needle most:

1. ⭐ **Run mutation testing on AI-written test suites.** Coverage isn't enough. Stryker.NET, mutmut, and Stryker.JS run in 5–30 minutes for projects this size — within "do it once before shipping" budget.
2. ⭐ **Push the AI to factor code for testability *before* writing tests.** Extract pure functions out of files with side effects so testable logic isn't tangled with un-testable scaffolding. Addresses both failure modes at once.

Then:

3. **Look at the coverage→mutation gap per file** — it tells you which files have weak assertions. cvesieve's `output.py` had a 49-point gap; that's where you'd start.
4. **Force the AI to mock at unit level for I/O-heavy code** rather than letting it defer to integration tests. Mocking forces precise assertions on `mock.call_args`. Docklet's 75% is the proof point.
5. **Manually review tests for output formatters, entry points, and HTTP-y code** — the AI consistently undertests these regardless of project.
6. **Don't mechanically chase 100% mutation score.** Some files (Electron lifecycle, SDK shims) shouldn't be unit-tested.
7. ⭐ **For production-quality work, build the mutation-in-the-loop pattern.** The most important recommendation here — see the proposal above.

## A loud caveat: this is observational, not a controlled study

- **n=5 projects, single sample per condition.** Session-to-session variance could move any individual score by 10+ points.
- **One model (Claude Opus 4.6).** No data on Sonnet, Haiku, GPT-5, Gemini, or others.
- **One author (me).** My prompting style and willingness to accept "looks fine" are baked in.
- **No prompting for stronger assertions.** I gave the AI what I wanted built and accepted what it produced. These scores reflect *AI tests at default prompt effort*, not *maximum effort*. A run with explicit "write tests that catch off-by-one errors and field-order swaps" prompting could land very differently — that's the experiment I most want to see next.
- **Different problem domains.** The cross-project comparison isn't apples-to-apples and the "50–60%" headline is itself shape-dependent.

Read this as **"here's what I saw, here's what looks like a pattern, here's what would need to be tested before claiming it's true in general."** Not "AI tests are X% good." That number doesn't exist yet.

## What would change my mind

The two main hypotheses in this post are *(a)* the laziest-assertion claim and *(b)* mutation-in-the-loop fixes it. Specific results that would falsify or weaken them:

- **Lazy-assertion hypothesis falsified if:** a run with explicit "write strict assertions, exact-match output, boundary tests" prompting closes the coverage→mutation gap to under 10 points on cvesieve `output.py` without any other changes. That would mean the AI *can* write tight tests when asked, and the failure mode is prompt-shaped, not model-shaped.
- **Mutation-in-the-loop hypothesis falsified if:** an agent given surviving mutants as targets converges on hyper-specific assertions that defeat the listed mutants but fail on a held-out set, *even with the framing-as-behaviours and operator-rotation guards above*. That would mean the metric is too gameable to be the loop signal.
- **Project-shape hypothesis weakened if:** repeating cvesieve and docklet with five fresh sessions each shows mutation scores varying by ±15 points within the same project. That would mean session noise dominates code-shape effect at this sample size.
- **Universal blind spots claim weakened if:** a different model family (GPT-5, Gemini Pro) doesn't show the same entry-point / output-formatter / source-generator pattern.

## Methodology and reproducibility

- All five projects had passing test suites before mutation runs.
- **All mutation tools were run with default operator sets**, no customisation. Cross-tool comparisons should be read with this caveat.
- Stryker.NET 4.14.1 — default level "Standard."
- mutmut 2.5.1 (mutmut 3.5 has a path bug; downgraded).
- Stryker.JS with vitest runner and TypeScript checker.
- open-race-telemetry: integration tests excluded via `test-case-filter "FullyQualifiedName!~Integration"`.
- Coverage measured separately via Coverlet, `pytest-cov --cov-branch`, and `vitest --coverage`.
- LOC counted as both total and "sloc" (non-blank, non-comment).
- Per-project breakdowns and raw artifacts at `/experiment-docs/mutation-testing/`.

---

*If you've run mutation testing on AI-built projects — or, especially, if you've built the mutation-in-the-loop system I'm proposing — I'd love to compare notes.*
