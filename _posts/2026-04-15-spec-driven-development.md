---
layout: post
title: "Spec-Driven Development: How AI and Formal Methods Will Change Software Engineering"
date: 2026-04-15
---

*The spec is the source code. The code is a build artifact.*

---

## The Bottleneck Nobody's Talking About

AI writes code faster than humans can review it. That's the new reality. Copilot, Claude, GPT — they generate thousands of lines in seconds. And somewhere a developer is squinting at a diff trying to decide if it's correct.

That doesn't scale. The bottleneck in software is no longer writing code. It's *verifying* code. And the gap is widening every day.

So what if you didn't have to review the code at all? What if a machine could *prove* it's correct?

That's not hypothetical. The technology exists. It's called formal verification. And AI is about to make it cheap enough for everyone.

## Formal Methods: Powerful but Expensive — Until Now

Formal verification gives you mathematical proof that code does what it's supposed to do. Not "we tested a bunch of cases and they passed." Proof. For all inputs. For all states. Always.

Airbus has used it since the A320. NASA uses it for mission-critical systems. Amazon used TLA+ to find critical bugs in S3 and DynamoDB that no amount of testing would have caught. The technique works.

And almost nobody else uses it. Because writing formal specs and proofs has always cost 3-5x more than just writing the code. Only industries where failure kills people or loses billions could justify it. Everyone else writes tests and ships.

AI changes this equation overnight. LLMs are already decent at translating natural language into formal specifications — TLA+, Alloy, Dafny, Lean. And here's what matters: **the verification side doesn't need AI at all.** Proof checkers like Coq, Lean, and Isabelle have small trusted kernels that have been scrutinised for decades. The AI sits entirely outside the trusted computing base.

**AI generates. Math verifies.** The trust doesn't depend on the AI being correct. It depends on the verifier being correct — and the verifier is a small, well-understood piece of mathematics.

Generating correct code is hard. Checking a proof is easy. That asymmetry is the whole game.

## The Spec Is the Source Code

This changes what developers actually do. Instead of writing code and hoping tests catch the bugs, you write a spec — a set of properties your system must satisfy. AI generates the code and a proof. A verified tool chain checks the proof. If it passes, the code is correct. Period.

The human moves up the abstraction ladder. Instead of coding, you become a **spec reviewer**. Humans are better at "is this what I actually want?" than "did I handle the edge case on line 4,217?"

I call this **spec-driven development.**

The spec is the versioned artifact. The code is derived — generated, proved correct, disposable. You don't edit it. You don't review it. You regenerate it. Just like nobody edits a Docker image. You change the Dockerfile and rebuild.

## What a Spec Actually Looks Like

This isn't abstract. Here's a real spec for a bank transfer system:

```
P1 — Conservation of money:
  accountA + accountB = constant (before and after every transfer)

P2 — Non-negativity:
  All balances >= 0, always

P3 — Sufficient funds:
  Transfer succeeds only if source balance >= transfer amount

P4 — Correct debit:
  On success, source decreases by exactly the transfer amount

P5 — Correct credit:
  On success, destination increases by exactly the transfer amount

P6 — No side effects on failure:
  If the transfer fails, nothing changes

P7 — Valid amount:
  Transfer amount must be positive

P8 — Transfer limit:
  No single transfer exceeds 10,000
```

That's it. That's what you review. It's small. It's readable. A banker could review it — no programming knowledge required. And from this, AI generates a verified implementation where every property is mathematically guaranteed.

Want to add a feature? Add P9. Regenerate. Re-verify. You don't touch code. You don't figure out where to add an if statement. You state what you want and the machines handle the rest.

## The Workflow

**1. Human describes intent in natural language.**

"I need a bank transfer function. It should only succeed if the source has enough funds. Total money should never change. No balance goes negative. Cap transfers at 10,000."

**2. AI drafts human-readable properties.**

The AI translates your intent into plain English properties — P1 through P8 above. No code. No syntax. Just clear statements of what must be true.

**3. Human reviews the properties in plain English.**

This is where domain expertise matters. A finance person reads the properties and catches things: "You forgot about transfer fees." "What about currency conversion?" "The limit should be per-day, not per-transfer."

You're reviewing a page of plain English statements, not code. No programming knowledge required.

**4. AI translates properties into formal syntax.**

Once the human signs off on the properties, the AI converts them into a formal language — Dafny `requires`/`ensures` clauses, TLA+ invariants, Lean theorems. The human never needs to read this. But as a sanity check, a second AI pass translates the formal spec *back* into plain English. If the round-trip doesn't match the original properties, the translation is wrong. The human reads English twice and checks it says the same thing both times.

