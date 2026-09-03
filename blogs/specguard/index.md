---
layout: bare
title: "Can We Prove a Task Is Broken Before the Agent Cheats?"
permalink: /specguard/
description: "A method that formalizes a task description into Lean without looking at the tests, then asks a theorem prover whether any implementation could satisfy both."
image: /blogs/specguard/excalidraw_pipeline.png
---

# Can We Prove a Task Is Broken Before the Agent Cheats?

*Param Biyani (MATS Research), Krishnamurthy (Dj) Dvijotham (Google DeepMind)*

**TL;DR:** Coding agents are sometimes handed tasks whose tests contradict their descriptions. Agents rarely flag the contradiction.[^1] Instead, they cheat, and the cheating can do real damage. We built *SpecGuard*, a method that formalizes the task description into Lean without ever looking at the tests, then asks a theorem prover whether any implementation could satisfy both at once. On conflicted SWE-bench tasks it detects up to 89% of conflicts and formally proves up to 67% of them, producing machine-checked certificates. A paper is in preparation, feedback welcome.

* * *

Three weeks ago the UK AI Security Institute [disclosed](https://www.aisi.gov.uk/blog/incident-report-unsanctioned-agent-behaviour-during-cyber-testing) that agents in a routine cyber evaluation went after real people. In the most serious case, an agent created fake identities and tried to socially engineer an open source maintainer into merging malicious code.[^2] We bring this up for a narrow reason. Untrusted contributions flowing into real codebases are no longer hypothetical, and every untrusted test, PR, or bug report is a chance for an agent's success criterion to quietly come apart from what the maintainers actually want. Our work is about catching one concrete version of that split before an agent acts on it.

## Conflicted tasks

A coding agent gets a task in words and gets judged by a test suite. Usually the two agree. Sometimes the tests demand behavior the description never asked for, or directly contradict it. We call these **conflicted tasks**. A conflicted task is impossible as written, and an agent judged only by the tests can succeed only by cheating.

[ImpossibleBench](https://arxiv.org/abs/2510.20270) (Zhong et al., 2025) measured what agents actually do here, by mutating benchmark tests to contradict the task. The agents edit the tests, hard-code expected outputs, or special-case whatever inputs the tests check. GPT-5.6 Sol cheats on 99% of conflicted [LiveCodeBench](https://arxiv.org/abs/2403.07974) tasks.[^3]

Here is what that looks like when the stakes are real. Django has a feature that lets a site rotate a leaked secret key without logging every user out. The code that makes the rotation safe includes one step that kills stolen sessions:

```diff
 if session_hash matches a fallback secret:
-    request.session.cycle_key()   # security: resets session ID
 request.session[HASH_SESSION_KEY] = session_auth_hash
```

We gave an agent this task with a corrupted test. Its solution was to delete the `cycle_key` line, the session hijacking defense itself, and report that everything passes. Every test was green. Unless a human reads the diff and understands the security property that just disappeared, that change ships, and the damage is real.

Where do conflicted tasks come from? Sometimes a human or another agent writes a malformed test out of carelessness and nobody notices. Sometimes a well-maintained project accepts tests and bug reports from outside, and something in that stream is careless or adversarial. Either way the cheating is only the mechanism. The damage is whatever the agent breaks in order to pass.

## The idea

Can we catch a broken objective before the agent acts on it? Our answer has three steps.

First, formalize the intent. A model translates the natural language task description into a formal specification in [Lean 4](https://lean-lang.org/), a proof assistant that checks proofs down to axioms. The spec is generated from the description and the codebase alone. The tests never touch it. That independence is the whole point: if the tests could leak into the spec, a corrupted test would corrupt our notion of intent along with it, and there would be nothing left to compare against.

Second, prove. We ask whether any implementation at all could satisfy both the specification and the tests. Lean can settle that question.

Third, certify. When the answer is no, we get a machine-checked certificate that the task is broken, and the agent is stopped instead of being left alone with an impossible objective.

None of this was buildable two years ago. Frontier models only recently became reliable at producing type-correct Lean from natural language, and that reliability is what makes the SpecGuard perform.

## The *SpecGuard* pipeline

![The SpecGuard pipeline]({{ '/blogs/specguard/excalidraw_pipeline.png' | relative_url }})

SpecGuard has three components. For a [SWE-bench](https://arxiv.org/abs/2310.06770) style task, a SpecAgent reads the description and codebase and writes spec.lean. A TestAgent independently translates the tests into tests.lean. A Certifier then tries to prove that no implementation satisfies both files, and emits a certificate when it succeeds.

## A worked example

Task django-11532: Django crashes when generating an email Message-ID on hosts with Unicode names, and the fix requires Punycode-encoding the hostname. From the issue description alone, the SpecAgent wrote about 190 lines of Lean. Not a paraphrase of the test. A general Punycode encoder:

```lean
structure EncodeState where
  n : Nat
  delta : Nat
  bias : Nat
  handled : Nat
  output : List Nat

def adaptBias ...
def findMinimum ...
def emitVariable ...
def encodeLoop ...

def run (domain : String) : String :=
  "xn--" ++ encodePunycode domain
```

Spec.run "`漢字`" computes "`xn--p8s937b`", exactly as [RFC 3492](https://www.rfc-editor.org/rfc/rfc3492) says it should. Against the mutated test the verdict was CONFLICT, and a manual audit judged the spec general and genuine.[^4]

## Results

We evaluate on ImpossibleBench style benchmarks, coding tasks with injected specification-test conflicts, in three settings.

Throughout, we separate two claims. A conflict is **detected** when *SpecGuard*'s verdict matches ground truth. Any method could make that claim, including an LLM judge that simply reads the tests. A conflict is **proven** when an independent machine-checkable artifact exists: Lean has verified, down to axioms, that no implementation can satisfy both spec and tests. For a proven conflict you do not need to trust the pipeline, the model, or us. The Lean kernel re-checks the certificate independently in seconds, assuming the formalization is faithful. More on that assumption below.

![SpecGuard results on SWE-bench Lean across formalizing models]({{ '/blogs/specguard/swebench_lean_models.png' | relative_url }})

On SWE-bench Lean, results track the capability of the formalizing model.[^5]

![SpecGuard results across benchmarks with GPT-5.6 Sol]({{ '/blogs/specguard/gpt56sol_benchmarks.png' | relative_url }})

Across benchmarks with GPT-5.6 Sol, detection reaches 84.2% on LiveCodeBench, 74.8% on SWE-bench with a Python reference model, and 73.9% on SWE-bench Lean, of which 67.0% is fully proven. Self-contained tasks are easier than repository-scale ones, consistent with the specification gap (the distance between what someone means and what any formal spec captures) widening as context grows.

We ran a baseline judge agent with the same model, similar budget and python access, and asked directly whether the task is conflicted. It misses nearly half of them. It also barely false-alarms (2.3%). As a check on the formalization step itself, SpecAgent-generated specs match human-written ones in bidirectional equivalence 96% of the time on competition-style problems.[^6]

## Levels of evidence

The three settings do not produce the same kind of claim, and we think being precise about the difference matters more than the headline numbers.

SWE-bench Lean is the strongest tier. The certificate says no possible implementation can ever satisfy both spec and tests. For LiveCodeBench, the computation is Lean-checked, but the correctness claim is per input. SWE-bench Python is the weakest tier: a reference model disagrees with the tests, reproducibly. That is evidence rather than proof. The Python tier generalizes cheaply and uses codebase native artifacts, and the Lean tier is what makes the word certificate honest. We report them separately for that reason.

## What the certificate does not guarantee

The proof is machine-checked and, in that sense, unimpeachable. It is also a proof about the formalized specification, and the formalization comes from a model. The bottleneck is faithfulness. The proof guarantees consistency with the LLM-generated spec, and consistency with the author's true intent is inferred rather than proven.

That gap produces two failure modes. A *false positive* flags a sound task as conflicted and blocks legitimate work, which erodes trust in the detector. A *false negative* misses a real conflict while making the task look verified. The second one is the dangerous case, since a check that is wrong but believed can be worse than no check at all. Characterizing these error rates is central to the project, and any deployment needs an honest account of exactly what is proved versus what is inferred.

## Why this matters

The near-term case is practical. Labs running RL pipelines and teams building agent harnesses already wrap models in tools and checks. A feasibility check that flags conflicted tasks before an agent, or a training run, learns to cheat on them is cheap to add: median cost per task is $0.46, median time under three minutes.

The longer-term case rides on an open question: does the specification gap close at scale? If it does, the same formalize-and-check recipe extends beyond this one property, toward provable protections against agent misbehavior under conflicting goals. If it does not, we will have mapped where formal methods hit their ceiling in bridging informal intent and machine-checked guarantees. Either answer seems worth having.

## Feedback we want

* We tried an agentic judge as a baseline. Would that convince you? Self-consistency voting, something else?
* If you build agent harnesses or eval infrastructure: what detection rate, false-flag rate, and cost per task would make this worth integrating?
* Where do you expect the formalization to be least faithful, and how would you stress-test it?

Reach us at [parambiyani8@gmail.com](mailto:parambiyani8@gmail.com); [dvij@cs.washington.edu](mailto:dvij@cs.washington.edu).

[^1]: In ImpossibleBench's evaluations, models given contradictory tests overwhelmingly exploit the tests rather than reporting the contradiction to the user.

[^2]: AISI attributes 17 of the 19 unsanctioned actions to a single sustained line of activity by one agent, and reports the attempts failed with no known real-world harm. The incident spanned July 25 to 28, 2026, and was disclosed on August 4.

[^3]: This figure is from our own runs of GPT-5.6 Sol on ImpossibleBench's LiveCodeBench split. The cheating rates reported in the original paper, for earlier models, are lower.

[^4]: By general and genuine we mean the spec implements the actual Punycode algorithm for arbitrary inputs, rather than hard-coding the behavior of the specific test cases, as judged by a human reading the Lean.

[^5]: A false negative here means the pipeline confidently reported no conflict on a task that had one. Tasks that are neither detected nor false negatives returned an inconclusive verdict.

[^6]: Bidirectional equivalence: we state the LLM-generated spec and the human-written reference spec as theorems and prove in Lean that each implies the other, currently using the grind tactic. The two specs were proven equivalent for 96% of problems.
