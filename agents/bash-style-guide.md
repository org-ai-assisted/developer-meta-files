# Bash Style Guide (AI-Assisted)

Bash style for assisted-by-ai org scripts. Rules are flat-numbered
(R-NNN) so they can be cited in code review, commit messages, and
PR replies. Project-specific rules for the github-org-* / dm-*
surface live in [github-org-tools.md](github-org-tools.md) under
G-NNN.

Each rule: a one-line statement (the bold first sentence), an
optional "Why" rationale, an optional code example. Skim the bold
lines to audit a diff; read the Why for context. Rules at the top
of a section are hard; rules at the bottom are softer preferences.
Rules that cite a helper script include the source path so a reader
can confirm intent against implementation.

Each rule is tagged `_auto-detected: yes|no | auto-fixed: yes|no_`:
whether the `dist-ai-style` gate mechanically flags it, and whether it
also fixes it in place. A `no` on detect is guidance the gate does not
check -- apply it yourself; `auto-fixed: yes` means the gate corrects it
for you, so you rarely hand-write it.


## File-level

**R-001: ASCII only.** Source code and commit messages are ASCII
only. No smart quotes, em dashes, zero-width spaces, emoji.
_auto-detected: yes | auto-fixed: yes_

Why: AI tools reflexively render text with cosmetic unicode (U+2014 em
dash, U+2192 right-arrow); strip them. ASCII-only files make
`LC_ALL=C grep -PlI '[^\x00-\x7F]'` a useful pre-push gate. The
runnable gate `pre-push-static` (ships in dist-ai)
applies that grep to both changed files and the commit-range message;
install it as `.git/hooks/pre-push` to make R-001 violations
impossible to push.

**R-002: File header includes the 'AI-Assisted' marker.** Every
new file from scratch carries the standard 5-line header:
_auto-detected: yes | auto-fixed: no_

    #!/bin/bash

    ## Copyright (C) 2026 - 2026 ENCRYPTED SUPPORT LLC <adrelanos@whonix.org>
    ## See the file COPYING for copying conditions.

    ## AI-Assisted

    ## <one-line description of what the script does>

Why: marks AI involvement; satisfies the project's attribution
policy.


## Shell options

**R-010: Set the strict-mode block at the top of every script.**
_auto-detected: yes | auto-fixed: no_

    set -o errexit
    set -o nounset
    set -o pipefail
    set -o errtrace
    shopt -s inherit_errexit
    shopt -s shift_verbose
    export LC_ALL=C

Why: `errexit` aborts on first uncaught failure. `nounset` catches
unset-variable typos. `pipefail` makes a pipeline's exit code the
last (rightmost) non-zero status, so a failure anywhere in the pipe
is not masked by a later command's success.
`errtrace` makes ERR traps inherit into shell functions.
`inherit_errexit` makes `$()` subshells respect errexit (bash >= 4.4).
`shift_verbose` logs when `shift` runs past argv end.
`export LC_ALL=C` pins the locale so `sort`, `[a-z]` ranges, `printf`
number formatting, and tool error-message text are deterministic
regardless of the caller's environment - the reproducible-build
default. Unlike the `set -o`/`shopt` lines it is an `export`, so it
propagates into child processes AND, if the script is sourced, into
the sourcing shell; it therefore obeys the same guard/library
placement rules as the rest of the block (see R-010a). A command that
must handle multibyte data prefixes its own override, e.g.
`LC_ALL=C.UTF-8 sort`, rather than dropping the block default. The C
locale also makes bash's OWN string operations byte-oriented: `${#var}`
counts bytes, `${var:off:len}` slices bytes (so a `${body:0:512}` log
truncation can split a UTF-8 sequence), and `${var^^}`/`${var,,}`,
`[[ =~ ]]`, and `[[:class:]]` match by byte. A per-command prefix cannot
reach an expansion the PARENT shell evaluates, so a script that must
measure or slice multibyte data in the shell scopes a `LC_ALL=C.UTF-8`
block around that code (e.g. a subshell) instead.

**R-010b: Do not declare a versioned `bash` dependency (`bash (>= 4.4)`
or similar) in `debian/control` for the strict block.** Supported Debian
(trixie and later) ships bash >= 4.4, and earlier Debian and bash
versions are unsupported, so `inherit_errexit` and the rest of the
preamble are always available. The dependency adds nothing; leave it out,
with no explanatory comment.

**R-010a: A script a dist-ai test sources shall be source-able:
`main()` holds the logic, and both the strict-mode block and the
`main` call are guarded by `was_executed`.**

A test that drives a script's functions must be able to `source` it
without running it or leaking strict-mode into the test shell. Such a
script sources `check_runtime.bsh`, keeps its strict-mode block and its
`main "$@"` call each behind `if was_executed "${BASH_SOURCE[0]}"`, and
moves its former top-level logic into `main()`:

    source /usr/libexec/helper-scripts/check_runtime.bsh

    if was_executed "${BASH_SOURCE[0]}"; then
       set -o errexit
       set -o nounset
       set -o pipefail
       set -o errtrace
       shopt -s inherit_errexit
       shopt -s shift_verbose
       export LC_ALL=C
    fi

    main() {
       ## former top-level logic; globals may stay global,
       ## function definitions stay at top level
    }

    if was_executed "${BASH_SOURCE[0]}"; then
       main "$@"
    fi

Scope: a NEW script shall use this form when it has, or is about to
receive, a dist-ai test that sources it. Existing untested scripts are
future work - do not churn them into this form without a test reason. A
pure sourced LIBRARY (only ever sourced, never executed - e.g.
tb-updater `version-validator`) does NOT use the guard: it defines
functions only and keeps zero top-level strict-mode (which would leak
into the sourcing shell).

Why: the gate (R-010) already recognises the guarded form - zero
column-0 strict directives plus a `was_executed`/`was_sourced` call in
command position exempts the script from the top-level all-seven
requirement, because enabling strict-mode at top level would leak into
any sourcing script.

The `shopt` half of the guarded block is the one part the gate DOES
enforce, and GATE-ENFORCED it is: authors reflexively copy the
`set -o errexit`/`nounset`/`pipefail`/`errtrace` lines but drop
`shopt -s inherit_errexit` and `shopt -s shift_verbose`, and nothing
used to check the indented block. So when a guarded block enables
`errexit`, the gate requires both `shopt` lines inside the guard (an
`errexit` guard without `inherit_errexit` leaves `$()` subshells not
respecting errexit - the very leak the strict block exists to close).
It also requires `export LC_ALL=C` inside that same guard: as an
`export` it is the MOST leak-prone line in the block - an unguarded
top-level `export LC_ALL=C` in a source-able script silently forces
the C locale onto the sourcing shell and everything it runs
afterwards, changing collation and multibyte handling with no error.
The `set -o` choice itself stays the script's own: a script may
deliberately defer `pipefail` (`live-mode.sh`, `get_writable_fs_lists.sh`)
or omit `nounset` (`onion-time-pre-script`, which carries a
`## style-ok: no-strict` waiver), and the gate does not second-guess it.

"Is it dist-ai tested" is not gate-knowable, so the gate cannot enforce
the rest of R-010a directly; that remains a manual pre-push item (see
[`pre-push-checklist.md`](pre-push-checklist.md)).


**R-011: Don't toggle errexit around a command to capture its rc.**
Use `||`-suffixed assignment.
_auto-detected: yes | auto-fixed: no_

Bad:

    set +o errexit
    out="$(cmd)"
    rc=$?
    set -o errexit

Good:

    rc=0
    out="$(cmd)" || rc=$?

Why: shorter, errexit-on-by-default never lapses, `inherit_errexit`-
safe.


**R-012: Arithmetic assignment uses `var=$((expr))`, never
`(( expr ))`.** Under `errexit`, an arithmetic expression that
evaluates to zero exits the shell.
_auto-detected: no | auto-fixed: no_

Bad:

    (( count += 1 ))            # if count was 0, now (( 1 )) -> ok
    (( found = 0 ))             # exits the script (rc=1)

Good:

    count=$((count + 1))
    found=0

Why: `(( expr ))` returns rc=1 when `expr` evaluates to 0 (POSIX
arithmetic-expression semantics), which `errexit` interprets as a
command failure. `var=$((expr))` is an assignment: for a well-formed
expression its rc is the assignment's (0), not the computed value. A
genuine evaluation error (division by zero, a malformed expression)
still fails and, under `errexit`, still aborts - the fix is about the
value-zero case, not a claim that arithmetic can never fail.

**R-013: Set shell options by long `-o` name, one per line -- even in
POSIX `sh`.** `set -eu` -> `set -o errexit` / `set -o nounset`.
_auto-detected: yes | auto-fixed: no_

    Bad:  set -eu
          set -euo pipefail
          set -o errexit -o nounset

    Good: set -o errexit
          set -o nounset
          set -o pipefail