The formal syntax is an implementation detail between the AI and the verifier. You don't read LLVM bytecode when you write C. Same idea.

**5. Model check the design.**

Before writing any code, run the spec through a model checker like TLC. It explores every reachable state and surfaces surprises:

*"Scenario: Account A has balance 100. Transfer of 100 succeeds. Account A is now 0. A second transfer of 1 is attempted. It fails. But the failure handling path doesn't check..."*

You're debugging your **thinking**, not your code. This is where Amazon found the most value with TLA+ — catching design-level bugs that testing would never find because you'd never think to write that test.

**6. AI generates code + proof.**

The AI can brute force this — generate attempt after attempt, thousands of them. The verifier sits there saying "no, no, no, yes." It doesn't matter how many bad attempts the AI produces. The verifier never lets a wrong one through.

AlphaProof did exactly this for the International Math Olympiad. Generate thousands of proof candidates in Lean, let the kernel check them. It solved problems most human mathematicians couldn't.

**7. Validate against the real world.**

Put it in a simulator. Run it in staging. This isn't verifying the code — the proof did that. This is validating the **spec** against reality. When you find discrepancies, update the spec, regenerate, re-prove. Cheap, because the AI does the heavy lifting.

## Feature-First Development

This is where spec-driven development changes how you plan work, not just how you write code.

Model checking explores every reachable state of a system. Two variables with 100 possible values each means 10,000 states. Add a third and it's 1,000,000. A fourth: 100,000,000. That's state explosion — the reason you can't model check an entire system at once.

But a single well-scoped feature? A transfer between two accounts with a balance cap of 10,000? That's a tiny state space. A model checker chews through it in seconds.

**The principle: the smaller you scope the feature, the simpler the verification becomes. Exponentially simpler.**

This aligns perfectly with how good teams already work — small cards, clear acceptance criteria, vertical slices. The difference is that acceptance criteria become formal properties, and "done" means "verified," not "tests pass."

Consider the difference:

A ticket that says *"build the payment system"* is impossible to model check. The state space is enormous. The properties interact in unpredictable ways. You'd be waiting until heat death of the universe for TLC to finish.

A ticket that says *"implement transfer between two accounts with a 10,000 limit where total funds are conserved"* — that's P1 through P8. Model checked in seconds. Verified in Dafny. Shipped with a proof.

**Each feature gets its own spec, its own properties, its own proof.** The bank transfer module is verified against P1-P8. The authentication module is verified against its own properties. The payment gateway against its own. You slice vertically by feature, not horizontally by layer.

This means your backlog drives your verification strategy. Smaller tickets aren't just good project management — they're what makes formal verification tractable. A team that already writes small, well-defined cards is already 80% of the way to spec-driven development. The missing step is writing the acceptance criteria as formal properties instead of prose.

The system-level integration — how verified features compose together — is validated through testing, monitoring, and model checking at the design level. But the critical logic within each feature slice is *proved*. And for most systems, that's where the bugs live.

## The Security Argument

Most CVEs are implementation bugs. Buffer overflows. SQL injection. Type confusion. Out-of-bounds reads. Integer overflows. The OWASP top 10 is essentially a list of things that formally verified code eliminates by construction.

This isn't a minor benefit. This is transformative.

If your code is proved to satisfy its spec, and the spec says "user input is always sanitised before reaching the database" and "array access never exceeds bounds" and "arithmetic never overflows" — then those entire vulnerability classes don't exist. Not because you wrote careful code. Not because you ran a scanner. Because it's *mathematically impossible* for them to occur.

Formal verification doesn't catch every security issue — side channels, hardware bugs, and social engineering are outside the model. But the implementation-level bugs that make up the vast majority of real-world exploits? Gone.

## Proof-Carrying Code and the Supply Chain Problem

Software supply chain attacks are the threat everyone is panicking about right now. You pull in a dependency — how do you know it's safe? You download a binary — how do you know it matches the source?

Spec-driven development offers something new: **proof-carrying code.** The artifact you ship isn't just code. It's code + spec + proof + cryptographic signature. Anyone can independently verify:

1. The spec says what it should (human review)
2. The code satisfies the spec (machine-checked proof)
3. The proof is valid (independent verification)
4. The artifact hasn't been tampered with (cryptographic signature)

The entire chain is verifiable. You don't trust the developer. You don't trust the build system. You don't trust the registry. You verify the proof. If it checks out, the code is correct — regardless of where it came from.

