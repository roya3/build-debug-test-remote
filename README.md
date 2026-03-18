# build-debug-test-remote README

## Overview

`build-debug-test-remote` is an skill for remote build, run, test, and debug workflows over SSH. The skill enforces directory confinement, redaction of sensitive values, controlled command categories, just-in-time approval before sensitive transmission, and refusal of unsafe or out-of-scope actions.

Use it when:
- remote execution on an approved host is required
- local toolchain commands have failed
- local toolchain commands cannot be used
- the next reasonable step is remote verification or remote debugging

This skill is restricted to one approved remote directory per invocation.

## What this skill does

For each invocation, the skill:

1. connects to the approved remote host
2. enters the approved remote directory
3. runs one related batch of build, run, test, debug, or verification commands
4. collects results
5. ends remote access for the current invocation

## What this skill does not do

For safety, this skill does **not** do any of the following:

- it does not allow arbitrary remote shell access outside the approved directory
- it does not sync, clone, pull, fetch, copy, or deploy code
- it does not allow commands that write outside the approved directory
- it does not allow privileged execution such as `sudo`
- it does not allow destructive commands such as `rm -rf`
- it does not allow path traversal using `..`
- it does not allow silent fallback to unsafe ad hoc remote execution
- it does not treat `REMOTE_DEBUG_DIR` as a local path, and it does not permit host-side path rewriting before remote execution
- it does not expose sensitive connection values in logs, prompts, summaries, or structured output without explicit just-in-time approval
- it does not keep SSH sessions open across future invocations

## Required environment variables

Set these host environment variables before use:

- `REMOTE_DEBUG_HOST`
- `REMOTE_DEBUG_PORT`
- `REMOTE_DEBUG_USER`
- `REMOTE_DEBUG_DIR`

### Important rule for `REMOTE_DEBUG_DIR`

`REMOTE_DEBUG_DIR` must remain an unchanged remote POSIX absolute path, for example:

```text
/opt/project/debug_workspace
```

Do not rewrite it into a local-path form before remote execution.

## Installation

Place the skill file at:

```text
./skills/build-debug-test-remote/SKILL.md
```

## Version

This README matches:
- skill name: `build-debug-test-remote`
- skill version: `1.2.8`
- skill author: Roy
