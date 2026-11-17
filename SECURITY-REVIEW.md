# revmux: how claude and codex are used, and data-exposure review

Scope of this review: `app/executor/`, `app/prompt/`, `app/revmux/config.go`, and the markdown files shipped in the
repository (prompts, lenses, skill trees, `.claude/rules`).
Not reviewed: `app/pipeline/`, the shell and Python scripts under `scripts/` in both skill trees, vendored dependencies.

## 1. How the model CLIs are used

revmux runs a review by spawning `claude --print` and `codex exec` as child processes and supervising them
(idle watchdog, kill, retry). Both are launched from a single place, `app/executor/executor.go:119`
(`exec.CommandContext`). The prompt goes in through **stdin** (`app/executor/proc.go:142`); argv carries only flags,
model and effort. The verbatim stdout of every agent is tee'd into the run archive (`agents/*.jsonl`, `agents/*.log`).

**claude** (`app/executor/claude.go:74-94`):

```
--print --output-format stream-json --verbose --permission-mode auto
--tools=Bash,Read,Grep,Glob,WebFetch,WebSearch
--strict-mcp-config --setting-sources project --disable-slash-commands
--no-session-persistence --include-partial-messages
[--model ...] [--effort ...] [--json-schema ...]
```

**codex** (`app/executor/codex.go:110-122`):

```
codex exec --skip-git-repo-check --sandbox read-only [-m <model>] [-c model_reasoning_effort=<effort>]
```

The sandbox is configurable with `--codex-sandbox` (`read-only` | `workspace-write` | `danger-full-access`,
`app/revmux/config.go:68`).

## 2. What goes into the AI processes as input

Prompt composition (`app/prompt/compose.go`):

1. profile body (preamble, severity bar, "what not to report");
2. the bodies of the lenses the agent carries;
3. `{{VAR}}` substitution (an unknown variable is an error);
4. an injected prior-rounds block (task directory path plus one generated line per round, counts only) followed by
   a fixed "re-evaluate independently" guard; omitted on a first round;
5. codex only: an appended output contract with the JSON schema (claude gets `--json-schema` instead);
6. claude only: a narration contract (one short line per finding) so the watchdog can see liveness.

Variables expand to **absolute paths, never to content**:

| Variable | Value |
|---|---|
| `{{SCOPE}}` | path of `input/scope.md` |
| `{{GOAL}}` | path of `input/goal.md` |
| `{{PROFILE}}` | path of the profile copy (`prompts/input-profile.md`) |
| `{{CONTEXT}}` | path of `input/context/` |
| `{{WORKDIR}}` | working directory |

Source code and diffs reach the model through the agent's own tools (`Read`, `Grep`, `Bash`), not through the prompt.
The agent opens `scope.md`, runs the diff command written there, and reads whatever else it needs.

Per stage:

- **find**: profile + lenses; the agent reads scope, goal, profile, context and the code itself.
- **synthesis**: receives `{{SOURCES}}` and `{{FINDINGS}}`, the **text of every agent's findings embedded in the
  prompt**. It is told not to open files.
- **verify**: receives `{{SCOPE}}`, `{{PROFILE}}`, `{{WORKDIR}}` and `{{FINDINGS}}` for its assigned group, and checks
  them against the code with tools.

## 3. Network calls and data-leak surface

revmux's own Go code makes **no network calls**: no `net/http`, `net.Dial` or similar outside tests and vendor.
All outbound traffic goes through the child CLIs, which of course send everything they read to Anthropic / OpenAI.

Findings, most serious first:

1. **claude agents have `WebFetch` and `WebSearch` together with `Bash`, under `--permission-mode auto`.**
   No confirmations. Prompt injection in reviewed code, a diff or a PR description can make an agent read a secret
   (`.env`, keys) and send it to an arbitrary URL.
2. **The environment is inherited almost whole** (`app/executor/proc.go:200-212`). Only `CLAUDECODE` and
   `ANTHROPIC_API_KEY` (unless `--preserve-anthropic-api-key`) are removed. `GITHUB_TOKEN`, `AWS_*`,
   `OPENAI_API_KEY` and the rest are visible to the child and to its `Bash` tool. This is a denylist, not an allowlist.
3. **`--setting-sources project` loads `.claude/settings*.json` from the working directory.** For an untrusted
   repository or PR, its project settings (including hooks) may run. Fine for a trusted repo, unsafe for a foreign one.
4. **codex**: default `read-only` sandbox, but `--codex-sandbox danger-full-access` is a valid value. Network access
   under `read-only` is normally disabled, but this was **not verified**. Env and files are read the same way as
   for claude.
5. **Run archive on disk.** `agents/*.jsonl` holds the verbatim agent output, which can include file fragments the
   agents read. If `.revmux/tasks` ends up in git or in CI artifacts, secrets leak from there.
   Whether the directory is git-ignored was **not checked**.
6. **Reach is not limited to the working directory.** `Read` and `Bash` can open absolute paths outside it, and
   anything read is sent to the provider.
7. **Findings are forwarded.** A finding that quotes a secret from code is sent again to the synthesis and verify
   agents. Because `{{FINDINGS}}` is model-authored text fed to another model that has `Bash`, it is also a channel
   for indirect prompt injection between stages.

Already done well: `--strict-mcp-config` with no `--mcp-config` disables MCP; no `Edit`, `Write` or `Task` tools for
reviewers; `--disable-slash-commands` and `--no-session-persistence`; context passed as paths; no git or repo reading
inside revmux itself.

## 4. Markdown files review (63 files)

No explicitly malicious content found: no hidden "ignore previous instructions", no `curl`/`wget`, no `.ssh`/`.aws`
access, no exfiltration, no obfuscation (base64, eval). All URLs point to `github.com/umputun/revmux` or
`openai/codex` (documentation and install).

Risks that follow from the design rather than from hostile text:

1. **"Read-only" is enforced only by prose** in `comprehensive.md:14`, `verify.md` and `synthesis.md`. For claude it
   is not backed by tool restrictions, since `Bash` is available without prompts. Good against model mistakes, weak
   against injection.
2. **`comprehensive.md:23-24` tells the agent to run the command found in scope itself.** The command is written by
   the caller into `scope.md`. If it is built from PR data (branch name, URL, description), command injection is
   possible and the agent executes it in `Bash`.
3. **`SKILL.md:5` grants `allowed-tools: [Bash, Read, Edit, Write, Grep, Glob, Agent, AskUserQuestion, Monitor, TaskStop]`.**
   Broad rights for the calling session; expected for this skill, but it writes files and runs scripts.
4. **Install instructions** (`SKILL.md:118-120`) use `go install ...@latest`, which does not pin a version.
5. The shell and Python scripts the skill asks to run are the more important surface than the markdown, and were not
   read in this review.

## 5. Recommendations

- Drop `WebFetch,WebSearch` from `--tools=` in `app/executor/claude.go:79` unless the review really needs them.
- Build the child environment from an allowlist instead of removing a few names.
- Do not review untrusted PRs without isolation (container or clean environment); consider a different
  `--setting-sources` for untrusted input.
- Make sure `scope.md` commands are never assembled from untrusted PR fields.
- Add `.revmux/` to `.gitignore` and keep it out of CI artifacts.
- Pin versions in install instructions.
- Follow-up: review `app/pipeline/` (how `{{FINDINGS}}` and history are assembled), the `scripts/` directories in both
  skill trees, and verify codex network behaviour under each sandbox mode.
