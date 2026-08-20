# Escape hatches

Every guardrail eventually grows a bypass. Someone needs to move faster
than the approval queue allows, someone needs a support path for when the
normal flow is broken, someone needs local development to not require
clicking through ten prompts to run a build. The bypass is not the mistake.
An undesigned bypass is.

## 1. Design it deliberately, or it appears by accident

If the design doesn't name an intended escape hatch, one gets added anyway
— usually under deadline pressure, usually as the smallest possible diff,
usually by someone who reasonably believes their specific case is safe. A
debug flag that skips a check "just for this investigation." A
trusted-caller exception added for one integration and never revisited. A
conditional that reads `if env == "dev"` in code that later runs somewhere
the author didn't anticipate.

The fix is not to prevent every bypass from ever existing — some are
genuinely necessary — it's to make the bypass a first-class, reviewed part
of the design, with the same scrutiny as the guardrail it bypasses. Ask,
while building the guardrail: what's the legitimate reason someone would
need to get around this, and build that path on purpose, narrowly, instead
of leaving it to be improvised later under worse conditions.

## 2. Bind it to a mode a served deployment cannot enter

The single most common legitimate escape hatch is "skip approvals" —
stop asking before every action, for a fully autonomous run. It is
genuinely reasonable in one specific context and genuinely catastrophic in
another, and the difference is not the flag, it's who else is exposed.

**On a machine one person owns, running locally**, a skip-approvals mode
removes friction for someone who already has full access to everything the
agent could touch — they could run the same commands themselves, by hand,
with no approval queue involved. The guardrail was protecting them from
their own mistakes and interruption fatigue, not from another party.

**On a deployment other people can reach** — hosted, multi-tenant, remote —
the approval gate is very often the *only* thing standing between an
authenticated but otherwise unprivileged caller and code execution or a
mutating action. A setting that turns that off is not a convenience
feature in that context. It's a remote-privilege-escalation switch wearing
a friendly name, and it is exactly as dangerous as whatever the approval
gate was created to prevent in the first place.

The fix is to bind the hatch to a property of the deployment itself, not
to a value the deployment can be told at runtime. A flag or environment
variable is something both deployment shapes can read — nothing stops
someone from setting it on the hosted instance by mistake, or a future
change from exposing a way to set it remotely. A property fixed at build
time — "this binary was built as the single-user local artifact" versus
"this binary was built as the served instance" — removes the code path
entirely from the shape that doesn't get to have it, rather than defaulting
it to off in a shape that technically could. This is the same absence-over-
refusal principle the whole skill starts from, applied to the guardrail's
own bypass: don't build a hatch that has to correctly refuse to open on the
served instance — build a served instance that has no hatch to open.

## 3. Make it observable

An escape hatch that's silent is worse than one that's loud, because the
whole point of a guardrail is that someone downstream — a reviewer, an
incident responder, a future version of the same operator — can trust that
its absence-of-friction means absence-of-risk. If the hatch can be open
without anything showing that, that trust is now wrong in a way nobody can
detect.

- **Surface it persistently in the interface**, not as a setting you'd
  only see by opening the exact menu that controls it. A banner, a status
  indicator, something visible during ordinary use — not something you'd
  only notice if you already suspected it was on.
- **Log every action taken while the hatch is open**, so it's still
  attributable after the fact. The hatch removes the human-in-the-loop
  check; it should never also remove the record of what happened without
  one.
- **Think carefully before letting it persist silently across a restart.**
  Persisting the choice is a legitimate convenience for someone who
  deliberately wants a long-running unattended session and doesn't want to
  re-enable it after every restart. The failure mode isn't persistence
  itself — it's persistence that doesn't announce itself. If the setting
  survives a restart, the restart is the moment to say so loudly, every
  time, not just at the moment someone first flipped it on. A hatch that's
  been open for three weeks because nobody remembered it was on is the
  scenario this guards against.

## 4. Approval fatigue is a real failure mode

A gate that fires too often gets rubber-stamped. This isn't a hypothetical
about lazy operators — it's a predictable consequence of asking a human to
make the same low-stakes decision repeatedly: attention calibrates to the
base rate. If forty consecutive approval prompts were all fine to approve,
the forty-first gets approved with the same speed and the same lack of
scrutiny, whether or not it's actually the one that mattered. A gate that
fires constantly and gets auto-approved by someone who's stopped reading it
provides less real oversight than a narrower gate that fires rarely and
gets genuinely read every time — the second one still functions; the first
one only looks like it does.

This makes the calibration of the ask tier itself a design problem, not a
one-time decision:

- **Too broad**, and everything routine interrupts — routine, safe,
  boring actions land in `ask` right alongside the genuinely risky ones,
  training the operator that approval prompts are noise to clear rather
  than decisions to make.
- **Too narrow**, and dangerous actions get moved into `allow` specifically
  to reduce the interruption count — the fatigue problem gets "solved" by
  quietly shrinking what asks in the first place, which is worse than the
  fatigue it was trying to fix.

Treat the ask tier as something to measure, not just define once: how
often does it fire, and on what. If the overwhelming majority of prompts
get approved instantly without hesitation, that's a signal the boundary is
miscalibrated — not evidence the operator is being careless. Move the
boundary; don't blame the reader.

## 5. Step and time budgets as containment, not a control on their own

An agent operating autonomously — planning and acting across multiple steps
without a human in every cycle — needs some bound on how long or how far
it can run unsupervised, independent of what any single action within that
run was authorized to do. A budget on step count, wall-clock time, or both
limits how much damage an agent that's gone wrong in a way nobody
anticipated can do before something notices and stops it.

But a budget bounds *duration*, not *capability*. It says nothing about
what's possible within the budget — if the tool surface underneath is
unscoped, a generous step budget just delays the bad outcome by letting
more of it happen per unit of unsupervised time, rather than preventing it.
A budget layered on top of a properly scoped, allow/ask/deny-gated tool
surface is real containment-in-depth: even a fully compromised agent inside
that budget can only do what the surface underneath permits. A budget
layered on top of an unscoped surface is not a security control at all —
it's a clock on an outcome that was already going to happen.

The budget should not be read as pre-authorization for everything inside
it, either. An autonomous loop running within its step budget should still
hit the same `ask` gates a single interactive call would — the budget
controls how long the loop is allowed to keep going without a human
checking in on the loop as a whole, not whether individual actions inside
it still need one.