Why: `dash` and `busybox sh` both VALIDATE and IMPLEMENT `set -o errexit`
/ `set -o nounset` / `set -o pipefail` (verified behaviourally, not just
accepted syntax), so the long form is portable to `#!/bin/sh`, not
bash-only. Long names self-document, and one option per line makes a
diff that adds or drops a single option reviewable. GATE-ENFORCED: a
short-flag enable (`set -e`, `set -eu`, `set -euo pipefail`) OR more than
one option on a single `set` line fails the gate. `set --` / `set --
"$@"` (positional parameters) and a lone `set -o <name>` are fine; the
`set +o <name>` toggle is R-011's concern.


**R-014: `errexit` is disabled inside any command that is the operand
of `||`, `&&`, or an `if`/`while` condition -- and an inner `set -o
errexit` does NOT re-arm it there.** So a failing command in a guarded
group runs past its failure and can yield a false success.
_auto-detected: no | auto-fixed: no_

Bad:

    (
       set -o errexit
       run_scenario
    ) || ec=$?                                   # errexit DEAD in the subshell

Good:

    set +o errexit
    (
       set -o errexit
       run_scenario
    )                                            # standalone -> inner -e is live
    ec=$?
    set -o errexit

Why: run the group STANDALONE (not as a `||`/`if` operand) with the
outer errexit briefly off, then capture `$?`. A failed step then aborts
the group instead of falling through to later commands (e.g. passing
assertions after a failed setup -> a false PASS in a test runner).


**R-015: A `while COND; do ...; done` loop's exit status is the LAST
command run in the body (or 0 if the body never ran) -- never the
EOF-terminating `read`.** So a stream-scanning value helper reports
success on no-match.
_auto-detected: no | auto-fixed: no_

    apt_candidate() {
       while IFS= read -r line; do
          case "${line}" in *Candidate:*) printf '%s' "..."; return 0 ;; esac
       done < <(apt-cache policy -- "$1")
    }

Why: on no match the last body command is the non-matching `case` (rc 0)
and an empty stream never runs the body (also 0); the failed EOF `read`
does not set the status. So `cand="$(apt_candidate x)"` under `errexit`
does NOT abort -- `cand` is empty and the script continues. Guard the
empty result explicitly; a 6-line repro settles any doubt (reviewers
routinely claim the opposite).


**R-016: Prefer running a command and CAPTURING its output before a loop
over feeding the command into the loop through process substitution
(`done < <(cmd)`) or a complex subshell.** Run the command on its own
line, check its status, then iterate the captured value.
_auto-detected: no | auto-fixed: no_

Bad:

    while IFS= read -r line; do
       ...
    done < <(apt-cache policy -- "$1")

Good:

    policy="$(apt-cache policy -- "$1")" || return
    while IFS= read -r line; do
       ...
    done <<< "${policy}"

Why: `< <(cmd)` runs the command in a subshell whose exit status the loop
never sees, so a failed producer reads as an empty stream (see R-015);
capturing first makes the failure catchable (`|| return`) and the flow
linear and readable. Reserve process substitution for a stream too large
to hold in memory, or one that MUST feed the CURRENT shell (a pipe would
run the loop in a subshell and lose its variable writes) -- and then
guard the producer's failure explicitly. Applies on TOUCH, like other
structural debt -- do not mass-rewrite pre-existing `< <(...)` unprompted.


## Variables

**R-020: Wrap every variable reference in `${var}` braces.** No
bare `$var`.
_auto-detected: no | auto-fixed: no_

Why: removes shell-parser ambiguity at concatenation boundaries
(`${prefix}foo` vs `$prefixfoo`); makes refactor regex-greppable;
matches what shellcheck would flag in pedantic mode.

**R-021: Declare locals at the top of the function, blank line,
then assignments.**
_auto-detected: no | auto-fixed: no_

    foo() {
       local repo url current_branch

       repo="$1"
       url="$(remote_url "${repo}")"
    }

Why: separates declaration from assignment; one place to audit
"what state does this function have"; matches the codebase norm.

**R-022: Don't combine `local` with command-substitution
assignment.** `local x="$(cmd)"` masks the substitution's exit
status (the `local` builtin returns 0 even when `$(cmd)` failed),
so errexit cannot fire. Split into two statements.
_auto-detected: no | auto-fixed: no_

Bad: `local out="$(cmd)"`

Good:

    local out
    out="$(cmd)"

**R-023: Variable names are descriptive.** No single-letter (`e`,
`x`, `t`) or cryptic abbreviations (`tmpfn`, `cfg2`).
_auto-detected: no | auto-fixed: no_

**R-024: Variable names in error messages are wrapped in single
quotes.** `log error "couldn't read '${path}'"`.
_auto-detected: no | auto-fixed: no_

Why: single quotes make trailing/leading whitespace in the
expanded value visible (otherwise lost in line-break artifacts).

**R-025: Arrays touched under `nounset` must be `arr=()`-
initialized before any access.**
_auto-detected: no | auto-fixed: no_

Why: `${#arr[@]}` and `"${arr[@]}"` raise `arr: unbound variable`
when the array has never been assigned. The first `arr+=(item)`
auto-creates, but that does not help paths where no items are
appended (e.g., a parser that sees no positional args).

**R-026: No obsolete empty-array guard `${arr[@]+"${arr[@]}"}`.**
Expand a `arr=()`-initialized array plainly: `"${arr[@]}"`.
GATE-ENFORCED.
_auto-detected: yes | auto-fixed: no_

Why: the `+alternate` operator applied directly to `[@]` only
existed to work around bash BEFORE 4.4, where `"${arr[@]}"` on an
empty array under `nounset` raised `unbound variable`. Bash 4.4+
treats an unset/empty array reference as zero words, not an error,
so the guard is dead weight -- and R-025 already requires the
`arr=()` init that makes the plain form safe on every path. Only
`${name[@]+...}` is flagged; `${#arr[@]}` (length), a plain
`${arr[@]}`, and the `${arr[@]:-default}` / `${arr[@]:+word}`
conditional-substitution forms are legitimate and spared.


**R-027: Hoist a `$(cmd)` out of another command's ARGUMENTS and out of
an `if`/`while` condition into a named variable on its own line.**
_auto-detected: no | auto-fixed: no_

Bad:

    if var="$(cmd)"; then ...
    safe-rm --force -- "$(resolve_dir "${id}")/muted"

Good:

    var="$(cmd)" || return 1
    if [ -n "${var}" ]; then ...
    local dir
    dir="$(resolve_dir "${id}")" || return 1
    safe-rm --force -- "${dir}/muted"

Why: a `$(...)` embedded in another command's argument MASKS the inner
command's exit code -- on failure the outer command silently runs with a
partial/empty argument. Hoisting lets `errexit` (or an explicit `||
return`/`|| die`) catch it first; and under `set -x` a separate
assignment line traces the resolved value before the branch. A plain
top-level `var="$(...)"` already lets errexit catch the failure (see
R-011, R-022 for `local`, R-033 for `printf`).


**R-028: Default an unbound variable at its SOURCE with `[ -v VAR ] ||
VAR=value`, not with per-consumer `${var:-}`.**
_auto-detected: no | auto-fixed: no_

Why: `set -o nounset` exists to catch a genuinely-unset variable as a
bug; once every consumer defaults it away with `${var:-}`, a real
missing-value bug passes silently. Set it once where it is defined --
ideally derived from a related always-set variable -- so consumers
reference it bare and nounset still catches the next real omission. Use
`[ -v VAR ] || VAR=value` (then `export` if needed): it defaults only
when the name is truly UNSET, so an intentional empty value is respected
(`${VAR:-}` clobbers it) and it is safe under nounset (`[ -v VAR ]` does
not dereference). Reserve `${var:-}` for genuinely-optional variables.


## printf

**R-030: Always `printf '%s\n' "..."`.** Format string is fixed;
all data goes in the data string. No `%d`, no `%q` (except where
shell-escaping is genuinely required), no extra `\n` in the format.
_auto-detected: yes | auto-fixed: no_

Numeric-probe carve-out (GATE-ENFORCED as an exemption): a `printf`
with a SINGLE-quoted literal format whose own command discards BOTH
stdout and stderr is a validator, not output, and keeps its format
verb. `is_integer` in helper-scripts' `strings.bsh` is the case:

    printf '%d' "$1" >/dev/null 2>&1 || return 1

Nothing is emitted, so neither of this rule's failure modes is
reachable, and the printf's FAILURE on a non-number is the check
R-141 relies on -- rewriting the format to `%s` would silently turn
that guard into one that always succeeds. Discarding stdout alone,
or `2>&1 >/dev/null` (which sends stderr to the original stdout),
does not qualify: those still emit.

**R-031: Multi-line block: ONE quoted string with embedded
newlines.** Multiple separate lines: one `printf '%s\n'` per line.
Blank line: `printf '%s\n' ""`, NOT `printf '\n'` by itself.
_auto-detected: yes | auto-fixed: no_

A standalone newline is ALWAYS `printf '%s\n' ""`, with the empty
string passed as an explicit data argument. Both `printf '\n'` (the
newline baked into the format string) and a bare `printf '%s\n'`
(the `%s` format kept but the data argument omitted) are forbidden
and GATE-ENFORCED -- they fail the static gate. Whether the blank
line should exist at all is R-042's separate call; this rule only
fixes its form once you decide to write one.

