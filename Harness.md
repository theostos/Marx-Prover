# Harness

[Introduction](Introduction.md)

The harness is the external scaffolding around a swarm run: it starts the agents, wires them to the model server, enforces the budget, applies the acceptance gates, and records traces. It imposes no organization — roles, protocols, and division of labor belong to the [agents](Agent-environment.md).

## Invariants

The harness is defined as much by what it refuses to do:

| The harness does | The harness never does |
| --- | --- |
| Launch N copies of the checkpoint, each with its own context and `$AGENT_ID`. | Assign roles, turn order, or a workflow. |
| Expose bash and the shared folder; keep `chat.log` appendable. | Read, filter, route, or answer messages. |
| Enforce the budget and expose what remains of it. | Retry, reschedule, or assist a struggling agent. |
| Check the final folder state against the [acceptance gates](Dataset-SFT-pilot.md). | Inspect or steer the path taken; only the end state is judged. |
| Record every trace, including failures, to `failures/`. | Discard runs that do not match an expected protocol. |

## Agent loop

Each agent is one conversation with the model through the OpenAI-compatible API:

```
system prompt (mission, environment description, acceptance criteria)
   ↓
model output → bash tool calls → results fed back
   ↓
repeat until the agent stops or the budget is exhausted
```

- A single system prompt, identical for every agent: mission, environment description, acceptance criteria. Nothing else.
- `$AGENT_ID` is injected by the environment. The prompt only states that `chat.log` is a shared channel that every agent can read and append to, to use as they see fit.
- Messaging is broadcast-only, so a dead or idle agent blocks no one. The harness does not require any agent to respond or act.

## Budget

- One token cap and one compiler-round cap per run ([pilot](Dataset-SFT-pilot.md)); all agents draw on the same pool.
- The remaining budget stays readable by the agents so the swarm can adapt ([learning signal](Learning-signal.md)).
- Budget exhaustion ends the run, whatever the folder state; the result is recorded, not retried.

Units and limits are [WIP](Learning-signal.md); comparisons across model sizes and quantizations need compute accounting.

## Verification

Acceptance gates run on the final folder state, never on the path taken ([details](Dataset-SFT-pilot.md)): the four artifacts present, `F.lean` compiles with no `sorry` and no extra axioms, `V.lean` proves it, and everything recompiles in a frozen Lean project outside the generated harness. Statements, allowed assumptions, and the verifier stay external to the swarm; [exploits can spread](Related-work.md).

## Traces

- One JSONL per agent per run: full conversation, tool inputs and outputs, loss mask.
- `manifest.json` per run: seeds, model, budgets, per-problem statistics.
- Failures are recorded under `failures/`, including silos, redundancies, and dead agents. Diversity of organizations is a feature, not noise.

## Deployment

The harness runs on CPU cores inside the same Slurm job as `llama-server`, and each agent POSTs to the job-local endpoint (see [CCUS HPC access](HPC-access.md)). Job start should also launch the verifier warm-up in parallel (one core doing a dummy `import Mathlib`, ~10 min cold on NFS — see the [Lean 4 retex](HPC-access.md)) so the first proof check is fast; on a Rocq pilot the same applies to the Rocq toplevel and its package loads. What exists on the cluster today (`~/gitlab/marx-prover/`) is the serving layer only: `test_swarm.sh` (batch launcher) and `test_api.py` (minimal client), validated end-to-end with a single request. The swarm harness itself is unwritten; `harness.py` appears only as a hypothetical invocation in the Slurm template.

### Pilot loop validated (2026-10-23)

The minimal loop — generate, check, repair, re-check — runs end-to-end on CCUS, one problem (miniF2F `amc12a_2017_p2`), Ternary-Bonsai-2-27B PQ2_0 on a single GPU, Lean checks as separate CPU jobs (`grant`):

1. Model proves in one request (81 s, 5712 tokens); proof fails to compile by one tactic (`ring` missing after a `field_simp`, leftover goal `y + x = x + y`).
2. Compiler error fed back verbatim; model repairs in one request (8 s, 787 tokens) with the expected one-line fix.
3. Repaired proof verified by Lean on the cluster (`Build completed`, ~3 min warm) — no human in the loop.

Full traces (prompts, CoT, proofs, compiler errors, timings) are kept for the SFT dataset on the laptop (`~/.marx-prover/lean4-pilot/`) and on the cluster (`~/results/lean_pilot_18222659/`, `~/results/lean_repair_18222951/`); scripts on the cluster in `~/gitlab/marx-prover/pilot/` (`prove.sbatch`, `repair_client.py`, `repair.sbatch`). Per-attempt overhead outside inference: one `lake build` (~3 min warm, no Mathlib recompilation). Next step: fold the check into the same job as the server and iterate repairs in a budgeted loop.

## Existing harnesses

Nothing off the shelf matches this design (N concurrent copies of one checkpoint, shared folder + broadcast log, no imposed protocol): the minimal loops are single-agent, and the multi-agent frameworks impose structure. Useful building blocks:

| Harness | What it provides | Fit |
| --- | --- | --- |
| [putnambench-deepseek](https://github.com/nearai/putnambench-deepseek), `agentlib.py` | Self-contained Python loop: OpenAI-compatible chat + tool calling, cgroup-sandboxed bash, SQLite trajectory capture, context compaction. 672/672 PutnamBench. | Closest single-agent base. Its parallel agents ran as independent processes; no shared state, no channel. |
| [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) | ~100 lines of Python, bash only, no tool-calling interface, one `subprocess.run` per action, fully linear message history. | Trajectory = message list, convenient for SFT loss masks. Single agent only. |
| [AxProverBase](https://arxiv.org/abs/2602.24273) (ICML 2026) and [mini-Lean-agent](https://arxiv.org/abs/2610.17331) | Minimalist Lean agents: compiler feedback (+ self-managed notebook for AxProverBase); a generic coding agent with a minimal harness already scores well on Lean benchmarks. | Design evidence that scaffolding can stay minimal; single agents. |
| [OpenAI Swarm](https://github.com/openai/swarm) / Agents SDK, LangGraph Swarm, swarm-kit, blackboard frameworks | Multi-agent orchestration via handoffs or shared state between heterogeneous agents. | Contradicts the design: roles and protocols are imposed by the framework, not learned. |

Implication: start from a minimal single-agent loop (agentlib or mini-swe-agent style) and add the swarm layer ourselves — shared folder, `chat.log`, N concurrent loops, budget pool, trace recording. Nothing of that layer exists yet.

## WIP

- Write the swarm harness (minimal loop, single bash tool) and version it in this repo; only the serving layer exists on the cluster.
- Exact budget units; how the remaining budget is exposed to agents.
- Agent termination criteria: when does the loop consider an agent done vs stuck.
- Loss-mask policy: system prompt, tool outputs, and injected state.
- Replay support: re-execute a recorded trace with messages, artifacts, or agents removed, for [controls and ablations](Experimental-controls.md).
