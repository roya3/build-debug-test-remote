---
name: build-debug-test-remote
description: Use this skill whenever build, run, test, or debug is needed, especially if local toolchain commands cannot be used or have failed.
metadata:
  author: Roy
  version: 1.2.8
---

# Build Debug Test Remote Skill

Use this skill for build, run, test, and debug on an remote host.

Load this skill when local build, compile, run, test, or debug commands such as `make`, `gcc`, `cc`, `cmake`, `ninja`, `gdb`, or local test runners are unavailable, missing, or have failed.

## Trigger Conditions

Use this skill when any of the following is true:

1. build, run, test, or debug is required, but local build, compile, run, test, or debug commands are unavailable, missing, have failed, or cannot be used
2. remote build, run, test, or debug is required
3. the next reasonable step is remote verification, remote reproduction, remote build, remote run, remote test, or remote debugging
4. local work has progressed to the point where remote validation is the next likely step
5. the remote environment is authoritative for final verification

## Mandatory Usage

Load this skill before any  build, compile, run, test, debug, log inspection, or approved-directory file inspection task on the remote host.

Also load this skill as a fallback when local toolchain execution is unavailable, has failed, or cannot be used.

## Goal

For each skill invocation:
- establish SSH access to the approved remote host
- enter only the approved remote directory
- run one related batch of build, run, debug, test, or verification commands
- collect results
- end remote access for the current invocation

Never sync code.

## Execution Model

Preferred mode:
- one short-lived interactive SSH batch session for the current invocation

Compatible fallback mode:
- if the execution channel cannot reliably preserve one interactive SSH batch session, run the same batch using controlled non-interactive SSH commands
- in fallback mode, each command must explicitly re-enter the approved directory
- fallback mode must preserve the same safety, redaction, and approved-directory constraints

Do not silently downgrade to unsafe ad hoc command execution.

## Runtime Configuration and Redaction

Read runtime configuration from host environment variables only:

- `REMOTE_DEBUG_HOST`
- `REMOTE_DEBUG_PORT`
- `REMOTE_DEBUG_USER`
- `REMOTE_DEBUG_DIR`

If any required environment variable is missing or empty, stop and report failure.

These variables exist only in the host process environment.
Do not assume they exist on the remote server after SSH login.

Use them only for:
- SSH connection setup
- approved directory entry
- approved directory comparison

Do not echo them in:
- prompts
- summaries
- logs
- diagnostic output
- final structured output

Never run:
- `env`
- `printenv`
- `set`
- `echo $REMOTE_DEBUG_HOST`
- `echo $REMOTE_DEBUG_USER`
- `echo $REMOTE_DEBUG_DIR`

When referring to the remote endpoint outside the actual execution command, use:
- `approved remote host`
- `approved remote target`
- `[REDACTED]`

If a command transcript must be referenced, redact:
- host
- user
- port
- target directory

## Just-in-Time Approval for Sensitive Information

Before any step that would write sensitive information into:
1. the skill body
2. logs
3. prompts
4. command text or command results that will be submitted to a model after direct value expansion

explicit user approval is required immediately before that transmission.

Approval must identify the exact sensitive information to be sent.

If approval is not granted:
- do not continue that execution path
- do not transmit the information
- exit safely

## Approved Directory Literal Rule

`REMOTE_DEBUG_DIR` must be treated as an opaque remote POSIX path literal.

It is not a local filesystem path.
It must not be normalized, rewritten, expanded into a local-prefixed form, or converted into any host-specific path syntax before remote execution.

Required form:
- POSIX absolute path beginning with `/`

Disallowed forms include:
- drive-letter paths such as `C:\...`
- backslash-separated Windows paths
- converted local-prefix forms
- paths prefixed with a local installation directory

If the approved directory value is transformed before remote execution, stop immediately.

## Approved Directory Entry Rule

To avoid host-shell path rewriting, enter the approved directory on the remote host using a positional argument.

Never rely on:
- `cd "$REMOTE_DEBUG_DIR"` executed remotely
- direct host-side string interpolation of the approved directory into the remote command body

Required interactive command shape:
- `bash -lc 'cd -- "$1" && exec bash -l' _ "<approved_dir_literal>"`

Required fallback command shape:
- `bash -lc 'cd -- "$1" && <substantive_command>' _ "<approved_dir_literal>"`

