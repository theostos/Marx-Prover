# CCUS HPC access

[Introduction](Introduction.md)

Operational notes for the Unistra Centre de Calcul (CCUS, cluster CAIUS): how to log in, what we were allocated, and how to serve a quantized Qwen3.8-27B on the GPUs for [swarm runs](Dataset-SFT-pilot.md). Sources: [access](https://hpc.pages.unistra.fr/access), [hardware](https://hpc.pages.unistra.fr/equipment), [Slurm doc](https://hpc.pages.unistra.fr/doc/slurm), allocation email (2026).

## Logging in

Cluster: Red Hat 8.7, x86_64. Authentication by SSH public key only.

```
ssh -Y <login>@hpc-login.u-strasbg.fr    # batch submission, CPU compile/tests, file transfers
ssh -Y <login>@hpc-glogin.u-strasbg.fr  # GPU compile/tests without the queue; X2Go; +128 Go RAM
```

*WIP: `<login>` — key registered with dnum-cesar-support@unistra.fr, login pending.*

Data transfers: `scp` / `rsync` via the same hosts. Default disk quota 500 Go per user (`diskquota` to check; 27B GGUF ~17 Go is fine). The cluster is working storage only: pull results back out. Support: dnum-cesar-support@unistra.fr, subject "HPC".

## 2026 allocations

| Account | Partition | Allocation | Notes |
| --- | --- | --- | --- |
| `<SLURM_ACCOUNT_CPU>` | `grant` | `<CPU_HOURS>` CPU-h | CPU jobs (Lean checking, trace post-processing, SFT prep). |
| `<SLURM_ACCOUNT_GPU>` | `grantgpu` | `<GPU_HOURS>` GPU-h | Inference. |

- Directives in every script: `#SBATCH -p grantgpu -A <SLURM_ACCOUNT_GPU>` (the two partitions need different accounts). Real account names and hour quotas come from the annual allocation email and stay out of the repo.
- Accounting: 1 GPU-h is billed as 6 CPU-h; all values printed by `sreport` are CPU-equivalent, divide by 6.
- Check consumption: `sreport cluster AccountUtilizationByUser start=2026-08-30 accounts=<SLURM_ACCOUNT_GPU> -t hours`.
- 2025 accounts are disabled on 2026-09-07; finish old runs before.
- Jobs in `grant`/`grantgpu` preempt jobs in `public` — our jobs have priority; expect few long queue waits, but keep `-t` realistic.

## GPU hardware

290+ GPUs total. Relevant ones for a 27B quant (~17 Go weights at Q4_K_XL):

| Type | Count | VRAM | Slurm feature |
| --- | --- | --- | --- |
| Tesla H100 | 12 | 80 Go | `gpuh100` |
| Tesla A100 | 16 | 40 Go | `gpua100` |
| Tesla L40S | 8 | 48 Go | `gpul40s` |
| Tesla A40 | 10 | 48 Go | (check `sinfo`) |
| Tesla V100 | 26 | 32.5 Go | `gpuv100` |

Too small / too old for this model: P100 (16.2 Go), 1080Ti (11.1 Go), RTX 5000/6000, K-series. Tensor cores (deep-learning workloads) are selected with the `gputc` constraint; A100/H100/L40S qualify. Slurm allocates a fraction of a GPU node (e.g. 1/4 node = 1 GPU); max 4 GPUs per node via `--gres=gpu:N`.

**Recommendation:** one GPU, `--constraint="gpua100|gpul40s|gpuh100"`. A 27B Q4_K_XL + 32k context fits comfortably in 40-48 Go; H100 if free.

## Software environment

Installed via modules (GCC, CMake, CUDA, Python, Singularity; check `module avail` for exact names/versions — *WIP: verify on first login*). No vLLM/llama.cpp module: build llama.cpp ourselves, same stack as local development.

Build on `hpc-glogin` (GPU tests without the queue) or inside a job:

```
module load gcc cmake cuda
git clone https://github.com/ggml-org/llama.cpp
cmake -B llama.cpp/build -S llama.cpp -DGGML_CUDA=ON
cmake --build llama.cpp/build --config Release -j8
```

## Model files

```
module load python
pip install --user huggingface_hub
hf download <repo> --local-dir ~/models/qwen3.8-27b --include "*UD-Q4_K_XL*"
```

*WIP: exact repo and quant (unsloth Qwen3.8-27B-GGUF UD-Q4_K_XL vs prism-ml Ternary-Bonsai-2-27B-gguf — see [Evaluation](Evaluation.md) models).*

## Serving the swarm (batch)

One job = one GPU serving several agent copies as parallel slots; the harness runs in the same job on CPU cores. Compute-node ports are not reachable from outside, so never plan on accessing `llama-server` from the laptop; talk to it from inside the job.

```
#! /bin/bash
#SBATCH -p grantgpu
#SBATCH -A <SLURM_ACCOUNT_GPU>
#SBATCH -N 1
#SBATCH --gres=gpu:1
#SBATCH --constraint="gpua100|gpul40s|gpuh100"
#SBATCH -c 8
#SBATCH --mem=32G
#SBATCH -t 04:00:00
#SBATCH -o swarm_%j.out

module load gcc cmake cuda python
SRV=~/llama.cpp/build/bin/llama-server
MODEL=~/models/qwen3.8-27b/*UD-Q4_K_XL*.gguf

$SRV -m $MODEL -ngl 99 -c 32768 -np 4 --cont-batching --jinja \
     --host 127.0.0.1 --port 8080 > server.log 2>&1 &
SRV_PID=$!

# wait for readiness
until curl -s http://127.0.0.1:8080/health | grep -q '"ok"'; do sleep 2; done

# swarm harness: N agents, shared folder, chat.log; each agent POSTs to
# http://127.0.0.1:8080/v1/chat/completions
python ~/marx-prover/harness.py --agents 4 --api http://127.0.0.1:8080/v1 \
      --out /scratch/job.$SLURM_JOB_ID/run

# copy results back before the scratch is wiped
rsync -a /scratch/job.$SLURM_JOB_ID/run ~/results/run_$SLURM_JOB_ID
kill $SRV_PID
```

Notes:

- `-np 4`: 4 parallel decoding slots (up to 4 concurrent agent requests) with continuous batching; the KV cache (`-c`) is shared across slots. On 80 Go (H100) raise `-np` for the 4-agent swarm plus compiler subprocesses.
- `-ngl 99`: offload all layers to the GPU.
- Budget discipline: one such job costs 4 GPU-h (billed 24 CPU-h in `sreport`).
- Kill the server explicitly; Slurm terminates the job's processes anyway, but the log tail stays clean.

Interactive debugging, no queue abuse:

```
salloc -p grantgpu -A <SLURM_ACCOUNT_GPU> --gres=gpu:1 --constraint=gpuh100 -c 8 -t 01:00:00
```

then run `llama-server` by hand and probe with `curl`. Quick GPU compile checks: `hpc-glogin` (no allocation, do not run long jobs there).

## Job monitoring

```
squeue -u <login>                    # my jobs
scontrol show job <jobid>            # details, estimated start
sinfo -l                             # partitions, default time limits
scancel <jobid>
```

Scratch `/scratch/job.$SLURM_JOB_ID` is wiped at job end — always rsync out.

## WIP

- `<login>` from the CCUS (pending key registration).
- Exact model repo/quant; compare ternary Bonsai-2 vs unsloth Q4_K_XL on one pilot problem.
- Module names/versions (`module avail gcc cmake cuda python`).
- Max walltime on `grant`/`grantgpu` (`sinfo -l`); public partitions cap at 24 h.
- Whether a second CPU job can join a running GPU job's server (multi-job swarm split), or always single-job.
