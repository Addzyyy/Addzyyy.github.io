---
layout: post
title: "Spec-Driven Development: How AI and Formal Methods Will Change Software Engineering"
date: 2026-04-15
---

*The code you write is about to become irrelevant. The spec is all that matters.*

---

For decades, formal methods have been the gold standard of software correctness. If you want mathematical proof that your code does what it's supposed to do, formal verification is the answer. Airbus has used it since the A320. NASA uses it for mission-critical systems. Cryptographers use it to prove their protocols are sound.

And almost nobody else uses it. Because it's too expensive.

That's about to change.

## The Cost Problem

Formal verification has always had a strong value proposition technically. The reason it stayed niche is cost, not capability. Estimates typically put formal verification at 3-5x the development cost of conventional approaches. That's why only domains where failure is catastrophic — aviation, nuclear, cryptography — could justify it. Everyone else writes tests and ships.

The cost comes from two places: writing formal specifications and constructing proofs. Both require specialised expertise. Both are painstaking manual work. Both are exactly the kind of structured, pattern-heavy tasks that large language models are getting good at.

## AI Changes the Economics

Here's the key insight: AI makes formal methods cheaper by an order of magnitude.

LLMs are already decent at translating natural-language intent into formal specifications — TLA+, Alloy, Dafny contracts, Lean theorems. They're getting better fast. And critically, the verification side doesn't need AI at all. Proof checkers like Coq, Lean, Isabelle, and Frama-C have small trusted kernels that are themselves formally verified or heavily audited. The AI sits entirely outside the trusted computing base.

This creates a powerful separation: **AI generates, math verifies.** The trust doesn't depend on the AI being correct. It depends on the verifier being correct — and the verifier is a small, well-understood piece of mathematics that has been scrutinised for decades.

Generating correct code is hard. Checking a proof is easy. That asymmetry is the whole game.

## From Writing Code to Writing Specs

This shifts what developers actually do. Instead of writing code and hoping tests catch the bugs, you describe what the system should do. AI drafts a formal specification. You review it. AI generates code and a proof that the code satisfies the spec. A verified tool chain checks the proof.

The human moves up the abstraction ladder. Instead of being a coder who occasionally thinks about requirements, you become a **spec reviewer** — which is where human judgment adds the most value. Humans are better at "is this what I actually want?" than "did I handle the edge case on line 4,217?"

I call this **spec-driven development**. The spec becomes the versioned artifact. The code becomes a derived artifact — generated, proved correct, disposable. You don't edit it, you don't review it, you regenerate it. Just like nobody edits a Docker image or a compiled binary. You change the Dockerfile or the source and rebuild.

## The Workflow

Here's what spec-driven development looks like in practice:

**1. Human describes intent in natural language.**

"I need a controller that keeps the aircraft within its flight envelope. It should prevent the pilot from exceeding angle of attack limits, respect structural load limits, and degrade gracefully if a sensor fails."

**2. AI drafts a formal spec.**

The AI translates that into properties:

- Angle of attack never exceeds 15 degrees in clean configuration
- Load factor stays between -1g and +2.5g
- If two of three sensors disagree, use the median
- If all sensors fail, revert to alternate law, then direct law

**3. Human reviews the spec.**

The domain expert — a flight dynamics engineer, not a programmer — reads the properties and catches things: "15 degrees is the clean wing limit. With flaps extended it's 20." "You forgot about icing conditions." "Direct law isn't the right degradation path."

This is where human expertise is irreplaceable, but the task is manageable — reviewing a page of properties, not thousands of lines of code.

**4. Simulate and model check.**

Run the spec through a model checker. It explores thousands of scenarios and surfaces counterexamples:

"Scenario: Sensor 1 fails, then sensor 2 reads erroneously high due to icing. The median vote selects the wrong value. Aircraft enters stall."

The human says "we need a plausibility check — if a reading changes by more than 5 degrees in one second, flag it as failed." The spec tightens. The checker runs again. Repeat.

**5. AI generates code + proof.**

Once the spec is stable, AI generates an implementation along with proof obligations. A verified tool chain checks that the code satisfies every property. The output is code plus a machine-checked certificate.

**6. Validate against the real world.**

Put it in a simulator. Run hardware-in-the-loop tests. This isn't verifying the code — the proof did that. This is validating the spec against physical reality. When discrepancies are found, update the spec, regenerate, re-prove. Cheap, because the AI does the work.

## The Techniques That Make Specs Better

