---
layout: post
title: "Spec-Driven Development: How AI and Formal Methods Will Change Software Engineering"
date: 2026-04-15
---

*The spec is the source code. The code is a build artifact.*

---

## The Bottleneck Nobody's Talking About

AI writes code faster than humans can review it. That's the new reality. Copilot, Claude, GPT — they generate thousands of lines in seconds. Google and Microsoft now report that 25-30% of new code is AI-generated, with credible projections that this hits 95% by 2030.

And somewhere a developer is squinting at a diff trying to decide if it's correct.

Andrej Karpathy said the quiet part out loud: *"Accept All always, I don't read the diffs anymore."* When AI code is good enough most of the time, humans stop reviewing carefully. That's already happening across the industry, and the people doing it are some of the best engineers in the world.

The bottleneck in software is no longer writing code. It's *verifying* code. And the gap is widening every day.

So what if you didn't have to review the code at all? What if a machine could *prove* it's correct?

That's not hypothetical. The technology exists. It's called formal verification. And AI is about to make it cheap enough for everyone.

## Formal Methods: Powerful but Expensive — Until Now

Formal verification gives you mathematical proof that code does what it's supposed to do. Not "we tested a bunch of cases and they passed." Proof. For all inputs. For all states. Always.

Airbus has used it since the A320. NASA uses it for mission-critical systems. Amazon used TLA+ to find critical bugs in S3 and DynamoDB that no amount of testing would have caught. Microsoft is using Lean to verify SymCrypt cryptographic libraries. AWS verified its Cedar authorization policy engine. The technique works.

And almost nobody else uses it. Because writing formal specs and proofs has always cost 3-5x more than just writing the code. Only industries where failure kills people or loses billions could justify it. Everyone else writes tests and ships.

AI changes this equation overnight. LLMs are already decent at translating natural language into formal specifications — TLA+, Alloy, Dafny, Lean. And here's what matters: **the verification side doesn't need AI at all.** Proof checkers like Coq, Lean, and Isabelle have small trusted kernels that have been scrutinised for decades. The AI sits entirely outside the trusted computing base.

This is the asymmetry that makes the whole thing work. As Leonardo de Moura — the creator of Z3 and Lean — puts it:

> "A proof tool doesn't assert that a theorem is true. It produces a certificate — a detailed formal artifact — which is then independently verified by a small, simple program called the kernel."

**AI generates. Math verifies.** The trust doesn't depend on the AI being correct. It depends on the verifier being correct — and the verifier is a small, well-understood piece of mathematics.

Generating correct code is hard. Checking a proof is easy. That asymmetry is the whole game.

De Moura's framing is sharper than mine: *"The answer is not to slow AI down. It is to replace human friction with mathematical friction: let AI move fast, but make it prove its work."*

## The Spec Is the Source Code

This changes what developers actually do. Instead of writing code and hoping tests catch the bugs, you write a spec — a set of properties your system must satisfy. AI generates the code and a proof. A verified tool chain checks the proof. If it passes, the code is correct. Period.

The human moves up the abstraction ladder. Instead of coding, you become a **spec reviewer**. Humans are better at "is this what I actually want?" than "did I handle the edge case on line 4,217?"

I call this **spec-driven development.**

The mental model that helps me: **LLMs are the new compiler. The spec is the new programming language.** You used to write code that the compiler translated to machine instructions. Now you write specs that the LLM translates to code. And just like nobody reads compiler output, nobody reads the generated code.

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

A good system doesn't just translate — it *interrogates*. "You said handle payments. What happens on failure? Retry? Refund? Error to user?" "You haven't mentioned authentication — is this endpoint public?" The AI's domain knowledge surfaces the questions you didn't think to ask. The human makes the decisions.

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

**6. AI generates code + proof together.**

The AI doesn't just write code — it writes the code *and* the formal proof that the code matches the spec. Same model, same generation, both artifacts together. The AI can brute force this — generate attempt after attempt, thousands of them. The verifier sits there saying "no, no, no, yes." It doesn't matter how many bad attempts the AI produces. The verifier never lets a wrong one through.