Where:
- `"<approved_dir_literal>"` is the exact host-captured value of `REMOTE_DEBUG_DIR`
- the remote shell receives that value as `$1`
- the remote shell performs the directory change itself

## Host Environment Detection

Detect the host environment first:
- Windows
- Linux
- macOS
- Unknown

Use conservative signals only.

If the host environment cannot be safely classified, stop and report failure.

## Local Shell Requirement

Any shell or terminal environment is acceptable if it can invoke `ssh` correctly and preserve `REMOTE_DEBUG_DIR` unchanged before remote execution.

The shell itself is not the safety boundary.
The real requirement is:
- `ssh` must be available
- the execution channel must preserve `REMOTE_DEBUG_DIR` as an unchanged remote POSIX absolute path literal

If the current shell, wrapper, or terminal environment rewrites the approved directory path before remote execution, stop and report failure.

## Remote Shell Requirement

This skill assumes `bash` for remote command execution.

If `bash` is unavailable on the remote host, stop and report failure.

## Standard Procedure

### Step -1: Approval Check

Before any step that would write sensitive information into logs, prompts, or model-submitted command text/results after direct value expansion, identify the exact sensitive information to be sent and obtain explicit user approval.

If approval is not granted, stop immediately.

### Step 0: Validate Runtime Environment Variables

Check that all required variables exist and are non-empty:
- `REMOTE_DEBUG_HOST`
- `REMOTE_DEBUG_PORT`
- `REMOTE_DEBUG_USER`
- `REMOTE_DEBUG_DIR`

Capture the approved directory literal value from the host environment before continuing.

Validate that the approved directory:
- is non-empty
- begins with `/`
- remains a POSIX-style absolute path literal
- does not contain backslashes
- does not start with a Windows drive prefix
- does not appear in a converted local-prefix form

Do not normalize, rewrite, canonicalize, or otherwise transform the path on the host side.

### Step 1: Detect Host Environment

Determine whether the host is Windows, Linux, macOS, or Unknown.

### Step 2: Verify SSH Availability

Check whether `ssh` is available using any shell-appropriate method.
If not available, stop and report failure.

### Step 3: Choose Execution Mode for This Invocation

Prefer one short-lived interactive SSH batch session.

If the execution channel can preserve that session for the whole batch:
- use `interactive_batch`

Otherwise:
- use `fallback_non_interactive`
- every fallback command must explicitly re-enter the approved directory using the positional-argument rule
- the batch must still behave as one logical batch for safety checks and reporting

### Step 4: Enter Approved Directory

#### Interactive batch mode

Establish one SSH session for the current batch and enter the approved remote directory in the same login flow.

Required behavior:
- connect to the approved remote host
- pass the approved directory as a positional argument
- execute `cd -- "$1"` on the remote host
- then start the remote shell for the batch

If SSH login fails, stop immediately.
If changing into the approved directory fails, stop immediately.

#### Fallback mode

For each command in the current batch:
- connect with SSH
- pass the approved directory as a positional argument
- enter the approved directory on the remote host
- execute the command
- collect the result

If the directory argument is observed in a converted local-prefix form before remote execution, stop immediately.

### Step 5: Validate Remote Directory

Validate that execution is occurring in the approved directory.

Preferred check:
- run `pwd`

If `pwd` exactly matches the approved directory string, continue.

If `pwd` does not exactly match:
- run `pwd -P` only if the mismatch might be caused by a symlink or logical-versus-physical path difference
- continue only when the mismatch is clearly attributable to another representation of the same approved directory
- otherwise stop immediately

In fallback mode, perform this validation at the start of the batch and again before sensitive build, debug, or test steps if needed.

### Step 6: Inspect Remote Workspace

Run only safe checks inside the approved directory, for example:
- `pwd`
- `ls -la`

In fallback mode, explicitly re-enter the approved directory in the same remote command before running them.

### Step 7: Run Requested Command Batch

Before each substantive build, run, debug, test, or log command, ensure the command runs from the approved directory.

Preferred method:
- explicitly re-enter the approved directory before each substantive command

Alternative method:
- verify with `pwd` that the current directory is still the approved directory, then run the command

Execute only approved commands and only inside the approved directory.

In fallback mode, remote command strings must never reference `"$REMOTE_DEBUG_DIR"` directly.
Use the unchanged host-captured approved directory literal as a positional argument.

