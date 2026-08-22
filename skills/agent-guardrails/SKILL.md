---
name: agent-guardrails
description: Design and review agent tool surfaces that fail safe — what capabilities a caller can even see, the allow/ask/deny gate around each one, command-execution policy, giving an agent an identity so its actions stay attributable, and the escape hatches every guardrail eventually grows. Also covers the surface you did NOT design: vetting a downloaded skill, subagent or plugin before installing it, since those declare their own tool permissions and are read before the model reasons. Use when registering a tool for an agent (CLI, MCP server, function-calling API), writing or reviewing a command allowlist, deciding what needs human approval, installing or updating a third-party skill or agent definition, or extending an agent's tool surface.
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

## Register per caller, not globally

The spine's claim is only worth anything if a tool's existence can differ by
caller. If every tool is registered once at startup for everyone, "absent" is
not available as a control and you are left with refusal alone.

The practical shape is a read identity and a write identity that are separate
credentials, with the mutating tools registered only when the write credential
is present. An agent running autonomously gets the read credential and never
sees the mutating tool in its tool list — there is nothing to reason about, and
nothing for injected content to name. The same binary, given the write
credential in an operator-driven session, offers the full surface.

Two properties make this worth the wiring:

- **The tool list is itself prompt surface.** A model that can enumerate a
  destructive tool will reason about it, mention it to the user, and try it
  when a task seems to call for it. Removing it removes that entire branch.
- **Credentials should not be shared across privilege levels.** If the read
  path and the write path carry the same token, then a leak of the read
  credential is a leak of write access, and the split you designed exists only
  in the code, not in the blast radius.

**And enforce server-side.** The gate belongs behind the API, not in the tool
wrapper. A wrapper is client code: it can be rewritten, bypassed, or simply
called differently. If the only thing preventing an out-of-scope action is the
tool definition, then the control lives in the least trustworthy place in the
system.

## An agent with no identity of its own leaves no attributable record

Registering per caller solves what an agent *can* do. It does not answer who
did it, and those are separate failures with separate consequences.

The shortcut is to let an agent act as its operator — same credential, same
subject, same rows in the log. It works immediately, and it destroys
attribution permanently: every action the agent takes is indistinguishable from
an action the human took, in the audit trail, in the access logs, in whatever
record someone eventually has to reconstruct an incident from. "Did she run
this, or did the thing she delegated to run it?" has no answer, and it is the
first question anyone asks.

Give the agent its own identity, issued by the principal, carrying a **subset**
of what the principal holds. Two properties follow, and both matter:

- **The subset relationship is what makes it a delegation.** An agent granted
  everything its principal holds is not acting on their behalf — it *is* them,
  and there is nothing left to distinguish. It must not be possible to grant an
  agent more than its issuer has.
- **A distinct identity can be revoked alone.** Revoking a compromised agent
  should not require locking out the person it worked for, and today that is
  frequently the only available response, which means the response gets
  delayed.

The same reasoning applies one level down, to how the grants themselves are
expressed. A single admin-or-not role cannot say "may read the issue tracker
but not open a merge request", so it gets granted to anyone who needs any part
of the surface, and it stops meaning anything. Named capabilities, granted as
data rather than compiled in, let one subsystem go to one caller — and let a
grant be withdrawn as an operation rather than a deployment.

One trap worth naming, because it is the usual way this goes wrong: a
permission field that nothing reads is worse than no field at all. It answers
"is this covered?" with a yes, it passes review, and its own tests pass — they
verify the field is set, not that anything consults it. Confirm the enforcement
path reads it, on the code path callers actually take. An unenforced gate and a
fully-authorised caller produce identical output and identical logs; there is
no observation that separates them, which is why this kind of gap survives for
years.

## A truncated result must say it was truncated

Tool results feed straight back into a model's reasoning, and a result that was
cut off silently becomes a false premise the model then builds on.

Cap what a tool returns — an unbounded result can exhaust the context, and one
large response can displace everything the model needed to remember. But a cap
without a signal is worse than no cap: a list of ten findings, where the tool
quietly dropped ninety, reads as "there are ten findings". The model reports
that to the user as a complete answer, confidently, and nobody can tell.

Return three things, always: the total that matched, the number returned, and
an explicit truncation flag. Then say in the tool's description that the
consumer must check the flag before drawing a conclusion. This is the same
failure the memory and extraction skills describe from other angles — absence
and limitation sharing a representation with completeness.

**Prefer narrowing to raising.** A tool that accepts a filter lets a caller ask
a smaller question rather than a bigger one, which is cheaper and produces a
complete answer to something rather than a partial answer to everything.

## Tool descriptions are instructions, not documentation

Whatever a tool's description says goes into the model's context and shapes
behaviour. It is not a docstring for humans who might grep it later.

- **Say what the tool must not be used for.** A description that only states
  the happy path invites every adjacent use.
- **Warn against expensive patterns explicitly** — polling a slow operation in
  a loop, calling a tool per item where a batch parameter exists. A model will
  do the naive thing unless told.
- **Describe the failure modes the caller must handle**, such as the truncation
  flag above, or a result that is a refusal rather than an error.

Treat any text that reaches the model as part of the guardrail, because it is.

## Content returned by a tool is untrusted input

A tool that fetches a page, reads a file, or queries a system returns content
that some other party may control. That content arrives in the model's context
in the same channel as instructions.

This is the delayed-injection path, and it defeats a surface designed only
against a hostile *user*: the user is benign, the fetched document is not.
Treat everything a tool returns as data — label it as such in the prompt, never
let it be interpreted as a directive, and be especially careful with a tool
whose output routinely contains instructions, such as one that reads issue
trackers, emails or code comments.

The capability-absence rule is the strongest defence here too. Injected content
can only name tools that exist for the current caller.

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
8. **If any part of this surface came from a downloaded skill, subagent or
   plugin definition, review its metadata before its prose** — see
   `references/untrusted-skills.md`. The artifact declares its own
   permissions, and that declaration is consulted before the model reasons
   about anything.
9. **State what you didn't check.** A review that claims completeness it
   doesn't have is worse than one that names its gaps.

## Adjacent skills

- **agent-memory** — when the concern is what an agent *stores* rather than
  what it can do. The truncation and absence problems above appear there in
  another form.
- **auth** — when the caller is a person or service rather than a model, and
  the question is object-level authorisation rather than tool registration.
- **delegation** — when the untrusted party is another agent producing work you
  must verify, rather than a tool surface you must scope.

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
- `references/untrusted-skills.md` — the surface you did not design: skills,
  subagents and plugins as executable configuration. Pre-granted tools that
  skip consent, a subagent declaring itself more privileged than its parent,
  preprocessing that runs before the model reasons, why update is the
  dangerous moment, and what you owe people if you publish skills yourself.
