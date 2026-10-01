# Training

[Introduction](Introduction.md)

Fine-tune one model through the experience of a [swarm of its copies](Organization.md). Coordination, tool use, and proving belong to the same learned behavior.

## Data and SFT

Run copies of a larger teacher, e.g. [DeepSeek V4.1 Flash](https://api-docs.deepseek.com/news/news260910/), as a swarm in the minimal environment.

Record how the copies organize, write scripts, exchange messages (*WIP: how to implement it?*), and complete proofs.

**SFT supplies collaborative examples to warm up RL.**

## Data autonomy policy

The generator may be autonomous; the judge is not. An agent may propose and formalize its own training problems, provided:

- Verification stays external and frozen: statements, allowed assumptions, and the Lean project are not under the swarm's control.
- Evaluation problems and their traces stay excluded and unreachable by the generation pipeline.
- Difficulty targets a success window of the current student (problems the student solves sometimes, not always or never).
- A quota of statements comes from sources outside the model, to avoid distributional collapse toward what it already solves.

Pilot plan: [SFT pilot dataset](Dataset-SFT-pilot.md).

## RL

For each rollout, run copies of the current student checkpoint on a shared problem and budget. They organize, prove, and revise their approach together.

Compare [credit assignment methods](Learning-signal.md) and measure the benefit of interaction through [controlled experiments](Experimental-controls.md).

Easy problems should need almost no initialization. Harder problems may justify more setup when it improves success. This tradeoff should emerge from training.

## Curriculum

Start with problems and verified subgoals the student sometimes solves. Mix these with harder tasks; adjust sampling as success rates change. Add teacher examples when rewards become too sparse.

[Prioritized Level Replay](Related-work.md) is a starting point for adaptive sampling.

**WIP:** training problem source; difficulty sampling rule; use of intermediate rewards; when to add fresh teacher data. Keep evaluation problems and their generated traces excluded.