If any command would leave the approved directory, stop and report failure.

### Step 8: End Remote Access for This Invocation

Preferred behavior:
- explicitly exit the remote SSH batch session at the end of the current invocation

Acceptable equivalent behavior:
- if the execution environment automatically closes the session or remote command channel at the end of the invocation, treat that as sufficient termination

Do not keep the SSH session open for future invocations.

### Step 9: Return Structured Result

Return:
- host environment
- selected shell
- execution mode used: `interactive_batch` or `fallback_non_interactive`
- whether SSH was available
- SSH target redacted
- approved directory redacted
- commands executed in sanitized form
- blocked commands
- exit codes
- key errors
- concise logs summary without secrets
- suggested next step

## Hard Rules

1. Never sync, clone, pull, fetch, rsync, scp, sftp, or copy code.
2. Never do any remote action before SSH login succeeds.
3. Never leave the approved remote directory.
4. Never use `..` in any remote command.
5. Never run privileged commands such as `sudo`.
6. Never run destructive commands.
7. If the approved remote directory does not exist, stop immediately.
8. If the resolved path cannot be confirmed as the approved remote directory, stop immediately.
9. Do not silently fall back to unsafe ad hoc command execution.
10. In fallback mode, every remote command must explicitly re-enter the approved directory before running the substantive action.
11. End remote access for the current invocation before the skill completes.
12. Final results must redact SSH destination and directory values.
13. Do not store raw command transcripts containing secrets in reusable notes or summaries.
14. Preserve one logical batch per invocation even when fallback mode uses multiple SSH commands.
15. If `bash` is unavailable on the remote host, stop and report failure.
16. Never write sensitive or user-specific information into the skill body, logs, prompts, or model-submitted command text/results after direct value expansion unless explicit user approval has been obtained beforehand.
17. Never convert `REMOTE_DEBUG_DIR` into a local filesystem path or any host-specific path before remote execution.
18. Treat `REMOTE_DEBUG_DIR` as an opaque remote POSIX literal and preserve it unchanged when constructing remote commands.

## Allowed Command Categories

Only allow commands needed for inspection, build, run, debug, test, and debugging inside the approved directory, such as:
- `pwd`
- `pwd -P`
- `ls -la`
- `find`
- `cat`
- `head`
- `tail`
- `grep`
- `make`
- `cmake`
- `ninja`
- project-local test runners invoked from the approved directory
- `gdb`
- `lldb`
- execution of binaries located in the approved directory
- log inspection of files inside the approved directory

These commands are allowed only when scoped to files and paths inside the approved directory.

## Blocked Commands

Refuse any of the following:
- `git clone`
- `git pull`
- `git fetch`
- `rsync`
- `scp`
- `sftp`
- `curl | sh`
- `wget | sh`
- `sudo`
- `rm -rf`
- `env`
- `printenv`
- any command containing `..`
- any command that writes outside the approved directory

## Output Format

```json
{
  "status": "success|failed|partial",
  "config_source": "environment_variables",
  "host_environment": "windows|linux|macos|unknown",
  "selected_shell": "powershell|pwsh|posix_shell|other|none",
  "execution_mode": "interactive_batch|fallback_non_interactive",
  "ssh_available": true,
  "ssh_target": "[REDACTED]",
  "target_dir": "[REDACTED]",
  "resolved_dir": "[REDACTED]",
  "commands_executed": ["sanitized command"],
  "blocked_commands": ["string"],
  "exit_codes": [{"command": "sanitized command", "code": 0}],
  "key_errors": ["string"],
  "logs_summary": "string",
  "next_step": "string"
}
```

## Refusal Conditions

Refuse immediately if:
- required runtime environment variables are missing
- host environment cannot be classified
- no working execution channel is available to invoke `ssh` without rewriting the approved directory path before remote execution
- `ssh` is unavailable
- SSH connection fails
- the approved directory is missing
- the final working directory cannot be confirmed as the approved directory
- the user asks to sync code
- a command includes `..`
- a command is destructive, privileged, or outside the approved directory
- explicit user approval has not been obtained before the relevant transmission when sensitive information would need to be written into the skill body, logs, prompts, or model-submitted command text/results after direct value expansion
- the approved directory value has been converted into a local-prefix form before remote execution
- the execution channel rewrites the approved POSIX path before remote execution
