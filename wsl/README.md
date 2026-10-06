# WSL2

Running always-on services inside WSL2 on a Windows desktop. WSL is a real Linux kernel in a
lightweight VM, and most of what goes wrong comes from forgetting that the VM, its network and
its lifetime all belong to Windows.

Measured on Windows 11 with WSL2, an Ubuntu distro and systemd enabled as PID 1, 2026-08 and
2026-09.

## Mirrored networking destroys every socket at once

**Under `networkingMode=mirrored`, a state change on any one Windows adapter can destroy every
established TCP socket in WSL, including sockets on a different, healthy adapter.**

- **Failure (2026-08, observed):** a Wi-Fi radio kept re-associating on a desktop that was
  also wired. The wired link never dropped, yet WSL lost all of its sockets four times in 28
  minutes. Hyper-V's virtual switch logged bursts of NIC delete and create events each time.
  A health check that treated "zero established sockets" as proof of death restarted a
  healthy service four times.
- **Tells:** nothing inside WSL logs it (`dmesg` shows no link changes). A high interface
  index on a machine with few adapters is the fingerprint of repeated churn. The cause is only
  in the Windows System event log, under the Wi-Fi driver and
  `Microsoft-Windows-Hyper-V-VmSwitch`.
- **Check:** `ip -o link` inside WSL and compare the interface indexes with the adapter count.
  On Windows, look for VmSwitch events in the System log around the time sockets died.
- **Fix that held:** disable the redundant adapter and set Windows to turn it back on only
  manually, so it cannot silently return. Any WSL probe that counts sockets needs two
  consecutive zeros, not one.

## A closed port on 127.0.0.1 hangs instead of refusing

**With `networkingMode=mirrored` and `firewall=true`, a TCP connect to a closed port on
`127.0.0.1` gets no reply, so it waits out the full timeout. The same connect to `::1` is
refused at once.**

- **Measured (2026-10-06):** a connect to `127.0.0.1:9` and `127.0.0.1:1` timed out at 25 s and 5 s
  (two runs, each at its configured timeout); the same to `[::1]:9` raised "connection refused" in 0.0 s. A test that pointed a
  client at a closed local port to prove "unreachable is an error" took 20 s per run, a quarter of
  its whole suite's time, because the client waited out its own 20 s timeout.
- **Cause, narrowed but not separated:** mirrored mode and the Hyper-V firewall were both on, on two
  machines with the same result. The same four connects made from the Windows host are refused
  in about 2 s, so the drop is on the WSL side of a mirrored-mode IPv4 loopback connect. Telling
  mirrored mode from the firewall needs a WSL restart with one of them off; until then, assume
  either can cause it.
- **Do instead:** to test "the service is unreachable", point the client at a listener that accepts
  and closes at once, or at `::1`, rather than at a closed IPv4 port. Give every client a short
  connect timeout. Do not read a slow failure as a hung tool: try the same connect against `::1`
  first.

## WSL resolves names the way Windows does

**WSL forwards DNS to Windows, hosts file included, so a name that resolves inside WSL may
resolve on exactly one machine in the house.**

- **Failure (2026-08):** another machine reported that a set of internal service names did
  not resolve anywhere. Checked from WSL, they resolved and answered HTTP 200, and a
  contradiction nearly went back with measurements attached. The desktop's Windows hosts file
  held a block of manual entries; every real resolver returned NXDOMAIN.
- **Check:** to test whether your network resolves a name, query a resolver by address:
  `dig @<router-address> <name>` and `dig @<tailnet-resolver> <name>`. `getent`, `curl` and
  bare `dig` from WSL answer "can this desktop reach it", a narrower question.
- **The wider rule:** WSL is not a neutral vantage point. Resolution, proxies, certificates,
  clock and drive letters measured there describe that one desktop.

## A check that passes from home can fail from CI

**A request that works from a residential address can be refused from a cloud runner, so a
check proven only at home has been proven in the one place it cannot fail.**

- **Failure (2026-08, observed):** a publish check fetched a live URL behind Cloudflare. It
  passed from home in both directions (real build found, fabricated one rejected) and failed
  every time in GitHub Actions, where Cloudflare answered HTTP 403 with a managed challenge.
  The first version printed only "did not answer", which merged a network refusal and an
  undeployed build into one line.
- **Check:** print the status code and response headers on failure. A `cf-mitigated:
  challenge` header names the control that refused you. Run the check from wherever it will
  run for real before trusting it.