**R-032: Quote choice.** Double quotes preferred. Single quotes
acceptable when the body has many doubles to escape:
_auto-detected: no | auto-fixed: no_

    printf '%s\n' '"has" "a" "lot" "quotes"'

**R-033: Don't inline `$(cmd)` in a printf format string.** Pre-
compute into a named variable.
_auto-detected: no | auto-fixed: no_

Bad:  `printf '%s\n' "warn: $(my_helper "${value}")"`

Good:

    result="$(my_helper "${value}")"
    printf '%s\n' "warn: ${result}"

**R-034: Never `echo`; always `printf '%s\n'` (see R-030).** `echo`
flag handling is problematic.
`printf '%s\n' "${data}"` is unambiguous. The gate flags `echo`
used as a command; a file that genuinely needs it carries a
script-wide `## style-ok: allow-echo` waiver (same shape as
`no-safe-rm`).
_auto-detected: yes | auto-fixed: no_


**R-035: Prefer pure-bash text processing over `awk`.** Express
line-scans and counts with `while read -r` (+ associative arrays) rather
than an embedded `awk` program.
_auto-detected: no | auto-fixed: no_

Why: an `awk` program is a second language inside a bash file, and
`read < <(awk ...)` hides failure modes a native loop does not have
(SIGPIPE, exit-status masking -- see R-014/R-015); `awk` is also not
`--`-safe. Applies on TOUCH (write/refactor), like other style debt --
do not mass-rewrite pre-existing `awk` unprompted. With assoc arrays:
an empty subscript is invalid, so bucket empties under a placeholder,
and `read ... || [ -n "${var}" ]` keeps an unterminated final line.


## printf vs log

**R-040: Output to user goes through `log`, not bare printf.**
Every line written to the operator's terminal uses `log notice` /
`log warn` / `log error` (helpers from
`helper-scripts/log_run_die.sh`). The helpers prefix with the
script name and a level tag (`script.sh [NOTICE]: ...`).
_auto-detected: no | auto-fixed: no_

Why: the operator gets context for each line - which tool is
talking and at what severity. Bare `printf '%s\n'` to stdout/
stderr loses both. Tests that grep substrings inside the line
still match because the log helper preserves the body verbatim
after the prefix.

**R-041: Reserve `printf` for cases where it is genuinely the
right tool.**
_auto-detected: no | auto-fixed: no_

- writing to a file or pipe: `printf '%s\n' "${name}" >> "${file}"`
- feeding a value through a subshell to another tool:
  `names="$(printf '%s' "${body}" | jq -r '.[].name')"`
- building strings via `printf -v`

**R-042: Drop blank-line separators (`printf '%s\n' ""` /
`log notice ""`).** Once every line carries a `[NOTICE]:` prefix,
blank lines are noise.
_auto-detected: no | auto-fixed: no_

**R-063: A `printf -v` with a DYNAMIC target name must be guarded by
`check_variable_name` on that same name first.** Bash evaluates an
array subscript inside the `-v` target, so `printf -v "${name}"` with
an unvalidated `name` of the form `x[$(cmd)]` RUNS `cmd` -- a command
injection driven by whatever supplied the name (`helper-scripts`
`strings.bsh` carries the regression test for exactly this).
_auto-detected: yes | auto-fixed: no_

    check_variable_name "${name}" || return 1
    printf -v "${name}" '%s' "${value}"

The gate FAILS on a `printf -v` whose name is built from an expansion
unless a `check_variable_name` naming that same variable appears
earlier in the same function. A LITERAL target (`printf -v out ...`)
carries no expansion and is spared. A file that genuinely needs an
unguarded dynamic `printf -v` carries a script-wide
`## style-ok: allow-unchecked-printf-v` waiver (same shape as
`no-safe-rm`).


**R-064: `read -a` / `readarray` / `mapfile` take the array NAME as the
argument right after `-a`, so a `--` end-of-options separator can NEVER
sit before that name** (see R-062).
_auto-detected: no | auto-fixed: no_

    read -r -a -- arr     # `--': not a valid identifier
    read -ra -- arr       # same

`--` is valid only AFTER the array name (`read -r -a arr --`) but
pointless there -- end-of-options only protects a following dash-prefixed
DATA value, and array data comes from stdin, not argv. Plain `read -r --
var` (no `-a`) IS valid. Consequence: a blanket "add `--`" sweep breaks
every `read -a`; the fix is to REMOVE the `--`, not reorder. Find them
with `grep -rnE '\b(read|readarray|mapfile)\b[^|;&]*\s--\s'`.


## Functions

**R-050: A function definition's closing `}` is followed by
exactly one blank line.** End-of-file is the only exception.
_auto-detected: no | auto-fixed: no_

    foo() {
       local x

       x="$1"
    }

    bar() {
       ...
    }

Why: without a blank line the next block runs into the function
body visually and the boundary is hard to spot in large files.

**R-051: Trap targets are standalone named functions, never inline
command strings.**
_auto-detected: yes | auto-fixed: no_

    foo_cleanup_tmp() {
       safe-rm --force -- "${tmp_file}"
    }

    foo() {
       local tmp_file

       tmp_file="$(mktemp)"
       trap foo_cleanup_tmp RETURN
    }

Why: the trap function references variables from the calling
scope via dynamic scoping; registering AFTER vars are initialized
means the reference is `nounset`-safe with no `${var:-}` default.

**R-052: Backgrounded children (`&`) cannot mutate the parent
shell's variables.** If a per-item loop with `&` needs shared
state (a counter, a flag), use other IPC mechanisms (flag files,
STDIO, etc.).
_auto-detected: no | auto-fixed: no_

Why: child shells get a copy of the parent's vars; assignments
inside the child are lost on `wait`.

**R-053: Always use the strings 'true' and 'false' for booleans.** Do
not use other truthy/falsey values (1/0, y/n, on/off) unless passing
values to another tool that does not understand 'true' and 'false'.
_auto-detected: no | auto-fixed: no_

Why: All code should use the same convention for booleans to avoid
mismatch bugs. The convention for Kicksecure and Whonix's codebase is
to use the strings 'true' and 'false'.


## Flags

**R-060: Long flag names whenever the tool supports one.**
`--quiet`, `--ignore-case`, `--lines=1`, `--unique`,
`wc --lines`, `sort --unique`.
_auto-detected: no | auto-fixed: no_

Why: long flags self-document; survive being copy-pasted into a
context without `man <tool>` open; reviewers don't need to recall
short-flag meanings.

**R-061: Split combined short flags.** `rm -rf` -> `rm -r -f`,
`declare -gA` -> `declare -g -A`.
_auto-detected: no | auto-fixed: no_

**R-062: Use `--` end-of-options separator wherever the tool
supports one and positional args follow.** Verified working in:
`git`, `grep`, `sed`, `tr`, `jq`, `head`, `tail`, `stat`,
`mktemp`, `wc`, `sort`, `cat`, `rm`, `safe-rm`, `mkdir`,
`sudo` (`sudo -- cmd args` ends sudo's OWN options, before the
command word). Verify before extending the list.
_auto-detected: yes | auto-fixed: yes_

NB: `find` is deliberately NOT in this list. `--` ends find's option
parsing but does NOT protect a starting path that begins with `-`:
`find -- -leading` still dies with "unknown predicate". A find starting
path that may begin with `-` must be written `./path` or an absolute
path -- `--` is not sufficient.

Why: a positional that begins with `-` (legitimate or hostile)
gets treated as a flag without `--`.

The positive half ("use `--`") is convention, not gate-enforced:
whether a positional follows and could begin with `-` is
undecidable by a single grep, so a positive gate would be noisy.

The NEGATIVE half IS gate-enforced. A `--` handed to a tool that
does NOT accept it is a bug -- the tool takes `--` as a literal
argument or errors out. The gate FAILS on a `--` passed to any
tool on a verified denylist. Verified rejecters:

- `git check-ref-format` -- `git check-ref-format -- <ref>` exits 129.
- `stcat` -- it takes EVERY argument as a path, so `stcat -- <file>`
  tries to read a file literally named `--` and dies with
  `FileNotFoundError`. Adding the separator here broke
  helper-scripts' `read_integer_file`, which then reported
  "Cannot stcat target file" for a file that was present and
  readable, and took four of tb-updater's e2e scenarios with it.

Extend the denylist only after confirming against the
actual binary. NB: `echo` also mishandles `--` (prints it
literally) but is already banned outright by R-034.


## Case statements

**R-070: A case arm is fully multi-line: the pattern label, each
statement, and the closing `;;` each on their own line.** No compact
one-liner arms, spaced or jammed (`amd64) arch="x86_64" ;;` and
`amd64) arch="x86_64";;` are both wrong).
_auto-detected: yes | auto-fixed: no_

    amd64)
       arch="x86_64"
       ;;
    "")
       arch=""
       ;;

