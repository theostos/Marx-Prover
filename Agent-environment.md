# Agent environment

[Introduction](Introduction.md)

**Base environment: bash + a communication channel.** Each copy of the model uses these tools to prove and coordinate. Rocq/Lean and scripts are accessible through bash. The swarm builds its own harness: roles, task allocation, shared artifacts, and coordination scripts.

Two motivations:

- [NEAR's bash harness](https://github.com/nearai/putnambench-deepseek) reports 639/672 PutnamBench problems solved on its first pass (~95.1%). [Goedel-Architect](https://arxiv.org/abs/2606.06468), with blueprint generation/refinement, reports 75.6% pass@1; 88.8% with proof hints. Simpler scaffolding, higher reported coverage; models and budgets differ.
- [Noam Brown's interview](https://www.youtube.com/watch?v=6AgOfiZOWiY) describes primitive messaging tools with minimal imposed structure; agents learn how to coordinate. [Transcript](https://www.dwarkesh.com/p/noam-brown).

Design hypothesis: a minimal environment leaves [organization](Organization.md) to training. Proof verification, allowed resources, and budget enforcement remain external to the generated harness; see [Harness](Harness.md). Log actions, messages, and shared artifacts for [controls and ablations](Experimental-controls.md).