AlphaProof did exactly this for the International Math Olympiad. ByteDance's Seed Prover hit IMO gold-medal threshold the same way — generate Lean proof candidates, let the kernel check them, iterate with structured feedback. The same architecture works for software correctness theorems.

**7. Validate against the real world.**

Put it in a simulator. Run it in staging. This isn't verifying the code — the proof did that. This is validating the **spec** against reality. When you find discrepancies, update the spec, regenerate, re-prove. Cheap, because the AI does the heavy lifting.

## AI Drives Automation, It Doesn't Replace It

Here's a subtle point that took me a while to internalise. The naive picture of "AI writes proofs from scratch" is wrong. We have decades of automated reasoning algorithms — SMT solvers, decision procedures, term rewriting, model checkers. They handle huge classes of proofs in milliseconds.

De Moura's correction is sharp: *"Training AI to synthesize proof terms is training it to simulate algorithms we already have. It's like training a model to output x86 assembly instead of Python."*

The AI's job isn't to be a brilliant theorem prover. It's to be a **conductor** — knowing which automation tool to use, how to configure it, what hints to provide. The actual proof-finding is done by deterministic algorithms that have been tested for decades.

In Lean, this means writing good *annotations* — metadata like `[simp]`, `[grind]`, `grind_pattern` — that tell automation how to use theorems. These annotations are extremely hard to write well. Most mathematicians can't, even after years of practice. But AI can be trained on a clear feedback signal: "did this annotation help close more goals?"

Translate that to software verification: your YAML spec doesn't compile to "AI generates 500 lines of proof." It compiles to "AI invokes the right tactic with the right hints, which closes the proof in milliseconds." The AI orchestrates trusted tools. The trusted tools do the verification. Speed and trust both win.

## Who Watches the Verifier?

If everything depends on the proof checker being correct, what happens when the proof checker has a bug?

This isn't paranoid. In 2026, an AI researcher used Claude Opus 4.6 to find seven distinct kernel bugs in Rocq — a proof assistant with 30+ years of development behind it. AI is now better than humans at systematically exploring weird edge cases. And that creates a new threat model: an adversarial AI doesn't need to prove a false statement, it just needs to find a kernel bug and exploit it. The kernel accepts the "proof" because of the bug. Now you have a "verified" theorem that's actually false. The whole verification stack collapses.

The defence is **multiple independent kernels**. Don't trust one verifier. Have several, written by different teams, in different languages, using different algorithms. Every proof gets checked by all of them. They have to agree.

Concrete example: in 2022, Lean 4's official kernel accepted an invalid proof because of an arithmetic bug with large numbers. Nanoda — an independent Lean kernel written in Rust, under 5,000 lines — rejected the same proof because it implemented arithmetic differently. Bug found, fixed in 24 hours. Single kernel: the bug ships, false theorems get certified, nobody notices. Multiple kernels: the disagreement immediately surfaces the bug.

This is why the Lean Kernel Arena now runs 7 kernels against 133 benchmarks continuously, with adversarial testing built into the development process. The system publicly dares people (and AIs) to break it.

The architecture matters. The kernels must be small enough to audit. The implementations must be diverse enough to fail differently. And — critically — the verifier must not be controlled by the same vendor that provides the AI. *"If the same vendor provides both the AI and the verification, there is a conflict of interest. Independent verification is not a philosophical preference. It is a security architecture requirement."*

## Why Lean Won

Every major AI reasoning system that has hit medal-level performance at the International Math Olympiad uses Lean — AlphaProof (DeepMind), Aristotle (Harmonic), Seed Prover (ByteDance), plus Axiom, Aleph, and Mistral AI. No competing platform was used by any of them.

This isn't accident. Lean has architectural properties that turn out to matter enormously in the AI era:

**Self-implementation.** Lean is implemented in Lean. The compiler, elaborator, parser, and tactics are all written in the same language users write proofs in. This means AI that's good at writing Lean is also good at extending Lean. The system can recursively improve itself. Other systems with a host-language layer underneath (OCaml, Haskell) can't do this — extending them requires dropping out of the proof language.

**Small trusted kernel.** A few thousand lines of carefully scrutinised code. Multiple independent implementations exist. The trust boundary is tight and auditable.

**Programmable tactics.** AI can compose existing automation rather than generating proofs from scratch. Mathlib contains 50,000+ lines of custom tactics built on Lean's metaprogramming layer.

**Massive existing library.** Mathlib has 200,000+ formalized theorems, 750 contributors, and 8,000+ public Lean repositories on GitHub. It's a training corpus and a foundation to build on.

**Open governance.** The Lean FRO (Focused Research Organisation) is structured as a nonprofit specifically to prevent vendor capture. The substrate stays open, no matter who builds on it.

The choice has already been made by the field. Lean is the substrate.

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

There's a class of bugs that testing literally cannot catch. Consider an AI rewrite of a TLS implementation. It passes every functional test. But it contains a subtle conditional that varies with key bits — a timing side-channel. Testing can't detect this; the timing difference is below noise on most runs. Code review probably misses it; humans aren't great at spotting timing dependencies. A formal proof of *constant-time behaviour* catches it instantly. The spec says "execution time is independent of secret bits" and either the code satisfies it or it doesn't.

Formal verification doesn't catch every security issue — hardware bugs and social engineering are outside the model. But the implementation-level bugs that make up the vast majority of real-world exploits? Gone.

## Proof-Carrying Code and the AI Supply Chain Problem

Software supply chain attacks are the threat everyone is panicking about right now. You pull in a dependency — how do you know it's safe? You download a binary — how do you know it matches the source?

AI makes this dramatically worse. If everyone's using the same handful of models to generate code, an attacker who can poison training data or compromise a model API can inject vulnerabilities at industry scale, simultaneously, across every system that AI touches. A determined adversary can study the test suite and plant bugs specifically designed to evade it. This is a systemic risk, not just a quality problem. Poor software quality already costs the U.S. economy $2.4 trillion a year. AI-driven supply chain compromise could push that an order of magnitude higher.

Spec-driven development offers something new: **proof-carrying code.** The artifact you ship isn't just code. It's code + spec + proof + cryptographic signature. Anyone can independently verify:

1. The spec says what it should (human review)
2. The code satisfies the spec (machine-checked proof)
3. The proof is valid (independent verifier — ideally several)
4. The artifact hasn't been tampered with (cryptographic signature)

The entire chain is verifiable. You don't trust the developer. You don't trust the build system. You don't trust the registry. You don't trust the AI that wrote the code. You verify the proof. If it checks out, the code is correct — regardless of where it came from.

This is what real supply chain security looks like. Not scanning for known CVEs. Not hoping your dependencies are honest. Mathematical proof of correctness with cryptographic provenance, checkable by anyone.

## Making Specs Better

The obvious concern: what if the spec is wrong? You get a perfectly verified implementation of the wrong thing. This is the irreducible hard problem — verification proves you built the thing right, never that you built the right thing.

But there are well-established techniques for catching bad specs, and AI accelerates all of them:

**Executable specs.** In TLA+ or Alloy, you can simulate the spec before any code exists. The model checker explores every reachable state and surfaces surprising ones. You're debugging your thinking, not your code.

**AI as interrogator.** Before generating any code, the AI asks clarifying questions about your spec — partial refunds, rate limits, GDPR concerns, error handling. The AI is better than you at thinking of edge cases because it's seen millions of similar systems. You're better than the AI at deciding what should happen. Correct division of labour.

**Property-based thinking.** Instead of specifying exact behaviour ("return 42 when X"), specify invariants ("balance never goes negative", "altitude stays within envelope"). These are harder to get wrong because they match how domain experts actually think. A banker thinks "we never give out money we don't have" — that's a property, not a function signature.