The `;;`-on-its-own-line half is GATE-ENFORCED: any `;;` with other
content on the line fails the static gate. The one-element-per-line
half (a label or statement must not share a line) is manual review;
a bare `)` is too ambiguous to grep (`$(...)`, `func()`, arithmetic,
globs), but a compact arm trips the `;;` check anyway.

**R-071: (folded into R-070.)** One element per line in a case arm.
_auto-detected: no | auto-fixed: no_

**R-072: Reserved-name and metachar-looking literals are quoted.**
`'.git'` not `.git`, `'-'*` not `-*`.
_auto-detected: no | auto-fixed: no_

**R-073: Quote interpolated values in case patterns: `"${x}"`,
not `${x}`.** Bash does not interpret `|` characters in an expanded
variable as special in this context. Only single-value semantics are
supported.
_auto-detected: no | auto-fixed: no_

Why: shellcheck SC2254 fires on the unquoted form. The quoted
form makes the interpolation a literal pattern. If you need
multi-value alternation, build the case manually rather than
expanding a `|`-separated string.

    case "${kind}" in
       repo)
          max_len="${MAX_REPO}"
          ;;
       user)
          max_len="${MAX_USER}"
          ;;
    esac

**R-074: No `; next-command` chaining.** Each statement gets its
own line. Bash's syntactic `;` (case-arm `;;`, C-style for-loop
`for ((i=0; i<N; i++))`) is the only exception; using `;` to
glue two arbitrary commands onto one line is prohibited.
_auto-detected: yes | auto-fixed: no_

The control-flow keywords `break`, `continue` and `return` are the
commonest offenders (loop bodies, one-line `if`s). A `;`-chained
`break`/`continue`/`return` is GATE-ENFORCED -- it fails the static
gate -- so always put the keyword on its own line. (A case arm cannot
produce this form under R-070, which already forbids the one-liner.)

Bad:

    cd "${dir}"; ls --long
    foo --quiet; bar --verbose
    if match; then hit=1; continue; fi
    [ -e "${x}" ] && { found="${x}"; break; }

Good:

    cd "${dir}"
    ls --long

    foo --quiet
    bar --verbose

    if match; then
       hit=1
       continue
    fi

    if [ -e "${x}" ]; then
      found="${x}"
      break
    fi

    --)
       shift
       break
       ;;


## Sourcing helper-scripts

**R-080: Pair every `source` with a `# shellcheck source=<relative
source-tree path>` directive on the line above; the source line
itself uses the system install path (with `${HELPER_SCRIPTS_PATH:-}`
on helper-scripts).** The directive is the lint hint; the source
line is the runtime resolution.
_auto-detected: yes | auto-fixed: no_

    # shellcheck source=../../../helper-scripts/usr/libexec/helper-scripts/<file>
    source "${HELPER_SCRIPTS_PATH:-}"/usr/libexec/helper-scripts/<file>

    # shellcheck source=../libexec/developer-meta-files/<lib>.bsh
    source /usr/libexec/developer-meta-files/<lib>.bsh

Why: the canonical context is the `derivative-maker` checkout with
submodules; the source-tree copy of the lib (next to the consumer)
may differ from the installed copy (`/usr/libexec/...`). The
relative `source=` directive points shellcheck at the source-tree
copy that the operator is actually editing, so lint findings track
the in-flight state. The runtime `source` keeps the system install
path so the script works the same on a packaged install.

Forbidden forms:

- `# shellcheck source=/usr/libexec/...` (absolute system path):
  points at the installed copy, which may drift from the source
  tree under review.
- `# shellcheck source=/home/<user>/...` (absolute developer path):
  not portable across machines / CI.
- `# shellcheck source=/dev/null`: silences cross-file checks
  entirely (also covered by R-081).
- `# shellcheck source=<bare-name>` with no `./` or `../` prefix
  (e.g. `source=get_colors.sh`): shellcheck resolves it the same, but
  the convention anchors a same-directory sibling as `./get_colors.sh`.

GATE-ENFORCED: the `source=` path must start with `.` (a `./` or `../`
relative source-tree path); an absolute or bare-name path fails the
static gate.
- Omitting the directive when shellcheck can resolve the path on
  its own: works for installed-path sources but doesn't track the
  source-tree copy; mandate the directive uniformly for predict-
  ability.

