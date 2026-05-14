# Swarm Skill

The skill bootstraps (just from the SKILL.md) an autonomous ralph-loop swarm of agents — a planner, a builder, and a reviewer — that work on your project continuously. The skill writes a `swarm.yaml` config (so you can tweak each agent's prompt) and a small `swarm` Node CLI that drives the pipeline, captures each agent's full context window to disk, and keeps running across iterations. Use it when you want a multi-agent team to keep shipping work on a codebase without you babysitting each step.

## Get started

No need to install the skill, simply paste the following into your LLM inside an empty project folder:

```markdown
Read and implement the following:
https://raw.githubusercontent.com/mj1618/swarm-skill/refs/heads/main/SKILL.md
```
