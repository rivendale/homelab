# systemd

Running home-lab services as systemd units, mostly on the user manager
(`systemctl --user`) with lingering enabled so they run without a login session.

Measured on Ubuntu with systemd and cgroup v2, on a small server and inside WSL2, 2026-08 and
2026-09.

## Ask the bus the unit actually lives on

**`systemctl is-active <unit>` asks the system bus; for a user unit it answers `inactive`
while the service runs normally.**

- **Failure (2026-08):** two healthy services on two hosts read `inactive` on every check,
  because both were user units. One of them was not named what anyone would guess, so even the
  right bus needed the right name.
- **Check:** `systemctl --user is-active <unit>`, or skip the unit layer and test the thing
  itself: an end-to-end request, or the process inside the unit's cgroup. `systemctl --user
  list-units --type=service` finds the real name.

## A oneshot reads activating while it runs

**A `Type=oneshot` unit reports `activating`, not `active`, while it runs, so waiting until it
is not active returns immediately.**

- **Failure (2026-09):** `until ! systemctl --user is-active -q <unit>` exited at once, and the
  `Result=success` read at that moment belonged to the previous run.
- **Check:** wait on `systemctl --user show -p ActiveState <unit>` reaching `inactive` or
  `failed`, then read `Result` and `ExecMainStatus`.
- **`systemd-run -p Type=oneshot` blocks until the unit finishes** unless you background it or
  pass `--no-block`, so a poll started after it returns sees nothing left to wait for.

## The user manager has its own PATH

**The user manager's `PATH` does not include `~/.local/bin`, so a unit that calls a tool
installed there fails at start.**

- **Failure (2026-09):** a new unit calling a CLI in `~/.local/bin` died at admission. Its
  `OnFailure=` named a template unit that had never been created, so the first failure alerted
  nobody.
- **Check:** `systemctl --user show-environment | grep PATH`. Set
  `Environment=PATH=%h/.local/bin:/usr/local/bin:/usr/bin:/bin` in the unit. For every
  `OnFailure=`, confirm the target exists with `systemctl --user cat <template>@.service`, and
  make it fire once on purpose.

## MemoryCurrent counts page cache

**A cgroup's `MemoryCurrent` includes reclaimable page cache, so comparing it with
`MemoryHigh` both raises false alarms and hides real growth.**

- **Failure (2026-08):** a monitor said a service sat at 91% of its `MemoryHigh` and had failed
  on it for days: 100 alerts in three days, 95 of them this bug. The process held 15%; the rest
  was page cache the kernel reclaims on demand. And a real runaway would have been missed until
  it pushed the total past the threshold, because cache was already using the budget.
- **Check:** read `anon` from the cgroup's `memory.stat`
  (`/sys/fs/cgroup/<path>/memory.stat`). Anon cannot be reclaimed and is what throttles the
  process. Any check comparing `MemoryCurrent` with a limit has this bug.

## MemoryHigh turns a crash into a hang

**`MemoryHigh` throttles instead of killing, so a runaway child freezes the whole unit while
every liveness probe reads green.**

- **Failure (2026-08 and 2026-09):** a child process grew past the unit's `MemoryHigh`. The
  cgroup logged hundreds of thousands to millions of `high` throttle events, `oom_kill` stayed
  0, and the service sat `active (running)` for hours doing nothing. `Restart=always` cannot
  fire because nothing exits. The second time, an hour went on waiting for recovery before
  anyone looked at what held the memory.
- **Check:** when a service "goes offline" but systemd says running, read `memory.events`
  (`high` climbing, `oom_kill` 0 is the signature) and `memory.pressure` first. Then find the
  pid: RSS for each pid in `cgroup.procs`, and `memory.stat` anon against file. Kill that one
  child and verify by its absence and the cgroup's usage dropping, not by restarting the unit.
  A child holding anon memory will not shrink by waiting.

## The cap that binds may belong to a parent

**A child cgroup cannot exceed its parent, so the limit stalling a service may be on a slice
above it, and raising the unit's own cap changes nothing.**

- **Failure (2026-09):** a service stalled at its 8 GB `MemoryHigh`. The cap was raised to
  11 GB and nothing changed: both `app.slice` and `user-<uid>.slice` above it were at their own
  limits.
- **Check:** for every ancestor from the unit up to the root, read `memory.high`, `memory.max`
  and `memory.current`, and name the tightest one that is at its limit.
- **Related (2026-09):** per-unit caps that add up to more than physical RAM do not protect the
  host. Each unit just hangs at its own limit while the host thrashes. What bounds it is one
  shared parent slice for the heavy services, sized well under RAM, plus a limit on how much
  work each one fans out.

## High load with an idle CPU means paging

**A load average of 60 with the CPU mostly idle is processes blocked on I/O, usually swap, not
a busy machine.**

- **Failure (2026-09):** a small server read load 67 with 59% idle CPU. Fourteen processes sat
  in uninterruptible sleep, `vmstat` showed thousands of pages swapped in and out per second,
  and one service's cgroup held 44 processes from a parallel test run. The service's network
  heartbeats were the first thing to fail.
- **Check:** `ps -eo stat | grep -c ^D` and `vmstat 1 5` before concluding anything from load.
  Judge "is it still working" by output written over a window, not by process count or CPU; a
  thrashing job produces nothing while looking maximally busy. Before restarting, check
  `KillMode=control-group` so the restart reaps the children rather than orphaning them.

## Run long jobs in their own transient unit

**A job that must outlive the shell or service that started it belongs in its own cgroup:
`systemd-run --user --collect`.**

- **Why:** `setsid` and `nohup` leave the process group but stay in the parent service's
  cgroup. With `KillMode=control-group`, restarting that service takes the job with it.
- **How:** `systemd-run --user --unit=<name> --collect -p WorkingDirectory=<dir>
  -p StandardOutput=append:<log> <command>`.
- **Check:** `cat /proc/<pid>/cgroup` must name the transient unit, not the service that
  launched it. Poll `systemctl --user is-active <name>` for completion, and keep the job's
  resume state on a durable path, not `/tmp`.
- **For AI coding agents:** whether a harness reaps its own background tasks at the end of a
  turn depends on the harness and its version. The measured details are in
  [hsi-operator docs/tools.md](https://github.com/rivendale/hsi-operator/blob/main/docs/tools.md).

## A wrapper's exit is not the work finishing

**A wrapper that backgrounds the real command exits 0 in seconds while the work is still
starting, and a waiter keyed on the success artifact cannot tell "still running" from "never
started".**

- **Failure (2026-09):** a backgrounded `nohup cmd &` reported completion with exit 0 while
  the real job ran on for minutes. In the opposite case, a job died at admission in its first
  second, wrote nothing, and a waiter watched nine minutes for an output file that would never
  come.
- **Check:** wait on the unit's own state and read its exit code; read the journal as soon as
  it goes inactive. A completion arriving in seconds for work that takes minutes is the tell.

## StandardOutput file does not truncate

**`StandardOutput=file:<path>` overwrites from the start without truncating, so a previous
run's completion marker can still be in the log.**

- **Failure (2026-09):** a waiter grepping a log for `DONE` fired at once on the second run,
  matching the first run's marker, while the real job was still working.
- **Check:** use `StandardOutput=truncate:<path>` for a fresh log each run, or wait on the
  unit's state instead of a marker in a file.

## A process can survive its own death

**A service whose network transport has died but whose process has not will read
`active (running)` forever, and `Restart=always` never fires.**

- **Failure (2026-08):** a long-lived client lost its connection when DNS failed, exhausted its
  own retries, and sat idle for good. The process was fine, systemd was green, and the network
  came back 26 minutes after the client stopped trying. Nothing retries after exhaustion.
- **Check:** decide what "working" means from the consuming end (a heartbeat record, a message
  that arrives) and alert on that. When the process cannot tell it is dead, a watchdog that
  exits or restarts it on that signal is the recovery.

## A prompt on discarded stdout blocks silently

**A service that stops at an interactive prompt waits forever with an empty journal, because
the prompt went to the stdout the unit discards.**

- **Failure (2026-09):** a CLI started as a user unit asked "do you trust this folder?" on first
  run in a new directory. `StandardOutput=null` threw the question away; the unit read active,
  the journal held two lines, and the service never connected.
- **Check:** run the unit's exact `ExecStart` in the foreground (under `script` if it needs a
  terminal) and read what it prints. Accept first-run prompts before handing a command to
  systemd.
- **Related:** wrapping a command in `script -qec "<cmd>" /dev/null` to give it a terminal puts
  both its stdout and stderr on the terminal, which `script` writes to its own stdout. With
  `StandardOutput=null`, `StandardError=journal` then captures only `script`'s own errors: three
  days of journal held nothing from the real program. Redirect the program's stderr to another
  descriptor inside the wrapper if you need its errors.

## Find a process without matching yourself

**`pgrep -f <pattern>` matches the shell running the check, because the pattern is in that
shell's own command line.**

- **Failure (2026-09):** a socket-count probe resolved its target with
  `pgrep -f '<pattern>' | tail -1`. It returned five pids, including two short-lived probe
  shells, and picked one that had already exited. A dead pid owns zero sockets, so the probe
  read a healthy service as dead, nondeterministically. The `[m]yservice` bracket trick does
  not help when the pattern is passed as a parameter.
- **Check:** resolve the pid inside the unit's own cgroup and gate on `comm`:
  read `/sys/fs/cgroup/.../<unit>.service/cgroup.procs` and keep the pid whose
  `/proc/<pid>/comm` is the program's name. `MainPID` may be a wrapper (`script`, a shell) that
  owns nothing.
- **Also:** `kill -0 <pid>` is not a liveness test across users. On another user's process it
  fails with EPERM, which a shell test reports exactly like "no such process". Use `/proc/<pid>`
  presence or `pgrep` instead.

## Decide what unattended upgrades may restart

**Unattended package upgrades on a server should be a decision you wrote down, not the
distribution's default: an upgrade that restarts the container runtime at an odd hour is a
silent outage.**

- **The failure shape (2026-09):** a scheduled upgrade replaces the container runtime overnight
  and restarts it, and every container on the host goes down with it. Nothing alerts, because
  nothing failed: the upgrade exits 0, and whether each container comes back depends on its
  restart policy. Written as a pattern to check for, not a measured incident on a named
  version. Distributions such as Ubuntu enable `unattended-upgrades` by default for
  security updates, and a runtime or kernel package can arrive through that channel.
- **Decide per host:** security updates only, at a time you would notice; packages that restart
  shared services (the container runtime, the tailnet daemon, the database) held back and
  upgraded by hand; and whether a pending kernel may reboot the machine on its own.
- **Check:** `systemctl list-timers 'apt-daily*'` shows when upgrades run;
  `apt-config dump | grep -i unattended` shows what is enabled, the allowed origins, the
  package blocklist and `Automatic-Reboot`; `grep -h ' upgrade ' /var/log/dpkg.log*` shows what
  was actually upgraded and when. Compare those times with container restarts
  (`docker ps --format '{{.Names}} {{.Status}}'`, or `docker inspect` for `StartedAt`). An
  uptime that resets at the same hour as an upgrade is this trap.