The obvious concern: what if the spec is wrong? You get a perfectly verified implementation of the wrong thing. This is the irreducible hard problem — verification proves you built the thing right, never that you built the right thing.

But there are well-established techniques for catching bad specs, and AI accelerates all of them:

**Executable specs.** If your spec is in something like TLA+ or Alloy, you can simulate it before any code exists. The model checker explores every reachable state and surfaces surprising ones. You're debugging your thinking, not your code.

**Property-based thinking.** Instead of specifying exact input-output behavior ("return 42 when X"), specify invariants and boundaries ("balance never goes negative", "altitude stays within envelope"). These are harder to get wrong because they're closer to how domain experts actually think. A banker doesn't think in function calls. They think "we never give out money we don't have."

**Counterexample-driven refinement.** The model checker finds a scenario the spec allows that shouldn't happen. The human says "that's wrong." The spec tightens. Each round surfaces requirements you didn't know you had — not by imagining them, but by seeing concrete scenarios where the spec produces bad outcomes.

**Domain-specific spec languages.** A spec language tailored to flight control or financial settlement constrains what you can express, which prevents entire categories of specification errors. You lose the ability to say arbitrary things, but that's the point — most of those things would have been mistakes.

With AI in the loop, the feedback cycle of write-spec, simulate, review, refine goes from months to minutes. An engineer who goes through that refinement cycle 50 times in a week develops better intuition for what they're missing than one who ships two spec revisions a year.

## It's Not Just for Avionics

The real impact isn't making aerospace verification slightly cheaper. It's bringing formal correctness to everything:

- **Medical devices** — currently under-verified relative to the risk
- **Automotive** — autonomous driving stacks that are tested but not proved
- **Financial systems** — where a subtle bug can move billions
- **Smart contracts** — where bugs are literally irreversible
- **CI/CD pipelines** — deceptively complex, constant source of security incidents
- **Ordinary software** — even a web API could have its core logic verified

Take something as mundane as a GitHub Action. "Run tests on PR, require approval before deploying to production." Simple, right? But the real requirements include: approval must be invalidated when new commits are pushed, secrets must never be exposed in steps that run untrusted code, the workflow shouldn't run on forks with write permissions, deploys must be atomic. These are the kind of edge cases that model checkers find in seconds and humans miss routinely.

The point of cheap formal methods isn't just that you can use them on small things. It's that the small things are where most real-world bugs live.

## What Changes

**Version control changes.** You diff specs, review specs, merge specs. A PR becomes "I changed this property from X to Y" — not "I modified 47 files across 3 services." Every commit directly expresses a requirement change. The git history becomes a readable record of how your understanding of the problem evolved.

**Code review changes.** Implementation-level debates — code style, naming, clever versus readable — become irrelevant. Nobody reviews generated code. Review is about intent: does this spec capture what we actually need?

**Technical debt changes.** You don't accumulate it in code, because the code is regenerated. You accumulate it in specs — properties that were good enough at the time but no longer reflect reality. Spec debt replaces tech debt.

**Certification changes.** Instead of telling a regulator "we tested 10,000 scenarios and they all passed," you say "here is a mathematical proof that this property holds in all scenarios, and here is evidence that the properties match the real system." The Airbus verification process that takes years could potentially be compressed dramatically.

**The job changes.** Software engineering starts to look more like engineering engineering — you produce specifications, and the fabrication is automated and certified.

## The Progression

Look at the history of what developers version control:

- **1970s:** Machine code
- **1980s:** Source code
- **1990s:** Source code + build scripts
- **2000s:** Source code + infrastructure config
- **2010s:** Source code + containers + pipelines
- **Next:** Specs. Code drops out entirely.

Each step moved the human further from the machine and closer to intent. Each layer we automated, we stopped worrying about. Nobody manually manages memory allocation anymore. Nobody hand-writes assembly. Nobody configures servers by hand. Each time, people said "you'll always need a human for that." Each time, they were wrong.

Code might just be the next layer that disappears from human concern.

## The Irreducible Human Part

None of this eliminates the need for human judgment. The chain is:

**Real world → Human understanding → Spec → Code → Proof**

AI and formal methods make the right side of that chain airtight. The left side — understanding what the world actually needs — remains a human problem. You still need domain experts who can look at a spec and say "that's not what the aircraft should do" or "that's not how settlement works."

But that's a healthy division of labour. Humans handle intent. Machines handle correctness. And the spec is the contract between the two.

---

*Spec-driven development isn't a framework or a tool. It's the natural consequence of making formal verification cheap. When proving correctness costs less than writing tests, everything changes.*
