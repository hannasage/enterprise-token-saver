---
name: delegated-coding
description: >
  Route mechanical code work to Haiku subagents and keep judgment work in the
  main session. Load this skill for any task that writes, changes, refactors,
  reviews, or tests code, including new features, bug fixes, endpoints,
  migrations, scripts, and test suites.
---

# Delegated coding

## Purpose

This skill cuts the cost of a coding session. It moves mechanical code
generation to Claude Haiku subagents. The main session stays on Sonnet and
keeps the work that needs judgment.

Three terms are used throughout.

- Orchestrator: the main session that reads the repository, plans, dispatches
  work, and validates results.
- Subagent: a separate Claude session that the orchestrator starts with one
  written task and no access to the orchestrator's context.
- Brief: the written task handed to a subagent.

Haiku output tokens cost a fraction of Sonnet output tokens. Current rates are
at https://www.anthropic.com/pricing. The saving is real only when the subagent
succeeds on the first pass. A failed dispatch costs the Haiku tokens plus the
orchestrator tokens spent fixing it.

This plugin ships two subagents.

- `enterprise-token-saver:code-writer` writes and edits application code.
- `enterprise-token-saver:test-writer` writes tests and test scaffolding.

## Routing rule

Keep this work in the orchestrator.

- Planning and task breakdown.
- Architecture decisions and choices between two valid designs.
- Interpreting an ambiguous specification or an unclear request.
- Root-cause analysis when debugging.
- Code review of any diff, including a subagent's diff.
- Security-sensitive code.
- Anything touching authentication, authorization, payments, billing, or data
  deletion.
- Any task where the brief cannot be made complete.

Send this work to Haiku through `code-writer` or `test-writer`.

- Boilerplate and scaffolding.
- CRUD endpoints that follow an existing endpoint in the same repository.
- Test scaffolding and test cases against a named framework.
- Repetitive refactors and renames across many files.
- Function implementations against a signature the orchestrator already fixed.
- Configuration files and schema definitions.
- Data mappers and type converters.
- Test fixtures and sample data.
- Docstrings and inline comments for existing code.

Apply one decision test to every subtask.

> Could a careful junior engineer do this from the brief alone, with zero
> questions?

If the answer is yes, delegate it. If the answer is no, keep it or split it.
A split means the orchestrator makes the decisions first, writes them into the
brief, and delegates what is left.

## Handoff rules

Write a complete brief before every dispatch. Use `brief-template.md` in this
skill directory. The brief must contain all seven fields.

1. Goal, in one sentence.
2. Exact file paths to create or edit.
3. Exact function, type, and interface signatures.
4. The existing code or pattern to copy, given as a file path with a line
   range, or as a pasted snippet.
5. Constraints, such as no new dependencies and the repository style rules.
6. Acceptance criteria, written as exact shell commands that must exit 0.
7. What to return: files changed, commands run, and command output.

A brief missing any field is not dispatched. Fill the gap first. If a gap
cannot be filled, the task fails the decision test and stays with the
orchestrator.

## Gate commands

A gate is a shell command that must exit 0 before work counts as done. Gate
commands differ per repository. Resolve them in this order.

1. Read `.claude/delegated-coding.json` in the repository. If the file exists
   and has a `gates` key, use that array of commands.
2. If the file is absent or has no `gates` key, read the repository's own
   documentation and manifests. Look at `CLAUDE.md`, `AGENTS.md`,
   `CONTRIBUTING.md`, `README.md`, the `scripts` block in `package.json`,
   `Makefile`, `pyproject.toml`, `Cargo.toml`, `go.mod`, and the continuous
   integration configuration. Pick the test, lint, and typecheck commands from
   what you find. Write them into the brief.

Never invent a gate command. Use only a command you read in the repository. If
no command is found, say so and ask the user for one.

## Validation and escalation

The orchestrator validates every returned result. The subagent's own report is
not proof.

1. Run the acceptance commands yourself. Record the exit codes.
2. Read the full diff the subagent produced.
3. Check that no file outside the brief changed.

If the gates pass and the diff is correct, accept the work and move on.

If a gate fails, or the diff is wrong, take one of two paths.

1. Fix it in place yourself, when the fault is small and clear.
2. Redispatch once, with a corrected brief that names the exact fault and adds
   the detail that was missing.

The retry cap is one. A second failure ends delegation for that subtask. The
orchestrator does the work itself. Never dispatch a third time on the same
subtask.

## Subagent questions

A subagent that finds a gap returns a `QUESTIONS:` block and writes no code.
Handle it in three steps.

1. Answer each question by amending the brief.
2. Redispatch the amended brief.
3. Count that redispatch as the one allowed retry.

A `QUESTIONS:` block means the brief was incomplete. Treat it as a signal to
write better briefs, not as a subagent failure.

## Routing summary

Print this block in the transcript at the end of every coding task.

```
Routing summary
| Subtask | Model | Result |
|---|---|---|
| ... | haiku / orchestrator | first pass / retry / escalated |
```

Use one row per subtask. The `Model` cell holds `haiku` or `orchestrator`. The
`Result` cell holds `first pass`, `retry`, or `escalated`.

If `.claude/delegated-coding.json` has a `log` key, append the same block to
that file. Add a date line above the block. Create the file and its parent
directory if they do not exist.

## Tuning the split

Move an item between the two lists when the evidence says to move it.

1. Move a task type from the keep list to the delegate list when the
   orchestrator has done it three times with no judgment call.
2. Move a task type from the delegate list to the keep list when a subagent has
   failed it twice, or when one failure reached production code.

Two ways to record a change exist.

1. Edit the lists in this file. That change applies everywhere the plugin is
   installed.
2. Write the change into `.claude/delegated-coding.json` in one repository. That
   change applies to that repository only.

The configuration file has four keys. All four are optional.

```json
{
  "gates": ["npm test", "npm run lint"],
  "log": "docs/delegation-log.md",
  "delegate": ["extra task types"],
  "keep": ["task types to pull back"]
}
```

- `gates` holds shell commands that must exit 0. It overrides the commands read
  from repository documentation.
- `log` holds a path for the routing summary log.
- `delegate` holds free-text task types to add to the delegate list.
- `keep` holds free-text task types to add to the keep list.

Read `delegate` and `keep` in addition to the built-in lists above. When a task
type appears in both `delegate` and `keep`, keep it in the orchestrator.

The file can also hold a `caveman` key. The setup command writes it. It records
whether the user enabled a separate third-party plugin. This skill does not act
on it.
