---
description: Detect this repository's gate commands, ask about the optional caveman plugin, and write .claude/delegated-coding.json
---

Set up delegated coding in the current repository. Work through the five steps
below in order. Ask the user before you write anything.

## Step 1: Detect the gate commands

A gate is a shell command that must exit 0 before work counts as done.

1. Read the repository's documentation and manifests. Check `CLAUDE.md`,
   `AGENTS.md`, `CONTRIBUTING.md`, `README.md`, the `scripts` block in
   `package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, `go.mod`, and
   the continuous integration configuration under `.github/workflows/`.
2. Pick the test command, the lint command, and the typecheck command.
3. Record where you found each one.
4. Use only a command you read in a file. Never invent a command.
5. Show the user the proposed list. Name the source file for each command.
6. Report any of the three you could not find. Ask the user for it. Do not
   guess.
7. Ask the user to confirm the list or edit it. Wait for the answer.

## Step 2: Ask about caveman

Caveman is an optional third-party plugin. This plugin does not require it.

Present this trade-off to the user, in three sentences.

1. Caveman is an MIT-licensed third-party plugin that compresses prose while
   keeping code, commands, and error text byte-exact.
2. Its authors claim about 65 percent average savings on output tokens.
3. It changes output tokens only, adds input-token overhead on every turn, and
   can cost more than it saves on tasks whose output is already terse.

Offer two options.

1. Enable now.
2. Skip for now.

Say that the user can enable it later by running this command again. Wait for
the answer.

## Step 3: Install caveman, only if the user enabled it

If the user chose "skip for now", go to step 4 and record `"caveman": false`.

If the user chose "enable now", do these three things and nothing else.

1. Run this command:
   `claude plugin marketplace add JuliusBrussee/caveman && claude plugin install caveman@caveman`
2. Report the exact output, including any error.
3. Record `"caveman": true` in the configuration in step 5.

Install caveman as its own plugin. Never copy its files into this repository.
Never reimplement its behavior in a skill, a rule file, or a prompt. Follow its
own documentation for anything beyond the install command above.

## Step 4: Ask for a log path

The routing summary is a short table that records which model did which
subtask. It prints in the transcript at the end of every coding task. A log
path makes it append to a file as well.

1. Ask whether the user wants a log file.
2. Suggest `docs/delegation-log.md` as a default.
3. Accept a different path, or accept no log at all.

## Step 5: Write the configuration

1. Create `.claude/` in the repository root if it does not exist.
2. Write `.claude/delegated-coding.json`.
3. Include `gates` from step 1 and `caveman` from step 2.
4. Include `log` only when the user gave a path in step 4.
5. Read the file back and show it to the user.
6. Tell the user that `delegate` and `keep` are two more optional keys, and
   that the `delegated-coding` skill documents them.

The file looks like this.

```json
{
  "gates": ["npm test", "npm run lint", "npm run typecheck"],
  "caveman": false,
  "log": "docs/delegation-log.md"
}
```

If `.claude/delegated-coding.json` already exists, show the current file first.
Ask whether to update it before you overwrite it.
