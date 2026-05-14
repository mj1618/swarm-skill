---
name: swarm-skill
description: Bootstraps a ralph-loop swarm of planner/builder/reviewer agents driven by a swarm.yaml config and a small `swarm` CLI, so the team can work on a project autonomously. Use when the user wants to set up an autonomous multi-agent team, a ralph loop, or a continuous planner-builder-reviewer pipeline for their codebase.
---

## Overview

This skill bootstraps a team of agents and a mini CLI tool that drives them in a ralph loop (planner → builder → reviewer, repeated). Follow the steps below in order.

## First understand the project

Unless `SPEC.md` already exists (meaning the user has already been through this step), ask the user some questions about what they're trying to build. Cover the tech stack and some high-level requirements of the project. Keep it short and succinct — 3 questions max. Write the answers to `SPEC.md` in the current folder.

## Playwright CLI

One prerequisite is to have `playwright-cli --help` available on the command line. This is the playwright-cli from Microsoft — look it up and install it with npm if it isn't available. Also write a config file:

`.playwright/cli.config.json`:

```json
{
  "browser": {
    "browserName": "chromium",
    "userDataDir": ".playwright/profile",
    "launchOptions": {
      "channel": "chrome"
    }
  }
}
```

## Write the swarm.yaml

The `swarm.yaml` lets the user easily view and modify the prompts of the agents and the rest of the config. The default below should work for most projects — write it as-is unless the user has asked for something different.

`swarm.yaml`:

```yaml
version: "1"
tasks:
  planner:
    concurrency: 1
    prompt-string: |
      Brainstorm and then plan the next logical task.
      Look at the SPEC.md, the code base, and use playwright-cli to view the application to come up with ideas for the next task.
      The frontend and backend should stay in sync, don't let one get too far ahead of the other.
      Write to ./swarm/tasks/{incrementalNumber}-{taskName}.pending.md

  builder:
    prompt-string: |
      Find a task ./swarm/tasks/*.pending.md and claim it by rewriting to .processing.md
      Build the task.
      Then mark the task as done by renaming to .done.md

    depends_on: [planner]

  reviewer:
    prompt-string: |
      Find the last .done.md task, rename it to .reviewing.md
      Thoroughly test the task.
      Use a browser with `playwright-cli` on the command line to test (run `playwright-cli --help` for instructions).
      Find other ways to test also.
      Rename to .reviewed.md when complete.
      Git commit and push everything when done.

    depends_on: [builder]

pipelines:
  main:
    iterations: 1000
    parallelism: 1
    tasks: [planner, builder, reviewer]
```

## Build the CLI tool

If a `swarm` CLI tool already exists under the current project folder, don't write a new one — but do make sure it meets all the requirements below.

If it doesn't exist, create a CLI tool called `swarm` in Node that:

- Reads and parses `swarm.yaml`.
- Runs the appropriate pipeline with a given parallelism for a given number of iterations.
- Gives complete visibility into each agent's context window under an `output/` directory — including the config used to start it, the JSONL output, and a human-readable text output. Iteration folder names must be sequential across different runs of the tool (a second run should NOT restart from iteration-1, as that's confusing for the user).
- Picks up changes to `swarm.yaml` on the very next step or iteration if the user edits it mid-run.

If the user uses Claude, invoke the `claude` CLI in non-interactive mode with the following args:

```bash
--system-prompt "You are an expert coding assistant operating inside Claude Code, a coding agent harness. You help users by reading files, executing commands, editing code, and writing new files."
--dangerously-skip-permissions
--model opus
--effort high
--print
--output-format stream-json
--verbose
```

If the user uses Codex, do something equivalent.

## Robustness

Sometimes Claude finishes its task but doesn't send the expected finish event. Sometimes it may just hang outright. Both will stop the entire pipeline if not handled.

Detect these failures and take appropriate remediations so the pipeline keeps running despite them.
