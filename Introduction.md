# Marx-Prover

Marx-Prover aims to fine-tune a language model so that a swarm of its copies learns to organize itself around formal proof problems. Every agent uses the same weights and can both coordinate and prove.

The project is inspired by [OpenAI's ~10 000-agent Navier–Stokes experiment](https://openai.com/index/navier-stokes-solution/). We want to explore smaller swarms using small or heavily quantized models.

The agents learn to build their working setup together: divide tasks, write scripts, and share results. The goal is better verified proof success under a fixed compute budget. Simple problems may need little preparation; harder ones may benefit from more organization.

- [Organization](Organization.md)
- [Environment](Agent-environment.md)
- [Training](Training.md)
- [SFT pilot dataset](Dataset-SFT-pilot.md)
- [Learning signal](Learning-signal.md)
- [Evaluation](Evaluation.md)
- [Experimental controls](Experimental-controls.md)
- [Related work](Related-work.md)
- [Research workflow](Research-workflow.md)
- [CCUS HPC access](HPC-access.md)
