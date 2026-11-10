# Per-daemon W^X hardening (systemd units)

Linux has no system-wide W^X (write-xor-execute) knob by design (JITs need
write-then-execute). security-misc's `vm.memfd_noexec=2` is only a bar-raiser,
not W^X. The idiomatic closure is PER-DAEMON, applied to resident daemons that
never JIT, via systemd unit directives. This is the pattern used by kloak,
sdwdate, privleapd, onion-grater, canary and the other resident daemons.

## When NOT to apply (SKIP + flag)

A daemon that legitimately does write-then-execute MUST NOT get this pattern:
browsers (V8/JSC/SpiderMonkey), Java, Mono/.NET, LuaJIT/PyPy, Wine, and CPython
ONLY if it uses ctypes CALLBACKS (`ctypes.CFUNCTYPE`/`PYFUNCTYPE` -> libffi
closures allocate executable trampolines) or the opt-in 3.13 experimental JIT.
Plain CPython + `dlopen` of `.so` libraries is FINE (library maps are PROT_EXEC
without PROT_WRITE). Classify each daemon; TEST, do not assume.

## The pattern

Layer 1 -- anonymous executable memory (apply to every non-JIT daemon):

    MemoryDenyWriteExecute=true      ## deny anon W+X and mprotect/shmat gaining PROT_EXEC
    SystemCallArchitectures=native   ## close the secondary-ABI (i386 ipc/mmap2) bypass of MDWE
    LockPersonality=true             ## block the READ_IMPLIES_EXEC personality bypass
    SystemCallFilter=~memfd_create   ## kill the anon memfd exec-bypass (or: an allow-list that omits memfd_create)

Why all four: `MemoryDenyWriteExecute` on current kernels is `prctl(PR_SET_MDWE,
PR_MDWE_REFUSE_EXEC_GAIN)` (violation errno EACCES; a seccomp EPERM filter is the
pre-6.3 fallback). Alone it leaves three bypasses: a written `memfd` mapped
`PROT_READ|PROT_EXEC` (closed by `~memfd_create`), the i386 secondary ABI
(`SystemCallArchitectures=native`), and `READ_IMPLIES_EXEC` on the seccomp
fallback (`LockPersonality`).

### memfd_create denial MODE: SIGSYS default vs EPERM (GUI/Wayland daemons)

`SystemCallFilter=~memfd_create` with NO `SystemCallErrorNumber` denies the call
with the systemd DEFAULT, SIGSYS, which HARD-KILLS the process. That is correct
and intended for a LEAF daemon that never calls memfd_create itself (most of
these units).

But a daemon whose OWN code -- or an inherited child (the filter inherits across
fork/execve) -- calls memfd_create is KILLED on that call. Qt (5 and 6) and GTK
Wayland clients allocate their shm draw buffers via libwayland's
`os_create_anonymous_file`, which tries memfd_create FIRST, so a GUI/Wayland
daemon SIGSYS-DIES the instant it maps a surface (a window, or a popup/menu) --
the service may "start" (a QSystemTrayIcon over StatusNotifier D-Bus maps no
surface) yet die on first draw. (Measured: a Qt5 and a Qt6 client, and the real
msgdispatcher QDialog, all die `status=31/SYS`, kernel audit `syscall=319`,
`code=SECCOMP_RET_KILL_PROCESS`, under the default.)

For such a daemon add `SystemCallErrorNumber=EPERM`:

    SystemCallFilter=~memfd_create
    SystemCallErrorNumber=EPERM

memfd_create then returns `-1 EPERM` instead of killing -- the W^X bypass stays
CLOSED IDENTICALLY (the call is still denied; nothing can create an executable
memfd) -- and libwayland/Qt takes its `shm_open`/`mkstemp` fallback and renders.
A SIGSYS *kill* gives the fallback no chance; an EPERM *return* does. This is the
SAME rationale as the escalator EPERM below: the daemon's OWN code legitimately
hits the syscall, so it must fail soft, not die. Keep `/dev/shm` in `NoExecPaths`
so the shm_open fallback buffer lands non-exec (both vectors stay closed).
(Measured: under EPERM the same Qt5/Qt6 clients and the real msgdispatcher render
and exit cleanly, memfd_create still returning EPERM.) Applied to the GUI/Wayland
user units msgcollector-gui, sdwdate-gui, livecheck. Do NOT remove `~memfd_create`
to "fix" a GUI daemon -- that reopens the mmap(PROT_EXEC)-on-memfd bypass MDWE
alone does not cover.

