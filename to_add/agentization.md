**Overview**
---

The idea here is to see how far we can go by giving an LLM access to the shell. This step lives in `2_agentic`.

We give the model a source statement, its context (i.e. the full source library), and a target library (with some restriction to avoid leakage) with fixed versions. Through bash, it can search the libraries, read definitions, write a candidate statement, and run Lean, Rocq, and Python. The setup can stay simple: [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) is a useful reference for an agent with bash as its only tool (and probably the reason why LLM are so proficient in using bash).

For the first experiments, we focus on statements whose underlying objects already exist in the target library. 

Later, we could try on never translated statement: if a definition is missing, the agent should report it so that we can measure how often this blocks translation.

We compare direct generation, an agent with compiler feedback, and an agent with compiler + judge/BEq feedback. We use the same model, examples and total token budget, with limits on tool calls and elapsed time. We record correctness, typechecking rate, cost etc.
*Remark:* in experiments without a target baseline, the benchmark reference files stay outside the agent's workspace and feedback commands.

*WIP:* We should describe what to check, and how to check safely the output of an LLM.

*WIP:* Design skills for bash access

**Technical**
---

[Container Technical Overview](container.md)

- Loop Agentic / inference server: We use vLLM for the inference server. For the agentic loop, a simple loop with bash access (containerized), a token count (cumulative IN/OUT), a bash output limiter (to avoid bloating the context), a time and/or token limiter (as argparse arguments), and time feedback (every k% of each limit, send a signal to the LLM).
Need to design skills:
    - explain how to use bash (basic commands, etc.)
    - the `timeout` command
    - how to end the run (special bash command)

    The system prompt + base prompt (to start the task) should be easy to access, inspect, and modify (with template values that are changed at runtime).
**Experimentation**
---