This is what real supply chain security looks like. Not scanning for known CVEs. Not hoping your dependencies are honest. Mathematical proof of correctness with cryptographic provenance.

## Making Specs Better

The obvious concern: what if the spec is wrong? You get a perfectly verified implementation of the wrong thing. This is the irreducible hard problem — verification proves you built the thing right, never that you built the right thing.

But there are well-established techniques for catching bad specs, and AI accelerates all of them:

**Executable specs.** In TLA+ or Alloy, you can simulate the spec before any code exists. The model checker explores every reachable state and surfaces surprising ones. You're debugging your thinking, not your code.

**Property-based thinking.** Instead of specifying exact behaviour ("return 42 when X"), specify invariants ("balance never goes negative", "altitude stays within envelope"). These are harder to get wrong because they match how domain experts actually think. A banker thinks "we never give out money we don't have" — that's a property, not a function signature.

**Counterexample-driven refinement.** The model checker finds a scenario the spec allows that shouldn't happen. You say "that's wrong." The spec tightens. Each round surfaces requirements you didn't know you had. With AI in the loop, this cycle takes minutes, not months.

**Domain-specific spec languages.** A spec language tailored to flight control or financial settlement constrains what you can express. You can't accidentally specify something physically impossible. The language itself prevents categories of errors.

## What This Doesn't Solve

Honesty time. Formal verification is not a silver bullet.

**Spec bugs.** The Ariane 5 exploded because the software was correct with respect to its spec — but the spec was inherited from Ariane 4 and didn't account for the new rocket's trajectory. Perfectly verified, perfectly wrong.

**Side channels.** Timing attacks, power analysis, electromagnetic emanation — these are physical phenomena outside the logical model. Your code can be proved correct and still leak secrets through the hardware.

**Hardware faults.** Cosmic rays flip bits. Memory degrades. Sensors lie. The proof says "if the hardware behaves as modelled" — and sometimes it doesn't.

**Concurrency at scale.** Verifying a single module is tractable. Verifying the emergent behaviour of thousands of distributed services communicating over unreliable networks is still an open research problem.

**The spec itself.** No tool can tell you whether your spec captures what the real world actually needs. That requires human judgment, domain expertise, and contact with reality. Always will.

Knowing these limits makes the approach stronger, not weaker. You use formal verification for what it's good at — eliminating implementation bugs — and other techniques for everything else.

## What Changes

**Version control changes.** You diff specs, review specs, merge specs. A PR becomes "I changed this property from X to Y" — not "I modified 47 files across 3 services." Every commit directly expresses a requirement change. The git history becomes a readable record of how your understanding evolved.

**Code review dies.** Style debates, naming arguments, clever-vs-readable — irrelevant. Nobody reviews generated code. Review is about intent: does this spec capture what we actually need?

**Tech debt becomes spec debt.** You don't accumulate cruft in code, because the code is regenerated. You accumulate it in specs — properties that were good enough at the time but no longer reflect reality.

**Certification changes.** Instead of telling a regulator "we tested 10,000 scenarios and they all passed," you hand them a mathematical proof. The Airbus verification process that takes years gets compressed dramatically.

**The job changes.** Software engineering starts to look more like engineering — you produce specifications, and the fabrication is automated and certified.

## The Progression

What developers version control:

- **1970s:** Machine code
- **1980s:** Source code
- **1990s:** Source code + build scripts
- **2000s:** Source code + infrastructure config
- **2010s:** Source code + containers + pipelines
- **Next:** Specs. Code drops out entirely.

Each step moved the human further from the machine and closer to intent. Each layer we automated, we stopped worrying about. Nobody writes assembly. Nobody configures servers by hand. Each time, people said "you'll always need a human for that." Each time, they were wrong.

Code is next.

## Start Now

You don't need to wait for the ecosystem to mature. The pieces exist today:

1. **Pick a verifier.** Dafny is the easiest on-ramp. Lean 4 has the most AI momentum. TLA+ is best for distributed systems design.
2. **Write an intent spec** for something small. A function. An API endpoint. A state machine.
3. **Send it to an LLM** with the prompt: "Translate this spec into Dafny with requires/ensures clauses. Generate an implementation the verifier accepts."
4. **Run the verifier.** If it fails, feed the error back to the LLM. Let it try again. Loop until it passes.
5. **Model check the spec** in TLA+ to find scenarios you missed.

That loop — spec, check, refine, generate, prove — is the future of software development. And you can start running it today.

---

*Spec-driven development isn't a framework or a methodology. It's the inevitable consequence of making formal verification cheap. When proving correctness costs less than writing tests, everything changes. The spec is all that remains.*