Layer 2 -- file-backed executable mappings (MDWE does NOT cover these: an
already-written file mapped `PROT_READ|PROT_EXEC` has no PROT_WRITE bit):

    ## dropped-privilege daemon, no AppArmor profile:
    NoExecPaths=/run /tmp /var/tmp /dev/shm   ## + the daemon's own writable state dir
    ## a write-nothing daemon can also take, for a near-complete close:
    ProtectSystem=strict
    ProtectHome=yes

For a ROOT daemon also list `/dev` (devtmpfs is rw and NOT noexec, so uid 0 can
drop a file there and execve it) and the daemon's own state dir (e.g.
`/var/lib/<daemon>`); `ProtectSystem=full/strict` does not cover `/dev`, `/var`,
`/opt`, `/srv`. Device NODES stay usable under `NoExecPaths=/dev` (only executing
FROM /dev is blocked). A DROPPED-PRIVILEGE daemon does NOT need `/dev` -- a
non-root uid cannot create files in root-owned `/dev`. GOTCHA: a `+`-prefixed `ExecStartPre=` or `PermissionsStartOnly=true`
makes that control process run as root EXEMPT from the unit sandbox (no MDWE /
SystemCallFilter / NoExecPaths) -- so W^X covers `ExecStart=` only; do not claim
the ExecStartPre helper is covered.

    ## daemon WITH an AppArmor profile: deny executable mapping (`m`) on every
    ## WRITABLE path -- grant `m` only on read-only trusted paths (its binary +
    ## system libs). e.g. sdwdate: `/tmp/* rw,` not `/tmp/* mrw,`.

Root daemon that spawns privilege-gaining children (an escalator, e.g.
privleapd):

    NoNewPrivileges=false   ## explicit; the MDWE/SystemCallFilter seccomp still
                            ## installs via the root executor's CAP_SYS_ADMIN, and
                            ## stays off so an escalated SUID helper still works.

All of MDWE / SystemCallArchitectures / LockPersonality / SystemCallFilter /
NoExecPaths are a seccomp filter or a mount namespace: they INHERIT across
fork/execve into the daemon's children. For an escalator this is by design (W^X
covers the escalated actions too) but means a 32-bit or READ_IMPLIES_EXEC child
would be refused -- document that caveat in the unit.

## Completeness (be honest in the unit comment)

- DROPPED-PRIVILEGE daemon: COMPLETE W^X. Its writable set is exactly the
  NoExecPaths dirs (ProtectSystem + non-root keep it there), or its AppArmor
  profile denies `m` on all writable paths.
- ROOT daemon: Layer 1 (anonymous) is COMPLETE; Layer 2 (file-backed) is PARTIAL
  -- NoExecPaths closes the common scratch dirs, but uid 0 can still exec a file
  written on another executable filesystem, or clear noexec via `mount_setattr`.
  A COMPLETE file-backed close for a root process needs MAC (a dedicated AppArmor
  profile / LSM), NOT systemd unit settings alone. State this residual in the
  unit comment; do not claim a complete close. For a SIMPLE write-nothing root
  daemon, `ProtectSystem=strict` + `ProtectHome` shrink the residual to near
  nothing; for an arbitrary-action escalator, leave it at Layer 1 + the scratch
  dirs + a documented residual (ProtectSystem would break its actions).

## NoNewPrivileges=yes on non-escalators (applied)

A non-escalator hardened daemon ALSO sets `NoNewPrivileges=yes` for
defense-in-depth. SEPARATE axis from W^X -- W^X does not need it -- and whether
it changes anything depends on the unit regime (verified on systemd 257.13, live
`/proc/<pid>/status`):

