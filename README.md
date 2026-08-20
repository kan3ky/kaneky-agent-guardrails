# kaneky-agent-guardrails

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-skill-6C4FF7)](https://docs.claude.com/en/docs/claude-code)
[![Dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)](#install)
[![Config](https://img.shields.io/badge/config-zero-brightgreen)](#install)

A Claude Code skill for designing and reviewing agent tool surfaces — the set
of capabilities an agent can call, and the gates around each one.

Most agent security advice is about prompts: what to tell the model not to
do. This is about what to build so the instruction is unnecessary — a tool
surface where the dangerous action either doesn't exist for this caller, or
waits for a human before it runs.

## Install

```sh
git clone https://github.com/kan3ky/kaneky-agent-guardrails
cp -r kaneky-agent-guardrails/skills/agent-guardrails ~/.claude/skills/
```

No configuration. No dependencies. Ask Claude Code to design a tool
registration, review a command-execution policy, add a new MCP tool, or
reason about what an agent can reach, and the skill loads itself.

## What it covers

| Question | Where |
|---|---|
| Should this be a refusal or an absent tool? | `SKILL.md` |
| Why "allow / ask / deny" beats a boolean | `SKILL.md` |
| Why rule order is itself a security property | `SKILL.md` |
| argv vs. a command line — why re-parsing args as shell syntax is a vulnerability class | `references/command-policy.md` |
| Quoting, `&&` vs. `&`, and why to refuse redirection and substitution instead of parsing them | `references/command-policy.md` |
| How one flag turns a safe binary into a write or exec primitive | `references/command-policy.md` |
| Why containment is a property of resolved paths, not program names | `references/command-policy.md` |
| Why widening a rule by prefix reopens what it used to block | `references/command-policy.md` |
| Inherit-minus-secrets vs. an allowlist of environment variables | `references/command-policy.md` |
| Testing a policy against the bypasses you thought of | `references/command-policy.md` |
| Registering tools per caller identity instead of globally | `references/tool-surface-design.md` |
| Separate credentials per privilege tier | `references/tool-surface-design.md` |
| Response caps that announce themselves instead of silently truncating | `references/tool-surface-design.md` |
| Tool descriptions as prompt surface, not just documentation | `references/tool-surface-design.md` |
| Why enforcement has to live server-side | `references/tool-surface-design.md` |
| Why scope allowlists should not be hot-reloadable | `references/tool-surface-design.md` |
| Designing an escape hatch deliberately instead of discovering one by accident | `references/escape-hatches.md` |
| Binding "skip approvals" to a mode a served deployment cannot enter | `references/escape-hatches.md` |
| Approval fatigue as a failure mode, and tuning the ask tier | `references/escape-hatches.md` |
| Step/time budgets as containment, and why they aren't a security control alone | `references/escape-hatches.md` |

## The idea behind it

> A capability that does not exist cannot be talked into firing.

A model that can see a tool will eventually be argued into a reason to use
it — by a user, by its own reasoning, by a crafted input the tool's output
feeds back into. A model that cannot see the tool has nothing to argue with.
Refusal-at-call-time is a real control, but it is the second line, not the
first: it depends on the refusal logic being correct on every input, forever,
including the one nobody tested. Absence depends on nothing at call time at
all.

Everything in this skill is a way of asking "should this exist for this
caller, and who decides when it fires" — before it asks "is this safe."

## Scope

This skill is about design and review, not enforcement code you install. It
teaches the shape of a safe tool surface and the failure modes of an unsafe
one, with invented examples for teaching — never a specific policy to copy in
directly. Your allowlist, your binaries, your rule order are yours to build
and adversarially test; this is the reasoning that gets you there.

## Contributing

Failure reports are the most useful contribution — a permission check that
looked right and wasn't, an escape hatch that leaked past its intended scope,
a truncated response a caller trusted as complete. Include the symptom, the
root cause, and the check that would have caught it.

## Licence

MIT.