Path conventions by depth (relative to the script's directory):

| Script location | Self-lib | helper-scripts |
| --- | --- | --- |
| `usr/bin/<s>` | `../libexec/developer-meta-files/<lib>` | `../../../helper-scripts/usr/libexec/helper-scripts/<file>` |
| `usr/libexec/developer-meta-files/<lib>` | `./<other-lib>` | `../../../../helper-scripts/usr/libexec/helper-scripts/<file>` |
| `ci/<s>` | `../usr/libexec/developer-meta-files/<lib>` | `../../helper-scripts/usr/libexec/helper-scripts/<file>` |
| `ci/tests/<s>` | `../../usr/libexec/developer-meta-files/<lib>` | `../../../helper-scripts/usr/libexec/helper-scripts/<file>` |

In a standalone single-repo checkout helper-scripts is not a sibling,
so shellcheck would emit SC1091 for that line. CI provides the sibling
via the `helper-scripts: true` flag (see R-085); the canonical
derivative-maker checkout already has it as a submodule sibling. Do NOT
silence it with a per-line `# shellcheck disable=SC1091` (R-085). For a
self-lib in a standalone checkout the in-tree copy is what matters, and
its SC1091 there is acceptable.

**R-081: Never fall back to `source=/dev/null`.** That silences
cross-file checks.
_auto-detected: yes | auto-fixed: no_

**R-085: No `# shellcheck disable=SC1091` on a helper-scripts
source.** The `pre-push-static` gate checks out
helper-scripts as the repo sibling when the consumer sets
`dist-ai-tests: helper-scripts: true` in `.github/dm-consumer.yml`, so
shellcheck FOLLOWS the `# shellcheck source=` directive instead of
emitting SC1091. The per-line disable is then dead code; drop it and
set the flag. See the `shellcheck` skill. Waiver:
`## style-ok: allow-sc1091-disable` for a genuinely unfollowable
optional source.
_auto-detected: yes | auto-fixed: no_

**R-082: Each consumer sources every helper-scripts file it uses
directly.** Don't rely on transitive sourcing.
_auto-detected: no | auto-fixed: no_

Why: `log_run_die.sh` happens to source `strings.bsh` for its own
use; if your script also calls `is_whole_number`, source
`strings.bsh` itself. Otherwise a future refactor that drops the
transitive source breaks your script silently.

**R-083: `wc` invocations are preceded by sourcing
`wc-test.sh`.**
_auto-detected: no | auto-fixed: no_

Why: this makes a broken `wc` binary fail loudly rather than silently
producing an empty count.

**R-084: Reuse strings.bsh helpers before reimplementing.**
`is_whole_number`, `validate_safe_filename`,
`check_is_alpha_numeric`, etc.
_auto-detected: no | auto-fixed: no_


## Command availability checks

**R-090: `has` from `helper-scripts/has.sh`, not
`command -v X >/dev/null 2>&1`.**
_auto-detected: yes | auto-fixed: no_

    # shellcheck source=../../../helper-scripts/usr/libexec/helper-scripts/has.sh
    source "${HELPER_SCRIPTS_PATH:-}"/usr/libexec/helper-scripts/has.sh

    has github-org-fork \
       || die 1 "'github-org-fork' not on PATH"

The `source=` depth follows the R-080 table (this one is for a
`usr/bin/` script); it must be relative, never absolute.

Why: `has` verifies the result is executable and guards against
aliases/functions; `command -v` matches any of those. To deviate
(rare, typically bootstrap scripts that run before helper-scripts
is installed and aren't on the R-093 allowlist), put
`## style-ok: no-has` anywhere in the script; the pre-push gate
skips R-090 script-wide when it finds that marker. Prefer a path
test (`[ -x /usr/sbin/foo ]`) over the waiver where the binary's
install location is fixed.

**R-091: Pre-flight checks at the top, not scattered.** A tool's
runtime command dependencies are checked once, near the top of
the script's setup phase, not lazily inside the function that
happens to need each one.
_auto-detected: no | auto-fixed: no_

Why: lazy checks fail halfway through a per-repo loop and leave
partial state behind; a top-of-file pre-flight bails before any
mutation runs.

**R-092: Where a family of tools shares the same deps, the lib
provides a single helper.**
_auto-detected: no | auto-fixed: no_

    ghorg_require_deps    ## base set, including git

Don't add an inline `has git || die ...` at the per-feature site
when the shared pre-flight already covers it.

**R-093: Exception for `.github/actions/install-deps/install-helper-scripts.sh`.**
That script runs BEFORE helper-scripts is installed, so it falls
back to plain `command -v`. The same exception applies to
`pre-push-static` (dist-ai), which must run as a bare git hook
without sourcing helper-scripts.
_auto-detected: no | auto-fixed: no_


## Workflow scripts

**R-100: Substantial bash logic does not belong inside a workflow
YAML's `run: |` block.** If the step is more than 5 shell
statements (counted from a real bash parse, so a single command
wrapped over many backslash-continued lines is one statement, not
a block), or carries substantial control flow (a retry loop,
polling, a real error handler), put it in a standalone script
under `ci/` and have the workflow call it. A small pre-flight
guard (a single `if`/`exit`) may stay inline.
_auto-detected: yes | auto-fixed: no_

    - name: Start systemd-enabled Debian container
      run: ./ci/dry-run-start-container.sh dryrun "${DEBIAN_IMAGE}"

Why: shellcheck only sees real `.sh` files, not YAML blocks;
inline shell silently bypasses linting. A standalone script is
usable from a developer machine. Diff reviews are line-level instead
of YAML-indent-embedded. The script's args are an explicit, named,
testable contract.

**R-101: Workflow YAML and its scripts share a prefix.**
_auto-detected: no | auto-fixed: no_

    .github/workflows/dry-run.yml
                      ^  (same prefix)
                      v
    ci/dry-run-derivative-maker.sh

Why: a reader scanning either folder finds the matching
counterpart at a glance.

**R-102: Don't prepend the interpreter when the shebang suffices.**
A script with a `#!/bin/bash` shebang and executable bit, invoked
as `path/script.sh`, runs under its declared interpreter. Adding
an explicit `bash` (or worse, `sh`) prefix is redundant or
actively wrong.
_auto-detected: yes | auto-fixed: no_

Bad:

    bash build.sh
    sh ci/foo.sh
    bash ci/dry-run-start-container.sh ...

Good:

    ./build.sh
    ci/foo.sh
    ./ci/dry-run-start-container.sh ...

Why: `sh script.sh` runs the script under /bin/sh, NOT bash,
regardless of the shebang line. Bash-specific syntax (arrays,
`[[ ]]`, `local`, `set -o pipefail`) silently breaks or behaves
weirdly. `bash script.sh` is merely redundant when the shebang
already says bash, but it also defeats the contract -- the
shebang declares the interpreter; the invoker shouldn't override
it. Applies to CI YAML `run:` blocks, Makefile recipes, wrapper
scripts, and ad-hoc invocations.

Exception: bootstrap that runs before the executable bit is set
(fresh `git clone` with `core.fileMode=false`, or a tarball that
lost +x), or surfaces that don't honor the shebang. State the
reason inline.

**R-193: Call an in-repo Python script directly, not via `python3 --
<file>.py`.** R-180 makes every `*.py` executable with a shebang, so
run it like any other program.
_auto-detected: yes | auto-fixed: no_

Bad:

    python3 -- "${dir}/report-summary.py" "${report}"

Good:

    "${dir}/report-summary.py" "${report}"

Why: the same contract R-102 states for shell -- the shebang declares
the interpreter, the caller should not restate it. It matters more
here: a `python3 -- file` prefix DROPS the shebang's own flags
(`#!/usr/bin/python3 -Bsu`), so the direct call is not just tidier, it
runs the interpreter the file asked for. A generic dispatcher passing
`"$@"` (no literal path) is glue, not a named call, and is fine.

Enforced by R-193 in the pre-push gate: it flags a literal
`<interpreter> -- <path>.py` at quote depth zero, outside comments.
Waiver: `## style-ok: allow-python-dashdash` for a script deliberately
not executable, or an external path you don't control.

**R-103: Don't replace the process with `exec <command>`; run it as
a child and forward the exit code.** Process-replacement `exec`
drops the wrapper from the `ps` tree (harder to debug) and skips
any cleanup the wrapper would run on exit.
_auto-detected: yes | auto-fixed: no_

Bad:

    exec sandbox-run --dir "${repo}" -- ./tests/suite.sh "$@"

Good -- just run it. A script's exit status is its last command's
exit status, so a plain call as the final line forwards the code
already; under `set -o errexit` a failure exits immediately with
the child's status, and any `trap ... EXIT` cleanup still runs
(exactly what `exec` would skip):

    sandbox-run --dir "${repo}" -- ./tests/suite.sh "$@"

Only reach for an explicit capture when you must run cleanup on
*every* exit path and still return the child's original code, and
that cleanup is inline rather than a `trap ... EXIT` handler. The
`|| rc=$?` is load-bearing here: it disarms `errexit` so the
teardown runs instead of the script aborting on failure:

    rc=0
    sandbox-run --dir "${repo}" -- ./tests/suite.sh "$@" || rc=$?
    teardown_temp_dirs
    exit "${rc}"

Do not add that ceremony when the command is simply the last thing
the script does -- the plain call is equivalent and cleaner.

This rule targets *process replacement* only. `exec` used purely to
open/redirect a file descriptor (`exec 9>"${lock}"`, `exec
{fd}>&-`, `exec >"${log}"`) is not process replacement and is not
flagged. A surface that genuinely needs to hand off the process
(a remote-command payload where a lingering wrapper would deadlock
the transport; a pty/login shim) carries a script-wide `##
style-ok: allow-exec` waiver stating the reason.

**R-104: Prefer multi-line, multi-step over a long single-line
pipeline.** A pipeline of five or more stages on one physical line is
hard to read, debug and diff. Assign an intermediate, or backslash-
continue one stage per line, so each step is named and inspectable.
_auto-detected: no | auto-fixed: no_

Bad:

    top="$(grep pat log | cut -f2 | sort | uniq -c | sort -rn | head)"

Good:

    matches="$(grep pat log | cut -f2)"
    counts="$(printf '%s\n' "${matches}" | sort | uniq -c)"
    top="$(printf '%s\n' "${counts}" | sort -rn | head)"

Why: a wedged or wrong stage in a six-stage one-liner leaves you no
intermediate to inspect, and a diff touching one stage rewrites the
whole line. Naming the intermediates turns "the pipeline is wrong"
into "step 2 is wrong."

Guidance only -- there is deliberately NO pre-push gate for this. A
`|` is too overloaded in shell (pipe operator, `||`, `case`/glob
alternation, a pipe nested in `$(...)`, a literal inside a string) for
a static rule to tell a long pipeline from an ordinary multi-line
`case` pattern without false positives, and the tree already follows
the convention. Keep it by review, not by gate.


## Errors and logging

**R-110: Use `log` and `die` from `helper-scripts/log_run_die.sh`,
not ad-hoc `printf >&2; exit N`.**
_auto-detected: no | auto-fixed: no_

    log error "couldn't read '${path}'"   ## log only
    log warn  "..."
    log notice  "..."
    log info  "..."
    die 1 "fatal: ..."                     ## logs error then exit 1
    [ "$#" -ge 2 ] || die 64 "missing value for --include"

Why: `die <code> <msg>` is the one-liner for "log error then
exit." Inside a function that should return rather than exit, use
`log error "..."; return N`.


## File deletion

**R-120: `safe-rm`, not `rm`.** Long-flag form: `safe-rm --force --`
or `safe-rm --recursive --force --`. To deviate (rare), put
`## style-ok: no-safe-rm` anywhere in the script; the pre-push
gate skips R-120 script-wide when it finds that marker.
_auto-detected: yes | auto-fixed: no_

Why: `safe-rm` consults a blocklist before deleting (paths like
`/`, `/usr`, `~`).


## Null command

**R-130: `true` instead of `:` for no-op placeholders.** Pass a
descriptive message so xtrace logs convey intent:
_auto-detected: yes | auto-fixed: no_

    true "INFO: ghorg_api: HTTP 429 - will retry"

**R-131: Bare `true > "${file}"` is fine for "truncate file".
`while true` is fine for an infinite loop.**
_auto-detected: no | auto-fixed: no_


## Untrusted external data

**R-140: Treat every byte returned by an external service as
untrusted.**
_auto-detected: no | auto-fixed: no_

- **Identifier sinks** (URL paths, file paths, command-line
  arguments): pass through a strict allowlist validator (e.g.
  `^[A-Za-z0-9._-]+$`) or a numeric-only regex with length cap
  before use.
- **Display sinks** (printf/log to stdout/stderr): pass through a
  sanitizer (e.g. `sanitize-string`) that strips ANSI escapes,
  control chars, HTML markup, Unicode, and truncates oversized
  payloads.
- **Don't sanitize the raw API body before parsing** - the parser
  (jq) is the schema validator. Sanitize after extraction, before
  display.

Why: the validator enforces what the consumer actually accepts;
the sanitizer enforces what's safe to render to a terminal. Both
are needed; one doesn't substitute for the other.

The github-org-* / dm-* tools implement R-140 via
`ghorg_validate_name` and `ghorg_safe_print`; see
[github-org-tools.md](github-org-tools.md) G-001 through G-004
for the project-specific implementation.

**R-141: Avoid or carefully guard code that causes implied `eval`.**
Arithmetic contexts (`(( ... ))`, `$(( ... ))`, any numeric comparison
options in `[[ ... ]]`), array indexing, dereferencing via `${!...}`,
the `-v` option of `test` and `[ ... ]`, and `printf -v` all can cause
`eval`-like behavior, where code in string literals is executed as if
it were part of the script. This injection can be done by passing a
string such as `a[$(date)]` where a variable name or integer is
expected. The following rules MUST be followed to avoid code injection
when using these features of Bash:
_auto-detected: no | auto-fixed: no_

* If a variable is expected to contain an integer but comes from a
  potentially untrusted source (i.e. a function argument), verify it
  using `is_integer` or `is_whole_number` from helper-scripts'
  `strings.bsh` library. This must be done BEFORE using the variable
  in an arithmetic context or as an array index.
* If a variable is expected to contain a variable name but comes from
  a potentially untrusted source, verify it using
  `check_variable_name` from `strings.bsh`. This must be done BEFORE
  assigning the string to a nameref variable, dereferencing it,
  checking for its existence as a variable with `test -v ...` /
  `[ -v ... ]`, or setting a variable with its name with `printf -v`.
* Prefer using `[ ... ]` over `[[ ... ]]` where possible.
* Be very cautious when passing an array by name to a function and
  setting its value in the function. In particular, do not EVER use
  `printf -v` to set the value of an array element:

    ## This is bad; arbitrary code can be injected via `arr_idx`:
    bad_fn() {
      local arr_name="$1" arr_idx="$2" element_val="$3"
      printf -v "${arr_name}[${arr_idx}]" '%s' "${element_val}"
    }

    ## This is good provided that `arr_ref` is set to the name of an
    ## associative array, but allows arbitrary code to be injected via
    ## `arr_idx` for non-associative arrays:
    good_for_assoc_array() {
      local arr_idx element_val
      local -n arr_ref

      check_variable_name "$1" || return 1
      arr_ref="$1"
      arr_idx="$2"
      element_val="$3"

      arr_ref["${arr_idx}"]="${element_val}"
    }

    ## This is good for non-associative arrays, no code injection is
    ## possible:
    good_for_normal_array() {
      local arr_idx element_val
      local -n arr_ref

      check_variable_name "$1" || return 1
      arr_ref="$1"
      arr_idx="$2"
      element_val="$3"

      is_whole_number "${arr_idx}" || return 1
      arr_ref["${arr_idx}"]="${element_val}"
    }


## Comments

**R-150: State rationale once per file.** Don't copy-paste a
multi-line `Why` block to multiple sites; at subsequent sites,
drop the comment or use a one-liner referencing a rule ID
(`R-NNN`, `G-A-NNN`, `W-NNN`).
_auto-detected: no | auto-fixed: no_

Why: copy-pasted rationale rots - site N+1 drifts from site 1
over time; readers stop trusting all of them. Single source of
truth survives. Applies to any source file the org maintains
(bash, YAML, python, markdown).


**R-151: Comment when the code couldn't express the intent.** A
comment is an admission the code failed to express itself; prefer
renaming, extracting, or restructuring first. When unavoidable,
reserve comments for hidden constraints, subtle invariants, bug
workarounds, surprising side effects. Don't restate WHAT (well-
named identifiers do that). Bad: `## initialize i with 0` over
`i=0`.
_auto-detected: no | auto-fixed: no_

Why: obvious comments dilute attention from the ones that matter;
reviewers learn to skim past them and miss the rare comment
documenting a real gotcha. Be concise: if removing the comment
wouldn't confuse a future reader, don't write it.

Tooling: the `comments-audit` heuristic (ships in dist-ai on PATH; run
`comments-audit <repo_root>` or `comments-audit --files FILE...`) flags
likely R-151 candidates. The `pre-push-static` gate runs it over the
changed files as an ADVISORY -- it prints candidates but never fails the
gate, because the heuristic has false positives and a human decides.


**R-152: Match the file's existing comment style.** Before
adding comments to an existing file, read the comments already
there - density, tone, idiom, voice - and match them. Don't
impose your preferred style on a file someone else established
(unless an explicit rule above says you must).
_auto-detected: no | auto-fixed: no_

Why: file-local consistency keeps each file readable as a unified
document; jarring shifts in voice signal copy-paste and undermine
trust in the prose. Match locally; impose org-wide style only
when it would otherwise conflict.

**R-153: Never extract a comment from the running script to display
it to the user.** A "help mode" is a dedicated function that PRINTS
the help; it must not scrape the source. In particular, never turn the
header comment into `--help` output with `grep '^##' -- "$0" | sed ...`.
_auto-detected: yes | auto-fixed: no_

Why: code that treats comments as user-interface text breaks the moment
a comment-only edit is made, and it couples the help wording to comment
syntax.

Canonical pattern -- a full help (both `-h` and `--help` print it) and
a short usage shown only when a REQUIRED argument is missing, kept as
two separate strings you own:

    me="${0##*/}"
    print_usage() {                 # short: the usage line(s) only
       printf '%s\n' \
          "Usage:" \
          "  ${me} ARG [--opt VALUE]"
    }
    print_help() {                  # full: usage + description + options
       print_usage
       printf '%s\n' "" "Longer description ..." "" "  --opt VALUE   ..."
    }
    # -h and --help are the SAME: both print the full help.
    #   -h|--help) print_help; exit 0 ;;
    # A tool that REQUIRES arguments prints the short usage when they are
    # missing (like 'mv'); one that runs argless (like 'nano') does not.
    #   [ -n "${arg}" ] || { print_usage >&2; error "ARG is required"; }

**R-154: No history in comments.** Comment the CURRENT state and why,
never how the code got there. Ban change-narrative: "formerly X",
"was broken", "moved from Y", "used to", "no longer", "renamed from".
A comment is not a changelog; keep it terse, bullet-style. Bad:
`## shared with foo (formerly inline here)`. Good: `## shared with foo`.
_auto-detected: no | auto-fixed: no_

Why: git carries the history. A comment narrating a past state goes
stale, misleads a reader who never saw that state, and grows on every
change. The diff and the log are the record.


## File search

**R-160: Never use the --quiet option of grep.** According to grep's
manpage, "if the -q or --quiet or --silent is used and a line is
selected, the exit status is 0 even if an error occurred." Silencing
errors is not acceptable. To silence grep's *output* (but not exit
code), append `>/dev/null 2>&1` to the end of the grep command. To
prevent grep from looking for more than one match, use `--max-count=1`
(but never on the reading end of a pipe -- see R-161).
_auto-detected: no | auto-fixed: no_

