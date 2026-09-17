# claude-agents-python

Seven role-scoped Claude Code subagents for solo Python development. Write permissions are deliberately split by role — the implementer can't touch tests, the test author can't touch source — so you stay the orchestrator.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

## What this is

A set of Markdown files you drop into `.claude/agents/`. Each defines a specialist with its own system prompt, tool allowlist, and model. Claude delegates to them; you decide what gets built and which findings matter.

Solo development has no second pair of eyes and no division of labor. A single agent doing everything will, sooner or later, weaken a test to make it pass or refactor something you never asked about. Splitting the work into roles with different permissions makes those failures structurally impossible rather than merely discouraged.

## The design: asymmetric write permissions

This is the part that matters more than the role names.

| Agent | Can write to | Model |
| --- | --- | --- |
| `librarian` | nothing | sonnet |
| `implementer` | source only | sonnet |
| `test-author` | `tests/` only | sonnet |
| `test-runner` | nothing | haiku |
| `debugger` | anywhere | opus |
| `reviewer` | nothing | opus |
| `documenter` | docs and docstrings | sonnet |

`implementer` cannot edit `tests/`, so it can never make a failing test pass by rewriting the test. `test-author` cannot edit source, so it can never bend a test to match a buggy implementation. `reviewer` and `test-runner` carry `disallowedTools: Write, Edit`, so critique stays separate from correction.

When one agent holds every permission, these failures happen quietly. Separating them forces every real decision back to you.

Model assignment follows the same logic: opus where judgment drives the outcome, haiku where the job is summarizing output.

## Quick start

```bash
git clone https://github.com/<user>/claude-agents-python.git
cd claude-agents-python

# Option A — available in every project
mkdir -p ~/.claude/agents
cp agents/*.md ~/.claude/agents/

# Option B — this project only, shareable via git
cd /path/to/your-project
mkdir -p .claude/agents
cp /path/to/claude-agents-python/agents/*.md .claude/agents/
cp /path/to/claude-agents-python/CLAUDE.md.template ./CLAUDE.md
```

**Restart Claude Code after creating the `agents/` directory for the first time.** A directory that didn't exist at startup isn't watched, so the agents won't be found until you restart. Adding or editing files in an existing directory is picked up within seconds.

Verify with `ls .claude/agents/`, then try one:

```
Use the test-runner subagent to run the suite
```

## Where the files go

Project-level agents live at the root of the folder you open in your editor:

```
your-project/
├── .claude/
│   ├── agents/          ← the seven definitions
│   ├── agent-memory/    ← created on first use
│   └── settings.json    ← optional, shared permissions and hooks
├── CLAUDE.md            ← at the root, NOT inside .claude/
├── pyproject.toml
├── src/
└── tests/
```

`CLAUDE.md` is loaded by every custom subagent (the built-in Explore and Plan agents are the exception). Put shared conventions there once rather than repeating them in each definition.

If a project-level and a user-level agent share a name, the project one wins. That lets you keep a general set in `~/.claude/agents/` and override individual roles per repository.

## Environment support

| | `~/.claude/agents/` | `.claude/agents/` | Extra work |
| --- | --- | --- | --- |
| CLI | ✅ | ✅ | none |
| VS Code extension | ✅ | ✅ | none |
| Claude Code on the web | ❌ | ✅ | commit the files; install your toolchain in the environment setup script |

Cloud sessions start from a fresh VM with your repository cloned into it, so your home directory isn't there. Commit `.claude/agents/` and `CLAUDE.md` to use these on the web. Settings that live in `~/.claude/settings.json` won't apply either — use the cloud environment's variables instead.

## Working cycle

```
You decide what to build          ← never delegated
  ↓
librarian     compare library options
  ↓ you pick one
Plan mode     agree on an approach
  ↓ you approve
implementer   build it
  ↓
test-author   write the tests
  ↓
test-runner   run them
  ↓ on failure
debugger      find the root cause  → back to test-runner
  ↓ on green
reviewer      critique the diff
  ↓ you decide what to act on
documenter    update docstrings and README
```

Name the subagent explicitly when it matters. Leaving the choice to Claude works, but it means the routing decision isn't yours.

Independent work can run in parallel:

```
Investigate the auth, database, and API modules in parallel subagents
```

Their reports come back into your main conversation, so running many verbose agents at once spends the context you were trying to save. Keep parallelism to genuinely independent research.

## Adjust for your toolchain

Two places assume **uv + ruff + mypy + pytest**. If that's not your setup, change them before first use — otherwise the agents will call commands that don't exist.

1. `CLAUDE.md` — the toolchain table
2. `agents/implementer.md` — the post-implementation verification commands

Optional: `agents/documenter.md` if you use NumPy-style docstrings, `agents/test-runner.md` if you run tests through nox or tox.

## Cost

`reviewer` and `debugger` run on opus, which is where most of the spend goes. To lower everything at once, in `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "sonnet",
    "CLAUDE_CODE_SUBAGENT_MODEL_FORCE": "1"
  }
}
```

`FORCE` overrides the `model:` field in every definition. Drop it to let each definition keep its own choice.

`/usage` breaks down consumption per subagent and flags subagent-heavy or highly parallel sessions, which is the fastest way to find out which of the seven is actually expensive for your workload.

## Accumulated knowledge

`reviewer` and `librarian` are set to `memory: project`. Findings collect in `.claude/agent-memory/<agent>/` and carry across sessions. Commit that directory if you want it to follow you between machines; leave it ignored if you'd rather keep review notes out of the repository history.

## Enforcing the boundaries

The write restrictions above are instructions, not guarantees. If you need one enforced mechanically, add a `PreToolUse` hook to that agent's frontmatter and exit with code 2 to block the call:

```yaml
hooks:
  PreToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/deny-tests-dir.sh"
```

Start without hooks. Add one only to the role that actually crosses a line.

## Troubleshooting

**Agents don't show up.** You created `agents/` during a running session — restart. Then check the path and that the frontmatter's `---` fences are closed.

**Delegation goes to the wrong agent.** `description` is what Claude routes on. Say when to use it, concretely. Name the agent explicitly when you need certainty.

**Warning about long descriptions at startup.** The combined `description` fields exceed the token budget. Move detail into the body; keep `description` to one or two sentences about when to use the agent.

**An agent edited something it shouldn't have.** Add a hook, as above.

**Higher token usage than expected.** Subagent reports return to the main conversation. Reduce parallelism and prefer agents that summarize (`test-runner`) over ones that dump.

**An agent doesn't know something obvious.** Non-forked subagents start with a clean context and no conversation history. State the premise in the request, or put it in `CLAUDE.md` if it's permanent.

## Not included, on purpose

- **planner** — Plan mode and the built-in Plan agent cover this, and deciding the approach is the part you should keep.
- **explorer** — the built-in Explore agent handles codebase search. `librarian` is scoped to external libraries so the two don't compete for the same delegations.

## Layout

```
claude-agents-python/
├── README.md
├── LICENSE
├── CLAUDE.md.template
└── agents/
    ├── librarian.md
    ├── implementer.md
    ├── test-author.md
    ├── test-runner.md
    ├── debugger.md
    ├── reviewer.md
    └── documenter.md
```

## License

MIT — see [LICENSE](./LICENSE).

These files are prompts rather than executable code, so treat them accordingly: copy them, rewrite them, strip out the parts that don't fit your workflow, and ship the result however you like. Attribution is appreciated but the license asks for little beyond keeping the copyright notice with substantial copies.