- SYSTEM units (plain root, and `User=` system units): the root executor installs
  the unit seccomp WHILE STILL ROOT (`CAP_SYS_ADMIN`), BEFORE the `User=` uid
  drop, so `no_new_privs` is never required and the W^X directives do NOT imply it
  -- a W^X-only system unit runs `NoNewPrivs=0` (contrary to the common reading of
  `context_has_no_new_privileges()`, which reviewers repeatedly flag; it does not
  match the 257.13 executor sequence, and a real `User=sdwdate` installed unit
  confirms `NoNewPrivs=0`). Here `NoNewPrivileges=yes` is a REAL tightening:
  verified `NoNewPrivs` 0 -> 1 with the directive, 0 without.
- USER-SESSION (`systemd --user`) units: NO root executor, so systemd MUST set
  `no_new_privs=1` to install any unprivileged seccomp. Each of `SystemCallFilter`,
  `SystemCallArchitectures`, and `MemoryDenyWriteExecute` (prctl) ALONE forces
  `NoNewPrivs=1` on a user unit (only `NoExecPaths` alone leaves it 0). So a
  W^X-hardened user unit ALREADY runs `NoNewPrivs=1`; the explicit
  `NoNewPrivileges=yes` only documents and pins that guarantee (no behavior
  change). MDWE itself is `PR_SET_MDWE` (prctl), not seccomp.

Under `NoNewPrivs=1` a setuid exec cannot gain privilege (`sudo` aborts: "the
'no new privileges' flag is set"; `pkexec` likewise). `NoNewPrivileges=yes` is
therefore added to every AUDITED non-escalator -- privileged work via `leaprun`
(a non-setuid privleapd socket client) or `systemctl` (D-Bus to PID1) is not an
in-process gain -- and the socket-proxyd units get it from their generator
template (`anon-ws-disable-stacked-tor/.../systemd-unit-files-generator`), not by
hand-editing the ~20 autogen files.

EXCLUDED (no `NoNewPrivileges=yes`; each gains privilege):
- `privleapd` -- escalates admin-defined root actions (`shim.py` -> Popen). A
  SYSTEM escalator: W^X via the root executor keeps `NoNewPrivs=0`, so escalation
  still works; the unit sets `NoNewPrivileges=false` explicitly.
- `kloak` -- execs across an AppArmor non-subset `Px` (`/usr/bin/kloakwl rPx ->
  ...`); `NoNewPrivileges=yes` would break that launch. SYSTEM unit, explicit
  `NoNewPrivileges=false`.
- `systemcheck` -- a `systemd --user` escalator: its main flow execs setuid
  `sudo` / `pkexec` (`check_privilege_escalation_tool.bsh:72,78`). CAVEAT: being a
  USER unit, its W^X seccomp ALREADY forces `NoNewPrivs=1`, so those checks are
  ALREADY refused in the autostart-service context -- omitting `NoNewPrivileges=
  yes` does NOT restore them. A `systemd --user` escalator is thus inherently
  incompatible with W^X seccomp. RESOLVED (PR #18): the escalation self-check is
  now `no_new_privs`-aware -- under `NoNewPrivs=1` it reports the setuid
  `sudo`/`pkexec` checks as `N/A (no_new_privs)` instead of false-failing, so the
  W^X hardening stays on the unit; the leaprun checks (non-setuid socket client)
  are unaffected and still run, and an interactive `systemcheck` run
  (`NoNewPrivs=0`) still exercises sudo/pkexec normally. The unit itself stays
  excluded from an explicit `NoNewPrivileges=yes` (it is an escalator).

## Verify empirically (do not theorize)

Per daemon: it still reaches active/running and performs its real function with
NO SIGSYS/seccomp denial in the journal (`/proc/<pid>/status` shows `Seccomp: 2`),
AND a probe proves closure -- `mprotect(+PROT_EXEC)` / `mmap(W|X)` -> EACCES,
`memfd_create` -> EPERM/SIGSYS, and a file-backed `mmap(PROT_READ|PROT_EXEC)` in
a NoExecPaths dir -> EACCES/EPERM. An escalator additionally: a real action
escalates+runs under the hardened unit. A GUI/Wayland daemon additionally: prove
it RENDERS -- actually maps a window or popup/menu SURFACE -- not merely that it
reaches active; a tray icon maps no surface, so a daemon can "start" green yet
SIGSYS-die on first draw (see the EPERM subsection above).