**R-161: A `grep` that consumes a pipe must not use a quiet flag, and
`grep` never takes a SHORT quiet flag.** Two related bans.
_auto-detected: yes | auto-fixed: yes_

*Pipe + quiet is a `pipefail` bug.* `-q` / `--quiet` / `--silent` make
grep exit at the first match. On the reading end of a pipe that closes
the pipe early, so the writer on the left gets `SIGPIPE` and dies with
141 -- and our default `set -o pipefail` (R-010) turns that into a
failed pipeline:

    seq 1 100000000 | grep --quiet 5     # pipeline exits 141, not 0

Whether it bites depends on how much the producer still had to write, so
it passes on small inputs and fails on large ones -- a latent,
size-dependent flake. Remedies:

- Streaming producer -- drop the quiet flag and send grep's stdout to
  `/dev/null`. grep then reads to EOF, so nothing is SIGPIPE'd; its exit
  code, and any real error on stderr, are preserved:

      producer | grep pattern >/dev/null

- Variable / string input -- use a here-string, which is a temp file,
  not a pipe, so there is no writer to kill. A quiet flag is fine here:

      grep --quiet pattern <<< "${var}"     # not  printf ... | grep -q

`--max-count=N` / `-m N` cause the SAME early exit; R-160's
`--max-count=1` tip is for a non-pipe grep only.

*Short quiet flag violates R-060.* `grep -q`, and any bundled cluster
carrying it (`grep -iq`, `grep -Fq`), use short options; write the long
form -- `--quiet`, `grep --ignore-case --quiet`, `grep --fixed-strings
--quiet`. A long quiet flag reading a file (`grep --quiet -- PAT FILE`)
stays the accepted boolean-test idiom; R-160's caveat is about masking a
real error, not about the flag existing.

Enforcement: the pre-push gate FAILS a quiet grep on the right of a `|`
(here-strings and plain file reads are spared) and a short quiet flag
anywhere; `pre-push-fix` auto-expands a short quiet cluster to its long
form. The pipe rewrite (drop the quiet flag, add the redirect or
here-string) is left to a human -- relocating a redirect is not a
mechanical single-token edit, the same line `pre-push-static` draws for
R-172's non-atomic `mkdir`.


## Temporary files

**R-170: Never hardcode `/tmp`. Initialise `TMP` once, then use
`"${TMP}/..."`.**
_auto-detected: yes | auto-fixed: no_

`libpam-tmpdir` gives every login session a private, mode-0700 temp
directory (`/tmp/user/<uid>`) and exports it as all four of `TMP`,
`TMPDIR`, `TEMP` and `TEMPDIR`. A path that names `/tmp` directly opts
out of that and writes into the world-writable root instead.

Set it at the top, with the other variable initialisations:

    [ -v TMP ] || TMP=/tmp

Then at every use site:

    work_dir="$(mktemp --directory -- "${TMP}/myscript.XXXXXX")"
    log_file="${TMP}/myscript.log"

