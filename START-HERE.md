# Start here

This folder is the starting point of the agency platform repository. It carries everything decided so far,
so Claude Code (and any developer) works from the same plan.

```
agency/
  CLAUDE.md                      rules Claude Code reads automatically at the start of every session
  START-HERE.md                  this file
  docs/analysis-and-design.md    what the system must do
  docs/development-strategy.md   how we build it, roadmap, concepts to learn
  docs/adr/                      decisions already made
  docs/learning/                 your notes, one per concept
```

## 1. Install the tools (once)

- Git, Docker Desktop, the current .NET LTS SDK, Node.js LTS and pnpm, VS Code.
- Claude Code: follow the official setup page — https://docs.claude.com/en/docs/claude-code/overview —
  then install the Claude Code extension in VS Code from the Extensions view.

## 2. Create the repository

1. Unzip this kit into a new folder, for example `agency`.
2. In a terminal inside that folder: `git init`, then commit everything.
3. Create an empty private repository on GitHub and push to it.
4. Open the folder in VS Code (`File → Open Folder`).

## 3. Start Claude Code in the project

Open Claude Code from the VS Code extension (or run `claude` in the VS Code terminal, inside the project folder).
It reads `CLAUDE.md` automatically. Start with this first message:

> Read CLAUDE.md, docs/development-strategy.md (sections Strategy, Roadmap, The walking skeleton, Repository)
> and the ADRs. Then propose a step-by-step plan for the walking skeleton only ("create a city and list cities"),
> as small tasks, each one a pull request. Do not write code yet.

Review the plan, then ask it to do task 1. For every task:

1. Ask it to explain any new concept first (it will link to the concept tables).
2. Write or review the domain test yourself.
3. Let it implement, then read the diff and run the tests.
4. Commit, push, and move to the next task.

## 4. Keep the context alive

- When a decision is made in a session, ask Claude Code to add an ADR in `docs/adr/`.
- When something in the design changes, update `docs/analysis-and-design.md` in the same pull request.
- Add useful commands to the "Commands" section of `CLAUDE.md` as the project grows.
- Long-running conversations are not memory: the files in this repository are.