## PID 1 inside WSL can be killed by the OOM killer

**WSL runs the distro in a nested PID namespace, so its PID 1 is not the kernel's global init
and is not exempt from the OOM killer, unlike PID 1 on bare metal.**

- **Failure (2026-08):** a memory cap on the cgroup holding PID 1 was dismissed as safe
  because "the kernel never kills init". That holds on bare metal and was false on the WSL host
  it was applied to. That cgroup (`init.scope`) also holds WSL's init shims and the
  9P server that backs `/mnt/c`.
- **Check:** `readlink /proc/self/ns/pid`. The initial namespace is `pid:[4026531836]`; any
  other number means nested. Read `/proc/self`, not `/proc/1`: as a normal user
  `readlink /proc/1/ns/pid` returns empty without an error.
- **Mitigation:** a boot-time unit that sets `oom_score_adj=-1000` on those processes. Match
  them by command line, never by name: the 9P server's `comm` is literally `init`, so a
  name-keyed guard protects nothing while returning success.

## The distro lives only while a client is attached

**A WSL distro stays running only while a Windows-side client holds it; systemd as PID 1 and
`loginctl enable-linger` do not keep it alive.**

- **Failure (2026-08):** closing the last terminal shut the distro down about 15 seconds later,
  taking every service and every watchdog inside it. 27 minutes dark with zero notifications,
  because every alert path lived inside the VM that died. Windows itself never rebooted.
- **Second failure (2026-08):** a logon task ran `wsl.exe -e /bin/true` to start the distro at
  boot. `/bin/true` exits at once, so the distro started, services came up, and WSL tore it
  down seconds later. The task reported Last Result 0 the whole time.
- **Fix:** make the logon task the held client:
  `wsl.exe -d <distro> -u root -e /usr/bin/sleep infinity`, with no execution time limit and
  "do not start a new instance". Wrap it in
  `powershell.exe -NoProfile -WindowStyle Hidden -Command "..."` to hide the console window;
  PowerShell blocks, so the task still reads Running. Changing the task needs an elevated
  shell.
- **Check:** after a reboot, compare `uptime -s` inside WSL with the Windows boot time. More
  than a minute apart means the distro was rebuilt, whatever the task reported. The task's
  healthy Last Result is `267009` (`0x41301`, "currently running"); `0` means the holder
  exited and the distro is about to go. The polarity is the reverse of an ordinary exit code.
- **Diagnose a teardown** with `journalctl -b -1` (the previous boot's journal is kept if
  `/var/log/journal` exists), `journalctl --list-boots`, and the Windows System log. Three
  commands settle "the VM died" versus "Windows rebooted".

## Nothing in `/tmp` survives a Windows restart

**A Windows restart tears down the distro and `/tmp` comes back empty, so anything that
authorizes a later destructive step must be written somewhere durable the moment it exists.**

- **Failure (2026-09):** an overnight Windows update rebooted the desktop and erased a day of
  analysis in `/tmp`: per-file hash lists proving which files had verified copies elsewhere.
  The summary survived because it had been saved; the row-level evidence did not, and only the
  rows could authorize deleting a single file. The whole hashing run had to be repeated.
- **Check:** `findmnt /tmp`. If it is tmpfs or on the VM's own disk, treat it as gone at the
  next restart. Write evidence to a path under your home directory or a repository as it is
  produced, not at the end.

## Call Windows binaries by full path from WSL

**`command -v cmd.exe` returning nothing does not mean interop is off; Windows directories may
simply be absent from `PATH`.**

- **Failure (2026-09):** interop was reported disabled, and a fix declared impossible, because
  `command -v cmd.exe` printed nothing. Interop was enabled; `PATH` did not include
  `/mnt/c/Windows/System32`.
- **Check:** `ls /proc/sys/fs/binfmt_misc/ | grep -i wsl` and then run the binary by full path,
  for example `/mnt/c/Windows/System32/icacls.exe`. Test the capability, not the shortcut.
- **Two more traps from the same session:** do not wrap a command in `cmd.exe /c` from a WSL
  working directory. The directory is a `\\wsl.localhost\...` UNC path, cmd warns that UNC paths
  are unsupported, and then resolves arguments wrongly. Call the `.exe` directly. And Windows
  binaries read stdin: inside `while read -r f; do ...; done < list`, one call consumed the rest
  of the list and the loop reported one item. Give every Windows binary in a read loop
  `</dev/null`.
