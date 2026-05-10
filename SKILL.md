# Swarm Skill

This is a skill that bootstraps a team of agents and a mini cli tool to help do this.

## First understand the project

You need to ask the user some questions about what they're trying to build.
This includes the tech stack, and some high level requirements of the project.
Write this stuff to SPEC.md in the current folder.

## Playwright CLI

One prerequisite is to have `playwright-cli --help` available on the command line.
This is the playwright-cli from microsoft - look it up and install with npm if not available.
Make sure to write a config file:

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

The swarm.yaml is a way for the user to easily modify and see the prompts of the agents, and the other config.
By default, the below should work for most projects.

`swarm.yaml`:

```yaml
version: "1"
tasks:
  planner:
    concurrency: 1
    prompt-string: |
      Brainsorm and then plan the next logical task.
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

Write a CLI tool called `swarm` in Node which does the following:

- reads and parses swarm.yaml
- runs the appropriate pipeline with a certain parallelism and for a number of iterations
- have complete visibility into the agents context window under an `output` directory - this includes the config used to start it, the jsonl output, and the text-output for humans to read.
- if the user changes swarm.yaml - make sure this is picked up on the very next step or iteration

If the user uses claude, the claude cli should be used in non-interactive mode with the following args:

```bash
--system-prompt You are an expert coding assistant operating inside Claude Code, a coding agent harness. You help users by reading files, executing commands, editing code, and writing new files.
--dangerously-skip-permissions
--model opus
--effort high
--print
--output-format stream-json
--verbose
```

If the user uses codex - do something equivelant.

## Robustness

Sometimes claude finishes its task but doesn't send the expected finish event.
Also sometimes there is a bug where it may just hang anyway.

We should be robust to these kinds of failures, as it will stop the entire pipeline.
Detect these and take appropriate remediations.