**Counterexample-driven refinement.** The model checker finds a scenario the spec allows that shouldn't happen. You say "that's wrong." The spec tightens. Each round surfaces requirements you didn't know you had. With AI in the loop, this cycle takes minutes, not months.

**Domain-specific spec languages.** A spec language tailored to flight control or financial settlement constrains what you can express. You can't accidentally specify something physically impossible. The language itself prevents categories of errors.

## What This Doesn't Solve

Honesty time. Formal verification is not a silver bullet.

**Spec bugs.** The Ariane 5 exploded because the software was correct with respect to its spec — but the spec was inherited from Ariane 4 and didn't account for the new rocket's trajectory. Perfectly verified, perfectly wrong.

**Side channels.** Timing attacks, power analysis, electromagnetic emanation — these are physical phenomena outside the logical model. Your code can be proved correct and still leak secrets through the hardware. (Though *some* side channels — like constant-time guarantees — are formalisable.)

**Hardware faults.** Cosmic rays flip bits. Memory degrades. Sensors lie. The proof says "if the hardware behaves as modelled" — and sometimes it doesn't.

**Concurrency at scale.** Verifying a single module is tractable. Verifying the emergent behaviour of thousands of distributed services communicating over unreliable networks is still an open research problem.

**Non-functional requirements.** "The app should feel fast." "The recommendations should be relevant." "The UI should be intuitive." These are real requirements that matter and they don't formalise cleanly. Functional requirements — what the system *does* — are provable. Non-functional requirements — how it does it — are mostly empirical.

**The spec itself.** No tool can tell you whether your spec captures what the real world actually needs. That requires human judgment, domain expertise, and contact with reality. Always will.

Knowing these limits makes the approach stronger, not weaker. You use formal verification for what it's good at — eliminating implementation bugs in functional requirements — and other techniques for everything else.

## Rebuilding the Stack from the Bottom Up

Here's the longer arc that makes this more than incremental improvement.

Software is built in layers. Your app sits on top of libraries, which sit on top of protocols, which sit on top of operating systems, which sit on top of cryptography and compilers. If you formally verify your app but the layers underneath are buggy, your verification is meaningless. You proved your code correct *assuming* the libraries work correctly. But OpenSSL has had bugs (Heartbleed). zlib has had bugs. SQLite has had bugs. Compilers have had bugs that change the meaning of your code.

It's like building a verified house on an unverified foundation.

The proposal — and it's de Moura's proposal explicitly — is to verify the stack from the bottom up. Start with the layers everything else depends on:

1. **Cryptography first.** Microsoft is verifying SymCrypt. AWS verified Cedar. Once crypto is verified, every system using it inherits a guarantee.
2. **Core libraries.** The Lean FRO has zlib in Lean now, with a proven theorem that decompression recovers the original data. Kim Morrison directed Claude — general-purpose, off-the-shelf — to do the translation and verification. *Today.*
3. **Protocols.** JSON parsers, HTTP, DNS, certificate validation. These have all had catastrophic bugs over the years. Verified versions eliminate entire categories of vulnerabilities.
4. **Compilers and runtimes.** CompCert proved this is possible for C. Extending it to mainstream compilers is engineering, not research.

Each verified layer makes the next layer easier and more meaningful to verify. Verified crypto means TLS can prove "this connection is secure" without re-proving the crypto. Verified TLS means HTTP can prove "this request is authenticated" without re-proving the connection. The guarantees compose. Verification gets *cheaper* as the stack matures.

Most catastrophic security incidents of the last 20 years — Heartbleed, Shellshock, Log4Shell, Dirty COW — were bugs in foundational layers. Every one would have been prevented by formal verification of the affected layer. A verified stack doesn't just make new software safer. It eliminates entire classes of past incidents from ever happening again.

This is a multi-decade project. But every piece exists today. AI makes the human-effort cost tractable for the first time in history. The question is no longer *whether* this happens, but *when*.

