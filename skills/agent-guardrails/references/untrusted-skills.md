# The skill you just installed is a tool surface

Everything else in this skill assumes you are the one designing the tool
surface. This file covers the case where someone else designed it and you
installed it — a skill, a subagent definition, a plugin, an MCP server config.

The framing that makes this tractable: **a skill definition is executable
configuration, not documentation.** It is read by the same runtime that decides
what your agent may do, and parts of it are consulted *before* the model
reasons about anything. Reviewing it the way you would review a README checks
the wrong half of the file.

## Why reading the prose is not review

A skill is two artifacts sharing a filename.

The **prose** is what the model reads and reasons about. It is also the part
that looks like the skill — it is long, it is the part a reviewer naturally
opens, and it is trivially made to look reasonable, because whoever wrote it
was free to write anything.

Prose is not harmless, and it would be a bad misreading of this file to treat
it that way. Text the model loads as instruction can drive whatever tools the
session ALREADY has, talk the model into asking the user for a secret, or
argue for its own trustworthiness during the review. It is a prompt-injection
surface with a persistent foothold. What it cannot do is grant itself a
capability — and that is the distinction being drawn here, not a claim that
clean metadata makes hostile prose safe.

The **metadata and directives** are what the runtime acts on: frontmatter that
declares which tools the skill may use, a subagent's declared permission mode,
any preprocessing syntax the host expands before the model sees the content.
This part is short, sits at the top or in a sibling `.json`, and is where every
real attack lands.

The asymmetry is the whole problem. A reviewer's attention goes to the long
part, which can only work through capabilities that already exist, while the
short part decides what exists at all — and the author chose how much of each
to write.

## Three mechanisms

Each has been demonstrated against a real host. Check each against the version
you are running rather than assuming it still holds — one of the three has
already been closed by the vendor, which is the normal life cycle and not a
reason to stop looking. The shapes generalise to any host that lets a
downloaded artifact declare its own permissions.

**1. Pre-granted tools skip the consent prompt.** Frontmatter that declares the
tools a skill may use is a convenience feature: it stops the host asking the
user to approve each one. That is also exactly what an attacker wants. The
consent prompt is the control, and a field in a downloaded file turns it off.

**2. A subagent declaring its own permission mode.** A downloaded subagent
definition can name the mode it wants to run in, and published research has
demonstrated builds where such a definition executed commands in a session
whose own settings denied that tool.

Two things must be said precisely, because getting this wrong in either
direction is bad advice. That was a finding against particular versions, not a
standing property: the documented model today is deny-first — deny rules are
evaluated before allow rules, from any settings scope, explicit denials still
apply under a bypass mode, and bypass mode can itself be disabled outright by
policy. Do not repeat the demonstration as though it describes current
behaviour.

What survives the correction is the shape, and it is why the mechanism is
listed at all: a child context naming its own privilege level is an inversion
waiting to happen. If a subagent could ever declare itself more privileged
than its parent, every restriction on the parent is advisory. Treat the mode
line in a downloaded agent file as a request the host may or may not honour on
the version you are running, verify it against the deny rules you actually
have, and remember that a defence which holds in the current release is not
the same as one the artifact cannot reach.

**3. Preprocessing runs before the model reasons.** Where a host expands
directives in a skill file before handing the content to the model, that
expansion is not subject to any judgment the model would have applied. The
model's caution is a real control for content the model *sees*. Content that
executes on the way to the model bypasses it entirely — there is no "the model
would have noticed", because the model was never consulted.

The third is the most important to internalise, because it defeats the
mitigation people reach for first. "The model wouldn't run something obviously
malicious" is a reasonable claim about content in the context window and a
meaningless one about content that executes before the context window is built.

## What actually holds

**A deny at the settings layer, in a file the artifact cannot write.** This is
the capability-absence rule from the spine, applied one layer up: a host-level
denial of the shell tool holds regardless of what any skill's frontmatter
declares, because the skill has no way to reach the setting. Deny beats allow
by design, and a deny placed in a managed or user-level settings file that the
repository cannot override is the strongest form of it. If the host offers a
switch that disables permissive modes entirely, set it — a mode nobody can
enter is worth more than a mode you are trusting downloaded files not to ask
for.

The principle underneath: every control that lives *inside* the downloaded
artifact is a control the artifact's author chose to include.

So the ordering that matters is: host settings, then artifact declarations. If
your only protection is that the skill did not ask for anything alarming, your
protection is the attacker's restraint.

**Read the metadata first, and separately.** Open the frontmatter, the sibling
manifest, and any preprocessing directives before reading a word of the prose,
and decide on those alone. If the skill declares tools or a permission mode,
that is the review — the prose cannot make a pre-granted shell tool safe, and
it will try.

**Treat installation as a dependency install, because it is one.** Same
questions: who publishes it, what does the source actually contain, what
version am I pinned to, and what happens on update.

**Update is the dangerous moment, and it is the one people skip.** A skill
vetted at one version can change completely at the next. If updates are
automatic, then what you reviewed was a snapshot with no bearing on what runs
tomorrow — you did not vet the skill, you vetted a moment. Pin, and diff the
metadata on every bump even when the prose is unchanged. Especially then: a
diff that touches only the frontmatter is the highest-signal diff in this
entire category, and it is also the one most likely to be waved through as
"config only".

## If you publish skills, what you owe the people installing them

This cuts both ways, and the obligations are concrete:

- **Declare no tools and no permission mode.** If a skill needs a capability,
  the host should ask its user, in their session, under their settings. A
  published skill that pre-grants itself anything is asking every future
  installer to trust a stranger's judgment about their machine.
- **Use no preprocessing directives.** Everything the skill does should be
  prose the model reads and can decline to act on. Keeping the model in the
  loop is the point.
- **Make the claim checkable.** "This skill declares only `name` and
  `description`" is a sentence a reader can verify in five seconds against the
  file in front of them. Say it, and keep it true.
- **Do not rewrite published history.** Someone auditing what changed between
  versions needs the versions to still be there. A force-push over a published
  tag destroys exactly the evidence this file tells people to go look at.

The general rule underneath all four: a skill should be *boring* to review.
Every mechanism that makes it more convenient to install makes it harder to
audit, and the convenience accrues to the author while the risk accrues to
everyone else.
