# Tool surface design

Command policy (see `references/command-policy.md`) governs one high-risk
tool. This file is about the surface as a whole — which tools exist for
which caller, what they return, and where the actual enforcement lives.

## 1. Register per caller identity, not globally

The easiest way to build a tool surface is to define one list of tools and
hand the same list to every caller, gating individual calls at
execution time. This is the weaker design. The stronger one registers a
different tool list per caller identity, so a caller authenticated at a
lower privilege never sees the higher-privilege tool in `tools/list` (or
whatever your protocol's discovery mechanism is called) at all.

The difference matters because of the spine principle this whole skill is
built on: a tool that's listed but refused still gives the model something
to reason about, argue for, and eventually find an edge case in. A tool
that was never registered gives it nothing. If a read-only credential is
handed to an agent, build the read-only tool list for that credential —
don't build the full list and rely on every mutating tool's own check to
turn the caller away every time. The refusal-at-call-time path should be a
backstop for a mistake in registration, not the primary mechanism.

## 2. Separate credentials per privilege tier

Once tools are split by privilege tier, the credentials backing them should
be too. A read tool and a write tool should never authenticate with the
same token, for two independent reasons:

- **Leak radius.** If the read credential leaks — logged somewhere it
  shouldn't be, echoed in a debug trace, cached in a place with looser
  access controls than the secret store — the blast radius is "read
  access," not "read and write." A shared credential makes every leak a
  worst-case leak, regardless of which tool actually caused it.
- **Revocation.** Rotating or revoking the write credential because a
  mutating operation misbehaved shouldn't take read access down with it.
  Separate credentials mean separate blast radius in both directions —
  compromise and remediation.

Practically, this usually means the tool server itself holds two
credentials (or more, for more tiers) and decides which mutating tools to
register based on which of them it was actually given — not one credential
with an internal flag saying "and also allowed to write." If the
write-tier credential is absent, the mutating tools shouldn't be
constructed at all, following directly from §1.

## 3. Response caps that announce themselves

Any tool that returns a list — findings, log lines, search results — will
eventually be asked for more than fits in one response. The cap itself is
not the interesting design decision; what happens at the cap is.

A response that silently truncates is a response that lies by omission: a
caller who doesn't know the list was cut short reasons from it as if it
were complete. For a security-relevant tool specifically — "list every
finding," "show every rule that matched" — a truncated list that reads as
exhaustive is close to the worst failure that class of tool can have,
because the caller's entire conclusion ("nothing here is critical," "no
rule blocked this") is built on data it doesn't know is partial.

The fix is cheap and has no excuse not to be universal: every list-shaped
response carries its own honesty alongside the data —

```
{
  "total_matched": 812,
  "returned": 50,
  "truncated": true
}
```

— and the tool's own description tells the caller how to narrow instead of
how to raise a limit that doesn't exist ("narrow with `severity`, don't ask
for a bigger page" beats a `limit` parameter with no ceiling explained).
Raising the limit does not scale; narrowing the query does, and a caller —
human or model — needs to be told which one is the actual lever.

## 4. Tool descriptions are prompt surface

A tool's description string is not documentation a human reads once during
setup. It's read by the model on every call it considers, and it functions
as an instruction, not a caption. Two consequences follow that get
underused in practice:

- **Say what the tool must not be used for**, not only what it's for. A
  description that states only the happy path invites use in situations
  the tool was never designed to handle safely — "start a scan" without
  "refused server-side if the target is out of scope" reads, to a model
  deciding whether to call it, as unconditional.
- **Warn against polling loops on anything slow or asynchronous.** A tool
  that kicks off work taking minutes and returns an id needs its
  description to say "check back later," explicitly, or a model will
  reach for the natural next move — call the status tool again immediately,
  then again, then again — turning one legitimate action into a
  self-inflicted load problem. This is cheap to prevent and expensive to
  notice after the fact, because "the model is polling" doesn't look like
  a bug from the model's side; every individual call is a reasonable
  status check.

Write descriptions the way you'd write a warning label, not a docstring:
assume the reader will act on the sentence, not just skim it as
metadata.

## 5. Server-side enforcement

Wherever the actual gate lives — the allow/ask/deny decision, the
scope check, the credential-to-tool-list mapping — it has to live behind
the API the tool wrapper calls, never solely inside the wrapper or the
client that's presenting the tool to the model.

The test: imagine the wrapper deleted, and the same caller hitting the
underlying API directly with the same credential. If that's *more*
permissive than going through the wrapper, the enforcement was in the
wrong place — the wrapper was providing the appearance of a boundary, not
the boundary itself. A client can be rewritten, a compromised host can
skip calling the wrapper's checks, a well-meaning integration can talk to
the API directly because the wrapper was inconvenient. None of those
should be able to reach anything the wrapper's checks were the only thing
preventing. The API is the trust boundary; the tool wrapper is a
convenience layered on top of it, and should be treated as one when
deciding where a check has to live.

## 6. Non-hot-reloadable scope

Whatever defines the boundary of what a tool surface can reach — the
allowlist of scannable targets, the set of registered mutating tools, the
credential-to-privilege mapping — should require a reviewed change to
widen: a commit, a build, a deploy. Not an environment variable a running
process re-reads, not a runtime flag flipped without restarting anything.

The reasoning is about what a single compromise can do. If scope widening
requires a code change and a deploy, a compromised runtime — the agent
itself, a process it spawned, an attacker who gained one write somewhere —
cannot widen its own reach; the worst it can do is whatever the current,
already-reviewed scope allows. If scope widening is a config value the live
process reads, that same compromise is one write away from becoming a
bigger compromise. Making the boundary require review is what keeps
"what can this agent reach" answerable by reading the repository, rather
than by also checking whatever the running environment happens to contain
right now.
