# Changelog

**Every entry names the failure that caused the change.** A line saying what moved is a
diff someone already has; the reason it moved is the part that does not survive anywhere
else, and it tells a reader whether the change still applies to them.

Nothing is listed until it is on `main`. Work in an open pull request belongs in the pull
request. Dates are the day the work landed.

## Unreleased

- **A metrics server stayed down after two reboots because its documented config reload,
  `docker kill --signal=HUP`, had marked it stopped by hand, and `unless-stopped` skips such a
  container.** New section [Docker](docker/README.md) with a four-container check you can run
  without rebooting, and the `docker exec <container> kill -HUP 1` reload that avoids it.
- **A release upgrade had silently commented out every third-party apt source, so four
  packages stopped updating while `apt` reported each as the newest version.** New section
  [Linux packages](packages/README.md) with the check (`apt-cache policy` showing only
  `/var/lib/dpkg/status`), plus a note that phased holds are a staged rollout, not a fault.
- **A reader who landed here found the sibling repos only at the bottom of the README, and
  nothing said which of them to start from.** `README.md` and `AGENTS.md` now say under the
  title that this is one of a set of repos maintained separately and that
  [hsi-operator](https://github.com/rivendale/hsi-operator) is where to start. The hub's CI
  checks that this pointer stays within the first 12 lines.

- **A tool trial could take the host down, a global npm install needed root, and a timer that
  commits a working tree would have published a new tool's index.** `systemd/` gains capping
  heavy installs and trials with `MemoryMax` plus `MemorySwapMax=0` (under the existing long-jobs
  entry) and installing user CLIs under a per-tool npm prefix; `sync-and-backup/` gains excluding
  a tool's cache with `.git/info/exclude` before an auto-commit timer runs. Indexed in `traps.md`.

- **An agent service could reach its model through a third-party router.** A published study
  of 428 LLM routers found 9 injecting malicious code and 17 touching planted canary
  credentials; `networking/` now says to talk only to the provider's endpoint or your own
  gateway, with a names-only check for base-URL and proxy overrides in the service's real
  environment, and links the agent-side rule in hsi-operator. Indexed in `traps.md`.

- **About forty lessons from two months of running a small home lab sat in one operator's
  private notes, where nobody else could use them.** Each came from a real failure: every
  socket in WSL dying because an unrelated Wi-Fi adapter flapped, a distro torn down fifteen
  seconds after its last terminal closed while a logon task reported success, a memory cap
  that froze a service for hours with every probe green, a Tailscale tag that took every
  node off the tailnet at once, a DHCP lease expiry that took a server offline to the
  minute, a Syncthing password made of whitespace, a version store that a local delete can
  never fill, a push notification over 4 KB delivered as an attachment nobody read, and
  files that every permission tool said were readable while every open failed. They are now
  in `wsl/`, `systemd/`, `networking/`, `sync-and-backup/`, `alerting/` and `windows/`, with
  an index in `traps.md`. Rules already published in the opensource lessons (identical
  alerts, `for:` longer than the outage, `curl -f`, comparing against a shared ref) are
  linked, not repeated.
- **The repository was empty, so a pull request had nothing to target.** An MIT license and
  a placeholder README went to `main` as the only direct commit.