Bad -- the fallback repeated inline at each use site:

    work_dir="$(mktemp --directory -- "${TMPDIR:-/tmp}/myscript.XXXXXX")"

Why: one initialisation is one place to audit and one place to change;
the inline form restates the `/tmp` default at every call, so a script
with six temp paths has six chances to disagree with itself. It is also
the same set-at-source discipline R-021 applies to every other variable,
rather than a `${var:-default}` guard per reference.

`mktemp` already honours `TMPDIR`, so keep using it -- it needs no
`/tmp` of its own.

**The four temp-dir variable initialisations are the only place the
literal belongs.** All of these are correct and are not violations:

    [ -v TMP ] || TMP=/tmp
    export TMPDIR=/tmp
    readonly TEMP="/tmp"
    bw+=(--setenv TMPDIR /tmp)          # bwrap namespace construction

**R-171: To use `/tmp` for anything else, waive the rule explicitly.**
Put `## style-ok: no-tmp-hardcode` anywhere in the script; the pre-push
gate then skips R-170 for that file (same mechanism as R-120's
`## style-ok: no-safe-rm`).
_auto-detected: no | auto-fixed: no_

Reserve it for paths that are not redirectable temp paths at all:

- `/tmp/.X11-unix` and `/tmp/.X<n>-lock` -- fixed by the X11 protocol;
  libX11 looks there and nowhere else, so it cannot follow `TMPDIR`.
- Namespace construction (`bwrap --tmpfs /tmp`): the private tmpfs has
  no per-user subdirectory, so `/tmp` is the only path that exists
  inside it.
- Administering the `/tmp` mount itself (`mount -o remount ... /tmp`) --
  that is the backing filesystem, not a path a variable could point
  elsewhere.

Not waiver material: a temp file that simply predates the rule.

Enforcement: the pre-push gate flags `/tmp` as an absolute path on a
non-comment line of a changed shell file. A path that merely ends in
`/tmp` (`debian/tmp`, `/var/tmp`, `./tmp`), one rooted in an expansion
or in HOME (`${build_dir}/tmp`, `$(pwd)/tmp`, `~/tmp`), or a longer name
starting with it (`/tmpfs`, `/tmp.bak`) is not matched. Comment lines
are excluded -- prose about `/tmp` is not a path.


**R-172: A `mkdir` that creates a temp directory must set the mode
ATOMICALLY with `--mode=`.**
_auto-detected: yes | auto-fixed: yes_

    mkdir --parents --mode=700 -- "${TMPDIR}"

`mkdir --mode=` applies the permission bits as part of the directory's
creation. Setting the mode any other way -- dropping it, or splitting it
into a following `chmod` -- leaves a window in which the directory exists
with the umask-default (world-traversable) mode. Another process can
enter that window and race the temp path; for a directory that will hold
a journal or any private data, that is a TOCTOU disclosure hole.

Bad -- the mode is not atomic (a `chmod` follows the create):

    mkdir --parents -- "${TMPDIR}"
    chmod 700 -- "${TMPDIR}"          # TOCTOU: dir is world-visible first

Bad -- no mode at all:

    mkdir --parents -- "${TMPDIR}"

Use the long `--mode`, not the short `-m`: `pre-push-fix` upgrades a
`-m 700` / `-m700` to `--mode=700` automatically, and the gate FAILS a
standalone short `-m` so the long form is what lands. `--mode=700` is the
canonical spelling; `--mode 700` (space) is equally atomic and accepted.
This is the same long-option discipline R-013 applies to `set -o`.

The mode is judged on the `mkdir` command itself, not the whole line: a
`--mode` in a trailing comment or in a second command sharing the line
does not satisfy the rule, and the fixer never rewrites another command's
options.

The atomic form pairs with a `# shellcheck disable=SC2174`:

    # shellcheck disable=SC2174
    mkdir --parents --mode=700 -- "${TMPDIR}"

`--parents` is what makes the create idempotent (re-running is fine when
the directory already exists); combined with `--mode=`, shellcheck raises
SC2174 -- "with `-p`, `-m` only applies to the deepest directory." That
is exactly the intent here: the parents (`/var/cache`, `~/.cache`, ...)
pre-exist, so only the temp directory itself is created and it gets the
mode atomically. There is no form that is both idempotent AND atomic
without the flag combination SC2174 warns about, so the disable is part
of the pattern -- `pre-push-fix` inserts it for you.

Waiver: `## style-ok: allow-mkdir-no-mode` anywhere in the script (same
mechanism as R-120's `## style-ok: no-safe-rm`). Reserve it for a temp
directory whose permissions genuinely do not matter, or a `mkdir` whose
mode is set by a form the rule cannot read (a symbolic `-m u=rwx`, a
`--mode="${mode}"` variable).

Enforcement: the gate flags a command-position `mkdir` whose operand is
a `TMPDIR` / `TMP` / `TEMP` / `TEMPDIR` variable and that carries no
`--mode=`, on a non-comment line of a changed shell file. A `mkdir` that
does not create one of those temp variables, and a name that merely
starts with the prefix (`${TMPFILE}`), are not matched.


**R-190: A substantial interpreter program does not belong in a
shell heredoc.** If the embedded body is more than ~5 lines, put it
in its own file with a shebang and call it.
_auto-detected: yes | auto-fixed: no_

    ## Bad -- invisible to every tool that would check it:
    summary="$(python3 - "${report}" <<'PY'
    ...40 lines of parsing...
    PY
    )"

    ## Good:
    summary="$("${helper_dir}/report-summary.py" "${report}")"

Why: this is R-100's defect in the other direction. ruff and pyrefly
only see real `*.py` files; coverage.py cannot measure a heredoc at
all; and no unit test can import a function that has no importable
home, so the body can only be exercised through the whole shell tool.
A 40-line parser embedded this way is typically the part that decides
what the tool concludes, and it is the part with no tests.

Resolve the helper RELATIVE to the calling script (prefer an in-tree
copy, fall back to the installed path) so editing the repo takes
effect without installing, and fail loudly when neither exists. A
silent fallback to a stale installed copy is worse than no fallback.

Short glue stays inline: a one-line `python3 -c` is not a program.

Waiver: `## style-ok: allow-inline-interpreter` anywhere in the file.

**R-191: A systemd unit does not embed a multi-statement shell
script.** An `Exec*=` directive must not carry embedded scripting:
_auto-detected: yes | auto-fixed: no_

    ## Bad -- invisible to shellcheck, no importable home, no coverage:
    ExecStart=/bin/bash -c 'mkdir -p /run/foo && chown x /run/foo; start'

    ## Good:
    ExecStart=/usr/libexec/foo/start

Why: the same defect R-100 catches in workflow YAML and R-190 in a
heredoc. The `-c` body is hidden from shellcheck, has no importable
home a unit test can reach, and no coverage tool can see it. Move the
logic into a script with a shebang and call that. A single-command
wrapper (`ExecStart=/bin/bash -c 'touch /run/foo'`) is glue, not a
program, and is allowed; only a `sh -c` / `bash -c` value carrying a
`;`, `&&`, `||`, a pipe, a shell control keyword (`for while if case
until do then`), or a `\` line-continuation is flagged.

Waiver: `# style-ok: allow-embedded-script` anywhere in the unit.

**R-194: An apt configuration hook does not embed a multi-statement shell
command.** A `Pre-Invoke` / `Post-Invoke` / `Pre-Install-Pkgs` directive runs
its double-quoted value through `sh -c`:
_auto-detected: yes | auto-fixed: no_

    // Bad -- a script inlined in a config file, invisible to every tool:
    DPkg::Post-Invoke {"if [ -x /usr/bin/foo ]; then /usr/bin/foo; fi";};

    // Good:
    DPkg::Post-Invoke {"/usr/libexec/mypkg/post-invoke-hook";};

Why: the same defect R-191 catches in a systemd unit. The command is hidden
from shellcheck, has no importable home a test can reach, and no coverage tool
sees it. Move the logic into a script with a shebang and call it from the hook.

Flagged: the INNER text of a hook's quoted value contains a `;` statement
separator or a `|` pipe. Tolerated as glue: a `&&` / `||` chain and a `|| true`
error-suppression tail -- an apt hook has no native alternative for them. The
`;` apt uses to terminate a directive or separate a `{...}` list element sits
outside the quotes and is config syntax, not a shell separator, so it is not
counted.

