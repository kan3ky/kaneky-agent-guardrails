---
name: agent-guardrails
description: Design and review agent tool surfaces that fail safe — what capabilities a caller can even see, the allow/ask/deny gate around each one, command-execution policy, and the escape hatches every guardrail eventually grows. Use when registering a new tool for an agent (CLI, MCP server, function-calling API), when writing or reviewing a command allowlist/policy, when deciding what needs human approval, or when an agent's tool surface is being extended and someone should check what that surface can now reach.
---

# Agent tool-surface guardrails

The strongest control on an agent is not a better refusal. It is a tool that
was never registered for this caller.

> A capability that does not exist cannot be talked into firing.

A model that can see a tool will eventually be argued into a reason to use
it — by a user pushing on it, by its own multi-step reasoning finding a path
that technically satisfies the tool's stated purpose, by content the tool
itself returns that the model then acts on. Every one of those arguments
requires the tool to be visible. Refusal-at-call-time — "you may call this,
but not with these arguments, not right now" — is real and necessary, but it
is the second line of defense, and it only holds if the refusal logic is
correct on every input, including the one nobody tested. Absence does not
depend on the input at all.

This reframes the design question. The instinct is to ask "is this tool
call safe?" The better question is "does this caller need this tool to
exist at all, and if it does, who decides when it's allowed to fire?"

## Three states, not a boolean

A tool gate that only allows or denies pushes every ambiguous case toward
one of two bad defaults: deny-by-default makes the agent useless for
anything it hasn't been explicitly cleared for, and allow-by-default makes
"is this safe?" a question you have to get right on every call, forever.

A third state — ask — is what makes a permissive-enough-to-be-useful
surface survivable:

- **allow** — runs immediately. Reserved for actions whose worst outcome is
  acceptable unattended: reading a file, listing state, running a
  build in a sandboxed workspace.
- **ask** — a human sees it before it runs. The default for anything that
  writes, spends money, leaves the machine, or composes several allowed
  actions into one you never separately evaluated.
- **deny** — refused outright, and no approval in the moment can override
  it. Reserved for things outside the tool's contract entirely — a path
  outside the sandbox, a binary that was never meant to be reachable here.

The question worth asking about any single action is not "is this safe" —
plenty of individually-safe-looking actions are dangerous in combination,
and plenty of nominally-risky ones are fine in context. The question is
"who decides." allow says the tool decides. ask says a human decides, per
call. deny says the design already decided, permanently. Most of the value
in a good policy is putting the ambiguous middle into ask by default — an
unrecognized shape of a known-dangerous action should route to a human, not
silently fall through to allow because nothing explicitly caught it.

## Ordered, first-match rules — order is a security property

A tool surface is usually governed by more than one rule, and the natural
way to write that is a list evaluated top to bottom, first match wins. That
ordering is not incidental plumbing. It is where the actual security
decision lives.

Put the narrow, dangerous shape of an action above the broad, permissive
one, and the dangerous shape wins when both would otherwise match:

```
rule 1: <tool>, subcommand "delete-everything", any args  → ask
rule 2: <tool>, any subcommand, any args                  → allow
```

Reverse those two lines and rule 2 now matches first for every call,
including the delete — the ask never fires, silently, because the broader
rule sits above the narrower one. This is the single most common way a
correct-looking policy stops doing what its author believes it does: every
rule reads right in isolation, and the bug is only visible by tracing one
call through the whole ordered list.

Two consequences follow:

- **Reviewing a policy means reading it top to bottom for a specific call**,
  not just checking that a rule for that call exists somewhere in the file.
- **Adding a new broad rule is a change to every narrower rule below it**,
  because a broad match placed above them shadows them. Add new specific
  rules near the top, and treat inserting anything above existing rules as
  a change that needs the same scrutiny as the rule itself.

## What to review in any agent tool surface

Work through this whenever a tool is added, or a surface is being handed to
a new kind of caller:

1. **Enumerate what's registered for this caller, not what's possible.**
   Two callers hitting the same backend can have different tool lists; the
   list a caller can see is the actual boundary, not the code path that
   would technically permit a call if it were offered.
2. **For every tool, name the worst single call it allows** — not the
   intended use, the worst-case argument a caller could legally pass.
3. **Confirm decisions are allow/ask/deny, not a boolean**, and that
   unrecognized shapes of dangerous actions land in ask, not allow.
4. **Trace a handful of concrete calls through the ordered rules by hand.**
   If you can't predict the verdict without running the code, neither can
   whoever adds the next rule.
5. **Confirm enforcement is server-side** — see
   `references/tool-surface-design.md` for why a client-side or
   wrapper-side check is not a boundary.
6. **Confirm any escape hatch (an admin override, a "skip approval" mode)
   cannot exist in the deployment shape a remote caller reaches** — see
   `references/escape-hatches.md`.
7. **If the surface includes running commands**, check it against
   `references/command-policy.md` specifically — command execution is
   where most of the subtle bypasses live.
8. **State what you didn't check.** A review that claims completeness it
   doesn't have is worse than one that names its gaps.

## References

- `references/command-policy.md` — the deepest file. argv vs. a command
  line, safe tokenizing, the escalating flag, the first escaping argument,
  the widening problem, environment scrubbing, and testing a policy
  adversarially.
- `references/tool-surface-design.md` — registering tools per caller
  identity, separate credentials per privilege tier, response caps that
  announce truncation, tool descriptions as prompt surface, server-side
  enforcement, non-hot-reloadable scope.
- `references/escape-hatches.md` — every guardrail grows a bypass; design
  it on purpose. Binding it to a deployment shape, making it observable,
  approval fatigue, and step/time budgets as containment.