## What Changes

**Version control changes.** You diff specs, review specs, merge specs. A PR becomes "I changed this property from X to Y" — not "I modified 47 files across 3 services." Every commit directly expresses a requirement change. The git history becomes a readable record of how your understanding evolved.

**Code review dies.** Style debates, naming arguments, clever-vs-readable — irrelevant. Nobody reviews generated code. Review is about intent: does this spec capture what we actually need?

**Tech debt becomes spec debt.** You don't accumulate cruft in code, because the code is regenerated. You accumulate it in specs — properties that were good enough at the time but no longer reflect reality.

**Certification changes.** Instead of telling a regulator "we tested 10,000 scenarios and they all passed," you hand them a mathematical proof. The Airbus verification process that takes years gets compressed dramatically.

**Trust changes fundamentally.** Today we trust software because smart people wrote it, tests passed, other smart people reviewed it, it hasn't broken yet. That's all *social* trust — it depends on humans being competent and honest. In the spec-driven world, trust is *mathematical*. Multiple independent verifiers all agree. The math works out. Anyone can re-verify. For the first time in computing history, software trust doesn't reduce to "trust the people who built it."

**The job changes.** Software engineering starts to look more like engineering — you produce specifications, and the fabrication is automated and certified. Productivity comes not from generating more code, but from generating code that is provably correct on the first attempt.

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

1. **Pick a verifier.** Dafny is the easiest on-ramp. Lean 4 has the most AI momentum and the strongest community. TLA+ is best for distributed systems design.
2. **Write an intent spec** for something small. A function. An API endpoint. A state machine.
3. **Send it to an LLM** with the prompt: "Translate this spec into Dafny with requires/ensures clauses. Generate an implementation the verifier accepts."
4. **Run the verifier.** If it fails, feed the error back to the LLM. Let it try again. Loop until it passes.
5. **Model check the spec** in TLA+ to find scenarios you missed.

That loop — spec, check, refine, generate, prove — is the future of software development. And you can start running it today.

## Further Reading

Most of the technical thesis here builds on Leonardo de Moura's writing. He's the main architect of Lean, Z3, Yices 1.0 and SAL — essentially much of the foundational tooling underneath modern automated reasoning. Today he's a Senior Principal Applied Scientist in the Automated Reasoning Group at AWS, having joined in 2023 after 17 years as a Senior Principal Researcher at Microsoft Research's RiSE group. He's also the co-founder and Chief Architect of the Lean FRO, the non-profit he started with Sebastian Ullrich to keep the verification substrate open and vendor-independent. His work has been recognised with the CAV, Haifa, and Herbrand awards, plus the ACM SIGPLAN Programming Languages Software Award twice — once for Z3 and once for Lean.

He's been blogging through the implications of AI for formal methods:

- *[Proof Assistants in the Age of AI](https://leodemoura.github.io/blog/2026-2-18-proof-assistants-in-the-age-of-ai/)* — why design of the verification language matters more, not less, with AI in the loop
- *[When AI Writes the World's Software, Who Verifies It?](https://leodemoura.github.io/blog/2026-2-28-when-ai-writes-the-worlds-software-who-verifies-it/)* — the case for replacing human review with mathematical proof
- *[Teaching AI to Make Proof Automation Work](https://leodemoura.github.io/blog/2026-3-14-teaching-ai-to-make-proof-automation-work/)* — why AI should orchestrate existing tactics rather than generate proofs from scratch
- *[Who Watches the Provers?](https://leodemoura.github.io/blog/2026-3-16-who-watches-the-provers/)* — multiple independent kernels as defence against adversarial AI
- *[Why Lean?](https://leodemoura.github.io/blog/2026-4-2-why-lean/)* — the architectural choices that made Lean the substrate

---

*Spec-driven development isn't a framework or a methodology. It's the inevitable consequence of making formal verification cheap. When proving correctness costs less than writing tests, everything changes. The spec is all that remains.*
