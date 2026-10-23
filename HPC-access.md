# CCUS HPC access

[Introduction](Introduction.md)

Operational notes for the Unistra Centre de Calcul (CCUS, cluster CAIUS): how to log in, what we were allocated, and how to serve a quantized Qwen3.8-27B on the GPUs for [swarm runs](Dataset-SFT-pilot.md). Sources: [access](https://hpc.pages.unistra.fr/access), [hardware](https://hpc.pages.unistra.fr/equipment), [Slurm doc](https://hpc.pages.unistra.fr/doc/slurm), allocation email (2026).

## Logging in

Cluster: Red Hat 8.7, x86_64. Authentication by SSH public key only.

```
ssh -Y phelluy@hpc-login.u-strasbg.fr    # batch submission, CPU compile/tests, file transfers
ssh -Y phelluy@hpc-glogin.u-strasbg.fr  # GPU compile/tests without the queue; X2Go; +128 Go RAM
```

Login `phelluy` (key registered with dnum-cesar-support@unistra.fr; connection verified).

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
| H200 | 5 (`hpc-n977-981`, AMD EPYC gen4, 2.3 To RAM) | 141 Go | `gpuh200` |

Too small / too old for this model: P100 (16.2 Go), 1080Ti (11.1 Go), RTX 5000/6000, K-series. Tensor cores (deep-learning workloads) are selected with the `gputc` constraint; A100/H100/L40S qualify. Slurm allocates a fraction of a GPU node (e.g. 1/4 node = 1 GPU); max 4 GPUs per node via `--gres=gpu:N`.

**Recommendation:** one GPU, `--constraint="gpua100|gpul40s|gpuh100"`. A 27B Q4_K_XL + 32k context fits comfortably in 40-48 Go; H100 if free. Nodes are Intel and AMD (EPYC Rome/gen4) — no constraint needed on CPU side, Slurm picks either.

Walltime: `grant`/`grantgpu` cap at 4 days (`4-00:00:00`, verified via `sinfo -l`); public partitions cap at 24 h.

## Software environment

Installed via modules (GCC, CMake, CUDA, Python, Singularity). Verified: `gcc/15.2.0`, `cmake/3.31.8`, `cuda/13.0`, `python/3.12.8` (defaults). No vLLM/llama.cpp module: build llama.cpp ourselves, same stack as local development.

**Build environment pitfall (verified 2026-10-07).** The cluster's gcc modules set `CPATH`/`C_INCLUDE_PATH`/`CPLUS_INCLUDE_PATH` to the module's **C++ header directory** (`.../include/c++/14.2.0`). Every C compilation then resolves `<stdatomic.h>` to the libstdc++ (C++-only) version, so C11 atomics fail (`atomic_int` unknown) — this breaks ggml and any C11 code. After `module load`, unset them:

```
module load gcc/14.2.0 cmake cuda
unset CPATH C_INCLUDE_PATH CPLUS_INCLUDE_PATH
```

Use `gcc/14.2.0`, not the default `gcc/15.2.0`: CUDA 13.0 needs a GCC host compiler <= 14. Pass compilers explicitly (`-DCMAKE_C_COMPILER/-DCMAKE_CXX_COMPILER/-DCMAKE_CUDA_HOST_COMPILER`); otherwise CMake may pick `/usr/bin/cc` (GCC 8.5). The system NCCL is too old for llama.cpp (`ncclBfloat16` missing); build with `-DGGML_CUDA_NCCL=OFF` (fine for 1 GPU per job; a `nccl/2.20.3` module exists if multi-GPU ever needs it). Keep `module load cuda` at runtime too, or `libcudart.so.13` won't be found.

Build on `hpc-glogin` (GPU tests without the queue) or inside a job:

```
module load gcc/14.2.0 cmake cuda
unset CPATH C_INCLUDE_PATH CPLUS_INCLUDE_PATH
mkdir -p ~/gitlab && cd ~/gitlab
git clone --depth 1 https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -S . -DGGML_CUDA=ON -DGGML_CUDA_NCCL=OFF \
      -DCMAKE_CUDA_ARCHITECTURES="80;89;90" \
      -DCMAKE_C_COMPILER=$(which gcc) -DCMAKE_CXX_COMPILER=$(which g++) \
      -DCMAKE_CUDA_HOST_COMPILER=$(which g++)
cmake --build build --config Release -j24
```

`CMAKE_CUDA_ARCHITECTURES="80;89;90"` covers A100, L40S, H100, H200 (the login node has no GPU, so auto-detection cannot be used). Verified build: llama.cpp 0.6.0-dev, commit 18b5f8b, ~20 min at `-j24` on `hpc-glogin`.

## Model files

`hf` (huggingface_hub CLI) is installed via `pip install --user` but lands in `~/.local/bin`, which is not on the default PATH; call the Python API instead. Downloaded 2026-10-07 into `~/gitlab/llama.cpp/models/`:

```
module load python
python -c "from huggingface_hub import hf_hub_download; \
print(hf_hub_download(repo_id='prism-ml/Ternary-Bonsai-2-27B-gguf', \
filename='Ternary-Bonsai-2-27B-PQ2_0.gguf', local_dir='~/gitlab/llama.cpp/models'))"
```

- `Ternary-Bonsai-2-27B-PQ2_0.gguf`, 6.8 Gio on disk (7.21 GB advertised). LLM only; `mmproj` vision projectors not downloaded.
- *WIP: compare against unsloth Qwen3.8-27B-GGUF UD-Q4_K_XL on one pilot problem — see [Evaluation](Evaluation.md) models.*

## Serving the swarm (batch)

One job = one GPU serving several agent copies as parallel slots; the harness runs in the same job on CPU cores. Compute-node ports are not reachable from outside, so never plan on accessing `llama-server` from the laptop; talk to it from inside the job.

**Validated end-to-end 2026-10-07** (job 18117006, `hpc-n888`, A100 40G): server ready in 16 s, one OpenAI-style `/v1/chat/completions` request from job-local Python, answer written to JSON, ~68 tok/s. Working scripts on the cluster: `~/gitlab/marx-prover/test_swarm.sh` (batch) and `test_api.py` (minimal client). Runtime pitfalls found on the way:

- `module load python` pulls `gcc-12`, whose old `libstdc++` lands first in `LD_LIBRARY_PATH`; binaries built with gcc 14 then die instantly with no message. Fix after all module loads: `export LD_LIBRARY_PATH=/usr/local/gcc/14.2.0/lib64:$LD_LIBRARY_PATH`.
- The Ternary-Bonsai GGUFs (`PQ2_0`/`PTQ1_0`) need the **PrismML-Eng/llama.cpp fork** (`prism` branch); upstream master rejects them (`invalid ggml type 142`). The fork also fixes a PQ2_0 load segfault on AVX-512 CPUs (our A100 nodes are EPYC Rome). The fork is checked out in `~/gitlab/llama.cpp` (remote `prism`, branch `prism`).
- Build with `GGML_NATIVE=OFF`: `-march=native` compiled on `hpc-glogin` SIGILLs on compute nodes (each node family has a different CPU ISA).
- Node-local `/scratch/job.*` can be **full**; write results straight to `$HOME/results/`.
- The model reasons before answering: give generous `max_tokens`, and use `reasoning_effort: "medium"`/`"xhigh"` — `"high"` returns HTTP 500 (vendor KNOWN_ISSUES).

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

source /etc/profile
module load gcc/14.2.0 cuda python
unset CPATH C_INCLUDE_PATH CPLUS_INCLUDE_PATH
export LD_LIBRARY_PATH=/usr/local/gcc/14.2.0/lib64:$LD_LIBRARY_PATH
SRV=$HOME/gitlab/llama.cpp/build/bin/llama-server   # prism branch of the fork
MODEL=$HOME/gitlab/llama.cpp/models/Ternary-Bonsai-2-27B-PQ2_0.gguf
OUT=$HOME/results/run_$SLURM_JOB_ID
mkdir -p $OUT

$SRV -m $MODEL -ngl 99 -c 32768 -np 4 --cont-batching --jinja \
     --host 127.0.0.1 --port 8080 > $OUT/server.log 2>&1 &
SRV_PID=$!

# wait for readiness, with a death check
for i in $(seq 1 300); do
    kill -0 $SRV_PID 2>/dev/null || { echo "server died at $((i*2))s"; break; }
    curl -s http://127.0.0.1:8080/health | grep -q '"ok"' && break
    sleep 2
done

# swarm harness: N agents, shared folder, chat.log; each agent POSTs to
# http://127.0.0.1:8080/v1/chat/completions
python ~/gitlab/marx-prover/harness.py --agents 4 --api http://127.0.0.1:8080/v1 \
      --out $OUT/run

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

## Lean 4 environment (validated 2026-10-23)

Needed to check Lean proofs produced during swarm runs (miniF2F pilot, see [Dataset](Dataset-SFT-pilot.md)). Everything is installed under `$HOME` via elan on the login node, so it works unchanged inside batch jobs.

- elan 4.2.4 + toolchain `leanprover/lean4:v4.27.0` (matches the [google-deepmind/miniF2F](https://github.com/google-deepmind/miniF2F) checkout in `~/gitlab/miniF2F`).
- Mathlib oleans come from the cache server (7869 `.ltar` files, 6.7 Gio unpacked into `.lake/`). Never build Mathlib from source on the cluster.
- Lean checks are CPU jobs: `#SBATCH -p grant -A <SLURM_ACCOUNT_CPU>`, 4-8 cores; `--mem=16G` is enough for `import Mathlib` (peak RSS ~8-10 Gio).

**Pitfall 1 — elan's bundled clang needs glibc 2.29, the cluster (RHEL 8.7) has 2.28.** Any step compiling generated C (`precompileModules`: Batteries, the Cache tool) fails. Workaround: `LEAN_CC` pointing at the cluster's clang 5 (`/usr/local/llvm/llvm-5/bin/clang`), via the wrapper `~/gitlab/marx-prover/pilot/lean-cc.sh` which also drops `-fstack-clash-protection` (rejected by clang 5) and appends `-L$HOME/.elan/toolchains/leanprover--lean4---v4.27.0/lib` so linking finds `-lc++ -lc++abi -luv -lgmp`:

```
export PATH=$HOME/.elan/bin:$PATH
export LEAN_CC=$HOME/gitlab/marx-prover/pilot/lean-cc.sh
```

**Pitfall 2 — the mathlib cache downloader's curl.** `lake exe cache get` fetches a static curl 7.88.1 into `~/.cache/mathlib/curl-7.88.1`; on the cluster that binary fails every transfer (`OpenSSL/3.0.8: STORE routines::unregistered scheme`). The system curl (7.61.1) downloads fine but is too old for the tool's `--parallel` + `--write-out %{json}` batch mode. Workaround: replace `~/.cache/mathlib/curl-7.88.1` with the bash wrapper `~/gitlab/marx-prover/pilot/curl-wrapper.sh` (kept as a copy), which emulates the batch mode with system curl + `xargs -P 8` and emits the same JSON status lines. With both wrappers in place, `lake exe cache get` runs end-to-end (~20 min: hashing, download, unpack).

**Performance retex (measured 2026-10-23).**

- `import Mathlib` = load ~8600 oleans (6.7 Gio): ~10 min with a cold NFS cache, ~3 min warm (a pure re-read of the tree takes seconds — server-side page cache). Mathlib is **never recompiled**: lake only re-checks ~7900 traces and compiles the target module.
- For fast proof-iteration loops, prefer `lean --server` (resident LSP process, incremental re-checks) or narrow imports (`Mathlib.Tactic.FieldSimp` instead of `Mathlib`) over one fresh `lean` process per attempt.
- Warm-up idea: inside the same Slurm job, start the Lean (or Rocq) warm-up in parallel with model loading and the first inferences — e.g. one CPU core runs a dummy `lake env lean` (`import Mathlib` on a scratch file, or a one-shot check of `MiniF2F.Valid`) while the GPU serves. The ~10 min cold NFS read then overlaps with inference instead of delaying the first proof check (~3 min warm).
- Do not pipe job logs through `tail`: output is buffered and the log stays empty until the very end. Monitor with a `vmstat`/`date` loop instead (`diag_check.sbatch`).
- Compute-node SSH from the login node is refused; live process inspection inside the job is the only option.

**Pilot result (first end-to-end proof loop).** Ternary-Bonsai-2-27B (PQ2_0, A100, job 18222659, 81 s / 5712 completion tokens) proved miniF2F `amc12a_2017_p2` in one request: its `calc`/`field_simp` proof was one tactic short (`ring` missing after the first `field_simp`, leftover goal `y + x = x + y`). The patched proof verified with `lake build MiniF2F.MarxPilot` (job 18222865, 2 min 58 s warm) and locally against Mathlib v4.35.0. Artifacts on the cluster: scripts in `~/gitlab/marx-prover/pilot/` (`prove.sbatch`, `prove_client.py`, `lean-cc.sh`, `curl-wrapper.sh`, `diag_check.sbatch`, `nfs_probe.sbatch`), inference results in `~/results/lean_pilot_18222659/`, proof file `~/gitlab/miniF2F/MiniF2F/MarxPilot.lean`.

## Job monitoring

```
squeue -u phelluy                    # my jobs
scontrol show job <jobid>            # details, estimated start
sinfo -l                             # partitions, default time limits
scancel <jobid>
```

Scratch `/scratch/job.$SLURM_JOB_ID` is wiped at job end — and can also be **full** (seen 2026-10-07: 6.8 Go copy failed, later even a small JSON write failed). Prefer writing results to `$HOME/results/` directly.

## WIP

- Exact model repo/quant; compare ternary Bonsai-2 vs unsloth Q4_K_XL on one pilot problem.
- Module names/versions (`module avail gcc cmake cuda python`).
- Whether a second CPU job can join a running GPU job's server (multi-job swarm split), or always single-job.
