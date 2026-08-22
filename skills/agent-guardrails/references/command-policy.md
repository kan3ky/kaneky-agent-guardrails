# Command policy

Any tool that lets an agent run a command line is the highest-leverage
surface it has — one call away from arbitrary code, if the policy around it
is wrong. This file is the deep dive on getting that policy right. The
examples below are invented for teaching. Build your own allowlist and rule
table against your own binaries; don't copy a table from somewhere else and
assume it fits a surface it was never measured against.

## 1. argv is not a command line

A tool that runs commands usually accepts input in one of two shapes:

- **A single string**, meant to be interpreted the way a shell would —
  `"git diff --stat"`.
- **A program plus a list of arguments**, supplied separately —
  `{"program": "grep", "args": ["-r", "TODO |rm -rf", "."]}`.

These are not the same problem, and the second one is not a smaller version
of the first. If the caller supplied program and arguments separately, the
arguments are **data**, not syntax. `TODO |rm -rf` in the example above is
one argument — the literal search pattern — not a pipe into a delete. It
must never be re-joined into a string and handed back through the same
parser that handles the single-string case. That reparse is a whole
vulnerability class on its own: it turns every metacharacter a well-behaved
caller might legitimately need to search for (`|`, `;`, `$`, backticks)
into an injection point the moment a differently-behaved caller sends it.

```
# WRONG — argv reduced to a string and reparsed
cmd = program + " " + " ".join(args)
run_through_shell_grammar(cmd)

# RIGHT — argv executed directly, no grammar involved
exec(program, args)   # args[i] is always literally args[i]
```

The rule is simple to state and easy to violate by accident, usually while
adding a logging line or a "preview the command" feature that stringifies
argv for display and then, months later, someone wires that display string
back into execution because it was sitting right there.

## 2. Tokenizing safely, when you do parse a line

Some tools legitimately need to accept a single string — an agent typing a
command the way a person would at a prompt is a more natural interface than
forcing every call into structured argv. If you take that string, you now
own writing a tokenizer, and a tokenizer is where correctness gets
expensive fast.

And a tokenizer for *which shell*. This is the part that quietly sinks the
approach: there is no generic "command line" to parse. `bash`, `dash`, `zsh`,
`cmd.exe` and PowerShell disagree about quoting, escaping, variable syntax and
what counts as a separator, so a policy that tokenizes a line without knowing
which interpreter receives it is guessing. PowerShell alone breaks most
Unix-shaped assumptions — different quoting, a different escape character, its
own operators.

Even within one dialect the executable grammar is far larger than the pieces
usually checked. Newlines as separators, parameter and brace expansion, command
and process substitution, here-documents, aliases and shell functions,
arithmetic evaluation: each is a path to execution that a policy matching on
words will not see.

That leaves two honest positions and no third:

1. **Do not involve a shell.** Take structured argv and execute it directly.
   The grammar problem disappears because nothing parses anything.
2. **If a shell is unavoidable, parse with the real grammar of the exact shell
   that will run the line, and reject every node you do not positively
   recognise.** Allowlist the tree; never scan the string.

A partial tokenizer over a generic command line is the option that looks like
security and is not, because the constructs it fails to model are exactly the
ones an attacker reaches for. The checks below assume a Unix-like shell you
have pinned deliberately — they are necessary, not sufficient, and they are
not portable to another dialect.

**Quote awareness.** A space inside quotes is not a separator; an empty
quoted string (`""`) is still a token, not nothing. Losing either of these
silently changes what the command means without erroring.

**`&&` is composition; a lone `&` is backgrounding.** These look like the
same character doubled, and a tokenizer that scans one rune at a time can
misread `a && b` as `a &`, followed by a stray `& b`, if it doesn't check
the next character before deciding. Backgrounding hands control back to the
caller before the command finishes — the tool loses the ability to bound
its runtime, cap its output, or know its exit code before reporting
success. Composition with `&&`/`||`/`;` is a sequencing decision the caller
made explicitly; backgrounding is the caller keeping a process alive after
the tool stopped watching it. Treat them as different constructs, not
variations of one.