Scope: apt config paths only (`apt.conf.d/`, `apt.conf`). The check is per-line
and does not parse a rare multi-LINE brace block across lines (a config parser
for a rare case is the wrong tool -- see R-195's note); such a form is a
documented fail-open. Waiver: `// style-ok: allow-embedded-script` (or `#`).

**R-195: A cron entry does not embed a multi-statement command.** A cron table's
command field runs through `sh -c`:
_auto-detected: yes | auto-fixed: no_

    # Bad -- an inlined script in a crontab:
    0 3 * * * root cd /srv/app && ./purge.sh; systemctl restart app | logger

    # Good:
    0 3 * * * root /usr/local/bin/nightly-maintenance

Why: the same defect R-191 catches in a systemd unit -- invisible to shellcheck,
no importable home, no coverage.

Flagged: the command contains a `;` statement separator or a `|` pipe. Tolerated
as glue: `&&` / `||` chains and a `( ... )` subshell -- the stock `/etc/crontab`
itself uses `cd / && run-parts ...` and `test -x X || ( cd / && run-parts ... )`,
and cron has no native directive for a working directory or a conditional run,
so flagging that glue would fight the OS default. Blank lines, comments, and
environment assignments (`SHELL=`, `PATH=`, `MAILTO=`) are not commands and are
skipped.

Scope: cron tables only (`cron.d/`, a `crontab` file) -- NOT the
`/etc/cron.{daily,hourly,...}/` run-parts directories, whose entries are ordinary
executable scripts already covered by the shell rules. Waiver:
`# style-ok: allow-embedded-script`.

Both PARSE the extracted shell value before testing (the gate's shfmt-backed
detector), so a `;` / `|` that is DATA -- an escaped `find ... -exec rm {} \;`
terminator, or a `|` inside a quoted pattern (`awk '/foo|bar/'`, `grep -E 'a|b'`)
-- is correctly a single command, never a multi-statement one. A separator hidden
inside a quote is no longer a false positive: the parser knows it is string
content.

The general rule behind R-194/R-195 (and the shell command-position rules): never
HAND-ROLL a parser -- a quote/brace/heredoc state machine -- but USE a real one.
The gate and the auto-fixer analyse the shell VALUE with the shfmt AST (the
`pre-push-detect` detector and `pre-push-fix`, via `dist_ai.bash_ast`), so a
command, a separator, or a quote is told from data EXACTLY, not by a fragile regex
that has to accept a documented fail-open. Only the config-line EXTRACTION
(finding the value in an `apt.conf` / crontab / unit) stays a simple per-line
scan; a rare multi-LINE brace block is still declined, because a config-format
parser for that one case is the wrong tool.

## Python files

**R-180: A Python file carries a shebang and is executable.**
_auto-detected: yes | auto-fixed: no_

    #!/usr/bin/python3 -Bsu

Both halves, for every `*.py` -- library modules included, not just the
entry points in `usr/bin/`.

Why: so a module can be run directly while debugging, instead of
having to reconstruct an invocation for it. The mode is what makes the
shebang mean anything; a shebang on a non-executable file is a
statement the filesystem contradicts.

Exempt: an EMPTY file. A zero-byte `__init__.py` is a package marker,
not code, and has nothing to interpret.

Caveat: a module using RELATIVE imports still cannot be run as a plain
script -- `./mod.py` reports "attempted relative import with no known
parent package", because a script has no package context. Use
`python3 -m package.mod` for those. The shebang and mode are still
required: they document the interpreter and keep the file uniform.

Enforcement is split across three checks, which is why the rule is
stated as a pair. `check-shebang-scripts-are-executable` fails a
shebang without the mode; `check-executables-have-shebangs` fails the
mode without a shebang; R-180 in the pre-push gate fails a file with
NEITHER, which would otherwise slip past both.


## External command timeouts

**R-200: `timeout` should almost always carry `--kill-after=`.** A bare
`timeout <N> <cmd>` sends only `SIGTERM` after `<N>` seconds; a child
wedged in an uninterruptible syscall can ignore `SIGTERM` and keep
running, defeating the very bound `timeout` was added for. Add
`--kill-after=<K>` so `timeout` follows up with `SIGKILL` `<K>` seconds
later:
_auto-detected: yes | auto-fixed: yes_

    timeout --kill-after=5 5 -- eglinfo -B

`--kill-after=<K>` is the canonical spelling (R-060 long flags); the
short `-k` provides the same safety. `<K>` is the grace window after the
`SIGTERM`, not the total budget -- mirroring the main duration
(`--kill-after=5 5`) is a fine default.

Why: the whole point of `timeout` is a hard upper bound on wall-clock;
without the `SIGKILL` follow-up that bound is only advisory, and the one
process you most need to bound (a hung one, blocked in the kernel) is
exactly the one that ignores `SIGTERM`.

The rare legitimate bare `timeout` (a command that MUST be allowed to
finish its own cleanup on `SIGTERM`, or where `SIGKILL` would corrupt
state) waives it: put `## style-ok: allow-bare-timeout` anywhere in the
script and the pre-push gate skips R-200 script-wide.

The pre-push gate (and its auto-fixer) flag a `timeout` in COMMAND
position with no kill-after option. A `timeout` inside a string (the
deferred `x="timeout 5"` spelling) or used as another command's argument
is spared -- a line-based gate cannot safely reason about it -- so add
the option to those by hand.


## Package management

**R-210: `apt-get-noninteractive`, not `apt-get`.** Use the
helper-scripts wrapper for every `apt-get` invocation (including behind
`sudo`/`doas`):
_auto-detected: yes | auto-fixed: no_

    sudo apt-get-noninteractive update
    sudo apt-get-noninteractive install --yes --no-install-recommends -- foo

Why: the wrapper exports `DEBIAN_FRONTEND=noninteractive`,
`DEBIAN_PRIORITY=critical`, a `policy-rc.d`, and force-conf* options, so a
scripted install never blocks on a debconf prompt or a conffile question
(the class of hang that wedges an unattended build or a boot-time
install). A bare `apt-get` inherits the caller's frontend and stalls.

**R-211: `dpkg-noninteractive`, not `dpkg`, for state-changing actions.**
Any action that unpacks or changes package state -- `--install`/`-i`,
`--unpack`, `--configure`, `--remove`/`-r`, `--purge`/`-P`,
`--record-avail`/`-A`, `--{set,clear}-selections`,
`--{update,merge}-avail`, `--forget-old-unavail`, `--triggers-only` --
goes through the wrapper (`dpkg --force-confnew`):
_auto-detected: yes | auto-fixed: no_

    sudo dpkg-noninteractive --install --refuse-downgrade -- ./foo.deb

A read-only QUERY (`dpkg --compare-versions`, `-l`, `-L`, `-s`, `-S`,
`--print-architecture`, `--get-selections`, ...) and the separate
`dpkg-*` tools (`dpkg-deb`, `dpkg-query`) stay bare -- the wrapper's
`--force-confnew` is meaningless there, and forcing a query through it
would break early-boot code where the wrapper may be absent.

**R-212: never `--allow-downgrades`.** A silent downgrade masks a
dependency or repository regression that should fail loudly; rely on the
default refuse-downgrade behaviour (`dpkg --refuse-downgrade`).
_auto-detected: yes | auto-fixed: no_

**R-213: never `make_use_lintian=false`.** Disabling lintian on a
genmkfile build hides packaging defects. Fix the lintian findings
instead.
_auto-detected: yes | auto-fixed: no_

Why a wrapper file is exempt: the helper-scripts scripts that DEFINE
`apt-get-noninteractive` / `dpkg-noninteractive` necessarily call bare
`apt-get` / `dpkg` -- that is their job -- so the gate never flags them
(matched by basename).

The pre-push gate flags R-210 through R-213 on shell files (command
position, sparing an apt-get/dpkg inside a string or comment); its
auto-fixer rewrites R-210 (`apt-get` -> `apt-get-noninteractive`) but not
R-211/R-212/R-213 (dpkg needs an action-aware decision, and the other two
are a deliberate removal a human must make).

The ONLY sanctioned override is an explicit human-operator decision, for
an environment where the wrapper genuinely does not exist (a minimal CI
image without helper-scripts): put the matching waiver -- `## style-ok:
allow-apt-get`, `allow-dpkg`, `allow-downgrades`, or
`allow-lintian-disable` -- in the script, with a comment stating why. A
model must not add these waivers on its own; converting to the wrapper is
the default.

**R-220: no unauthorized SKIP.** A test SKIP -- `exit 77` / `return 77`,
the reserved skip code -- must be authorized:
_auto-detected: yes | auto-fixed: no_

    # Bad -- a required tool absent is an ENVIRONMENT BUG, not a skip:
    type -P helper-script >/dev/null || exit 77

    # Good -- a required tool absent fails LOUD:
    type -P helper-script >/dev/null || { printf 'FATAL: helper-script absent\n' >&2 ; exit 1 ; }

    # Good -- a genuinely OPTIONAL target may skip, WITH a reason:
    [ -x /usr/bin/optional-e2e-daemon ] || exit 77  ## style-ok: allow-skip: e2e-only daemon, absent in the core lane

Why: a skip added to make a red suite green is a silent pass -- the gate
exists to stop exactly that. A REQUIRED dependency's absence is an
environment bug and must be `exit 1` (FATAL), so it fails loudly and gets
fixed; only a genuinely OPTIONAL target may `exit 77`, and it must say why
in a PER-SKIP `## style-ok: allow-skip: <reason>` waiver on the `exit 77`
line or the line directly above it. The gate flags an unwaived `exit 77` /
`return 77` in command position (an `exit 77` inside a string or a comment
is not a skip). Unlike the package-wrapper waivers, the reason is
mandatory: `allow-skip` with no rationale still reads as a bare skip.
