# Technical Notes — Singularity Sandbox

## Goal

Each Lean → Rocq translation agent runs inside an isolated Singularity container.

The container provides:

- Bash access;
- Lean;
- Rocq;
- required libraries;
- a writable workspace.

The agent must not have access to:

- the host home directory;
- the orchestration repository;
- hidden Rocq references;
- unrelated benchmark solutions;
- hidden evaluation data.

---

## Singularity Image

A `.def` file is used to build the execution environment:

```bash
singularity build leantorocq.sif leantorocq.def
```

The image should contain at least:

- Python;
- Lean;
- Rocq;
- required Lean/Rocq libraries;
- standard Unix tools.

It may later be extended with:

- Mathlib;
- `kimina-lean-server`;
- additional Rocq libraries.

The environment should be fully self-contained for execution on Jean Zay without Internet access. Pytorch version should be compatible with h100 jeanzay nodes.

---

## Filesystem Isolation

The agent container should be launched without access to the host home directory or current working directory:

```bash
singularity exec \
  --containall \
  --no-home \
  --no-mount cwd \
  leantorocq.sif ...
```

Only explicitly selected directories should be mounted.

---

## Writable Workspace

Each agent receives its own writable host directory:

```bash
mkdir -p /path/to/runs/task_001
```

mounted as:

```bash
--bind /path/to/runs/task_001:/workspace:rw
```

Example:

```bash
singularity exec \
  --containall \
  --no-home \
  --no-mount cwd \
  --bind /path/to/runs/task_001:/workspace:rw \
  leantorocq.sif ...
```

The agent can create:

```text
/workspace/candidate.v
/workspace/scratch.v
/workspace/answer.v
```

These files remain available on the host after the container exits.

---

## Rocq Reference Isolation

For each task, the Rocq target comes from a source file such as:

```text
file.v
```

The agent should not have access to the complete original module.

Instead, create a task-specific truncated version containing only the Rocq code appearing before the target declaration:

```text
original file.v
├── prefix before target     → visible
├── target reference         → hidden
└── code after target        → hidden
```

The sandbox should therefore expose:

```text
file.v     → truncated version
```

while hiding compiled artifacts from the original full module:

```text
file.vo
file.vos
file.vok
file.glob
```

This prevents the agent from recovering or importing the hidden target through the compiled module.

Unrelated Rocq library `.vo` files remain available so that dependencies can still be imported normally.

The intended environment is:

```text
Rocq libraries
├── dependencies/*.vo       visible
├── file.v                  truncated
├── file.vo                 hidden
├── file.glob               hidden
└── downstream artifacts    hidden if they depend on full file
```

The main invariant is:

> The agent can access everything needed before the target, but no source or compiled artifact containing the target itself.

---

## Hiding Files Already Present in the Image

If the original module is already present inside the Singularity image, its paths can be masked using empty bind mounts.

For example:

```bash
mkdir -p /tmp/empty_target_vo
```

and bind an empty path over the compiled artifact or its containing directory when launching the container.

The truncated `file.v` is then mounted explicitly from a task-specific host directory.

Reference files should preferably remain completely outside the sandbox whenever possible.

---

## Agent Launch

A typical worker launch becomes:

```bash
singularity exec \
  --containall \
  --no-home \
  --no-mount cwd \
  --bind "$RUN_DIR:/workspace:rw" \
  --bind "$TRUNCATED_FILE:/path/to/rocq/file.v:ro" \
  leantorocq.sif \
  python /agent/launch_agent.py
```

Additional bind mounts can mask the original compiled artifacts associated with the target module.

For parallel evaluation, create one workspace and one Singularity process per task:

```text
runs/
├── task_0001/
├── task_0002/
├── task_0003/
└── ...
```

Each task receives its own truncated Rocq context and isolated workspace.

---

## Main Principle

The container should expose only:

```text
Lean/Rocq environment
+ allowed compiled libraries
+ truncated task-specific source
+ /workspace
```

Everything related to the hidden reference, downstream benchmark code, evaluation logic, or other tasks remains outside the agent sandbox.