**Refuse redirection and substitution outright — don't try to analyze
them.** `>` and `<` redirect a stream to or from the filesystem, outside
whatever path-containment check the policy applies to command arguments.
`$(...)` and backticks run an inner command and splice its output back into
the line being evaluated — which means the actual command your policy
needs to classify doesn't exist until after something already ran. You
cannot decide allow/ask/deny for a command whose real shape depends on
output your policy hasn't seen yet. Parsing these "properly" means
reimplementing a shell's grammar, edge cases included, and a 90%-correct
shell grammar isn't 90% secure — it's a well-documented puzzle for whoever
finds the missing 10%. Refuse the construct at the character level, before
tokenizing, and say why in the error: a caller who wanted output redirected
to a file almost always has a structured "write this to a file" tool
available instead, and naming that path is more useful than a bare syntax
error.

## 3. The escalating flag

A binary being on the allow-list says nothing about which invocations of it
are safe. Plenty of ordinary tools have a flag that turns a read into a
write, or a query into an execution:

- A search tool with a "run this command on every match" flag turns a
  read-only search into an arbitrary-exec primitive.
- An editor or stream-processing tool with an "edit the file in place"
  flag turns "preview a transformation" into "overwrite the file" —
  without it, the same tool only ever writes to its own stdout.
- A version-control tool with a "run this as if I were in a different
  directory" flag turns "read status here" into "read (or write) status
  wherever I name."

None of these are exotic — they're standard, documented flags on
extremely common tools, which is exactly why judging the binary alone is
insufficient. The flag is what the command *is*; the binary is just what
family it belongs to.

Two things fall out of that:

- **Check for the flag wherever it can legally appear**, not just in the
  position your pattern-matching happens to look. A rule that matches
  `tool subcommand *` positionally can miss a flag smuggled in as
  `--option=value` if your check only looks for a bare `--option` token, or
  miss it entirely if it can appear before the subcommand as well as after.
  Enumerate every syntax the tool's own argument parser accepts for setting
  that flag, and check for all of them — not the one you'd type.
- **An escalating flag should promote the decision, not just get logged.**
  If the base command was `allow`, the flag's presence should push it to at
  least `ask` — the caller gets exactly the same command they asked for,
  just with a human looking at it first, because the human is now agreeing
  to something materially different from what the bare subcommand implied.

## 4. The first escaping argument

A read command is safe until its target is outside the sandbox. `cat` is
harmless — until it's `cat` against a path the workspace boundary was
supposed to keep off-limits. The binary allowlist has nothing to say about
that; only path resolution does.

Containment is a property of the **resolved** path, not of the program
name, and not of the literal string the caller typed:

- Resolve `..` before comparing against the boundary. `workspace/../secrets`
  is a string that looks contained and a path that isn't.
- Resolve symlinks before comparing. A path that starts inside the sandbox
  and follows a link to somewhere outside is still an escape; checking the
  string, not the resolved target, misses it entirely.
- **Resolve relative to the sandbox root, not the process's current working
  directory.** A tool invoked with its cwd already pointed somewhere odd
  doesn't get to redefine what "inside" means for every relative argument
  that follows — anchor at the fixed root every time, independent of where
  the process happens to be standing.
- A bare word that isn't a path at all — a search pattern, a branch name, a
  filter expression — shouldn't be run through path resolution in the first
  place; doing so produces false positives that train reviewers to ignore
  the check. Distinguish "this argument names a location" from "this
  argument is opaque data" before deciding whether containment even
  applies to it.

If the check that resolves a path and the step that actually opens or
executes it are not the same atomic operation, there is a window between
them — a symlink swapped in after the check passes but before the command
runs would defeat it. Whether that's practically exploitable depends on
your runtime, but it's worth asking explicitly rather than assuming the
check and the action are the same moment just because they're a few lines
apart in the source.

## 5. The widening problem

This is the failure mode that shows up after the policy has been live for a
while, not on day one. You allow a binary's read-only surface — a handful
of named, harmless subcommands. Someone hits a false positive: a common,
genuinely read-only subcommand you didn't enumerate gets treated as unknown
and routed to `ask` (or `deny`, if your default is stricter). The fast fix
is tempting and wrong:

```
# before — explicit, narrow
tool status *  → allow
tool log *     → allow
tool <anything else> → ask

# "fix" — widen by prefix to catch the missed subcommand
tool * → allow      # reopens every write subcommand too
```

Widening by prefix doesn't just admit the one subcommand you meant to fix —
it re-opens every other subcommand under that prefix, including the ones
the narrow rule was written specifically to keep out of `allow`. The
correct fix is almost always more rules, not a broader one: add the missed
subcommand explicitly, keep the rest narrow, and if the read-only surface
is large enough that enumerating it is unwieldy, write a wildcard scoped to
that specific read-only namespace and place it above the catch-all —
never a wildcard scoped to the whole binary.

Generalize the instinct: any time a rule is widened to fix a false
positive, the question to ask is not "does this fix the case I hit" but
"what else does this now match that it didn't before." A rule change that
can't be described that way hasn't been reviewed, only tested against one
example.

## 6. Environment scrubbing

A command run by the tool inherits process state unless something actively
strips it, and the parent process's environment routinely holds
credentials that have nothing to do with the command being run — API keys,
database passwords, session tokens set for the tool's own operation, not
for whatever the agent asked it to execute.

Two designs handle this, with a real trade-off between them:

- **Allowlist the variables that pass through** — `PATH`, `HOME`, a fixed,
  known-safe set. This is the stronger guarantee: nothing gets through that
  wasn't explicitly named. It also breaks real toolchains in practice,
  because compilers, package managers and language runtimes each read
  their own sprawling, semi-documented set of environment variables for
  caching, proxying, locale and configuration behavior — an allowlist has
  to keep pace with every tool it might be asked to run, or those tools
  quietly misbehave.
- **Inherit everything, minus anything that looks like a secret** — strip
  variables whose name matches credential-shaped patterns (contains
  `SECRET`, `TOKEN`, `KEY`, `PASSWORD`, and similar markers), pass the
  rest. This keeps the toolchain working the way the caller's own shell
  would, at the cost of being a denylist: it only catches variable names
  that match a known pattern, and a credential given an unexpected name —
  or one that doesn't happen to contain any of the marker substrings —
  passes straight through.

Neither is free. Pick one deliberately, write down why, and if you pick
inherit-minus-secrets, treat the marker list as something that needs to
stay current as new kinds of credentials get introduced — it is a living
list, not a fact you set once.

## 7. Testing a policy: enumerate the bypasses you thought of

A policy that has never been attacked by its own author is an intention,
not a guarantee. Before trusting one, sit down and try to break it, then
write each attempt down as a test that asserts the exact expected verdict —
not "is not allow," the specific decision.

Things worth trying against your own policy, adapted to whatever your
tool's actual attack surface is:

- **Reparsing tricks** — an argument crafted so that, if it were ever
  rejoined into a string and reparsed (see §1), it would smuggle a second
  command. This should be inert if argv handling is correct; the test
  proves it, rather than hoping it.
- **Encoding and lookalike tricks** — a path using URL-encoding, a
  differently-normalized Unicode form, or a case variation of a binary name
  the policy matches case-sensitively. Confirm the policy sees the same
  thing a human reading the raw bytes would see.
- **The escalating flag in every syntax the real tool accepts** —
  bundled short flags, `--opt=value`, the flag appearing before instead of
  after the subcommand.
- **The widening-by-prefix mistake**, deliberately introduced and then
  caught by a test that asserts the write subcommand is still `ask`/`deny`
  even after the read-only wildcard is added.
- **A boundary path via `..`, an absolute path, and a symlink pointing
  outside the sandbox** — three separate tests, because they exercise
  different code paths in most path-resolution implementations even though
  they express the same intent.
- **An unrecognized subcommand or argument shape**, confirming it lands in
  `ask` (or `deny`), never silently in `allow` because nothing matched.

Name each test after the bypass it closes, not after the function it
calls. A year from now, a failing test named `TestNoPrefixWideningOnVCSWrite`
tells the next reader what broke; `TestPolicy7` does not.
