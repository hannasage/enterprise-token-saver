# enterprise-token-saver

A Claude Code plugin that cuts the cost of a coding session.

## What it does

The plugin splits a coding task in two. The main session, called the
orchestrator, keeps the work that needs judgment. Mechanical code generation
goes to Claude Haiku subagents that run on a written brief. Haiku output tokens
cost a fraction of Sonnet output tokens, so the mechanical half of a task gets
much cheaper.

The routing rule in brief: delegate anything a careful junior engineer can
finish from the brief alone with zero questions. Keep everything else. Planning,
architecture, ambiguous requirements, debugging, code review, and any code
touching authentication, payments, or data deletion stay with the orchestrator.

The plugin ships one skill, two subagents, and one setup command.

- `delegated-coding` is a skill that loads on any coding task and holds the
  routing rule, the handoff rules, and the escalation policy.
- `code-writer` is a Haiku subagent that writes and edits application code.
- `test-writer` is a Haiku subagent that writes tests and test scaffolding.
- `/enterprise-token-saver:setup` detects your repository's gate commands and
  writes a configuration file.

## Install

Run these two commands.

```
claude plugin marketplace add hannasage/enterprise-token-saver
claude plugin install enterprise-token-saver@enterprise-token-saver
```

Restart Claude Code. Then open the repository you want to use it in and run:

```
/enterprise-token-saver:setup
```

Setup asks a few questions and writes `.claude/delegated-coding.json`. The
plugin works without that file. With the file, it stops rereading your
repository documentation to find the test and lint commands.

Inside a session, the subagents are addressed as
`enterprise-token-saver:code-writer` and
`enterprise-token-saver:test-writer`.

## How a task flows

1. The orchestrator reads the request and breaks it into subtasks.
2. For each subtask, the orchestrator applies the decision test. A subtask a
   careful junior engineer can finish from a brief alone goes to Haiku. Every
   other subtask stays in the orchestrator.
3. The orchestrator writes a complete brief for each delegated subtask. The
   brief names the goal, the exact file paths, and the exact signatures. It also
   names the pattern to copy with a line range, the constraints, the acceptance
   commands, and the return format. A brief missing any of those is not
   dispatched.
4. The orchestrator dispatches the brief to `code-writer` or `test-writer`.
5. The subagent writes the code and runs the acceptance commands. It reports
   the files it changed, the commands it ran, and the exit codes.
6. If the brief has a gap, the subagent writes nothing and returns a
   `QUESTIONS:` block instead.
7. The orchestrator validates the result. It runs the acceptance commands
   itself and reads the full diff. The subagent's own report is not proof.
8. On a failure, the orchestrator either fixes the diff in place or redispatches
   once with a corrected brief. The retry cap is one.
9. On a second failure, the orchestrator does the subtask itself. There is
   never a third dispatch on the same subtask.
10. At the end of the task, the orchestrator prints a routing summary.

The routing summary looks like this.

```
Routing summary
| Subtask | Model | Result |
|---|---|---|
| POST /invoices handler | haiku | first pass |
| Invoice total rounding | orchestrator | escalated |
| Handler tests | haiku | retry |
```

## Configuration

Setup writes `.claude/delegated-coding.json` in your repository root. Every key
is optional.

```json
{
  "gates": ["npm test", "npm run lint", "npm run typecheck"],
  "log": "docs/delegation-log.md",
  "caveman": false,
  "delegate": ["OpenAPI client regeneration"],
  "keep": ["database migrations"]
}
```

`gates` holds an array of shell commands that must exit 0. The orchestrator
copies them into every brief as the acceptance criteria. Without this key, the
orchestrator reads your `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `README.md`,
`package.json` scripts, `Makefile`, `pyproject.toml`, and continuous integration
configuration to find the commands. It never invents a command it did not read.

`log` holds a file path. When set, the orchestrator appends the routing summary
to that file with a date line. Without this key, the summary prints in the
transcript only.

`caveman` holds true or false. Setup writes it to record whether you enabled the
optional plugin described below. Nothing in this plugin acts on the value.

`delegate` holds free-text task types. The orchestrator reads them in addition
to the built-in delegate list.

`keep` holds free-text task types. The orchestrator reads them in addition to
the built-in keep list. A task type listed in both keys stays with the
orchestrator.

## Caveman option

Caveman is a separate MIT-licensed plugin by Julius Brussee. It compresses the
prose in Claude's replies while keeping code, commands, and error text
byte-exact. Its authors claim about 65 percent average savings on output tokens.

The trade-off has two sides. Caveman changes output tokens only, and it adds
input-token overhead on every turn. On a task whose output is already short, it
can cost more than it saves.

The setup command asks whether you want it. If you say yes, setup installs it as
its own plugin using its own published install command, and records
`"caveman": true` in the configuration. This plugin records the choice and
nothing more. It does not bundle caveman, patch it, or reimplement it. If you
say no, nothing is installed.

## Tuning the split

Two ways to change the routing exist.

1. Edit the two lists in `skills/delegated-coding/SKILL.md`. That change applies
   in every repository where the plugin is installed.
2. Add the task type to `delegate` or `keep` in
   `.claude/delegated-coding.json`. That change applies in one repository.

Use the log to decide what to move. Move a task type into the delegate list
after the orchestrator has done it three times with no judgment call. Move a
task type back into the keep list after a subagent has failed it twice, or after
one failure reached production code.

## Cost note

Haiku output tokens cost a fraction of Sonnet output tokens. Current rates for
every model are at https://www.anthropic.com/pricing.

The size of the win depends on the first-pass success rate. A brief that lands
on the first pass moves its output tokens from the expensive model to the cheap
one. A brief that fails costs the Haiku tokens, plus the orchestrator tokens
spent reading the diff, plus the tokens spent on the retry. A low first-pass
rate can make a task cost more than it would have cost undelegated.

Two things raise the first-pass rate. Write complete briefs, and delegate only
work that passes the decision test. Read the routing summary and the log to see
your real rate.

## License

MIT. See `LICENSE`.
