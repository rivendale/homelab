# Linux packages

Rules for keeping a home server's packages current. Each entry: the rule, the failure behind
it, the version measured, and how to check.

## A release upgrade can silently switch off every third-party source

**After an Ubuntu release upgrade, check that each third-party apt source still has a live
line; the upgrade may have commented it out, and nothing reports it.**

- **Failure (2026-10):** a server upgraded from Ubuntu 24.04 to 26.04 kept running the
  versions it already had of Tailscale, Docker, the Google Cloud SDK and Trivy. The upgrade
  had commented out or renamed (`*.list.disabled`) every source under
  `/etc/apt/sources.list.d/`, leaving the note *"This file could not be automatically
  migrated to a .sources file during the upgrade."* `apt update` and `apt install` both
  succeeded and reported each package as "already the newest version", because apt no
  longer knew a newer one existed. It surfaced only when a security release for one of
  them did not arrive.
- **Measured on:** Ubuntu 26.04 after `do-release-upgrade` from 24.04, observed 2026-10.
- **Check:** `apt-cache policy <package>` lists where a package can come from. A version
  table that shows only `/var/lib/dpkg/status` means no source offers it: the package can
  never update. Also run `ls /etc/apt/sources.list.d/` and look for `.disabled` files or
  files whose only `deb` / `URIs:` line starts with `#`.
- **Fix:** re-add the vendor's line for the new release name, checking first that the
  vendor publishes it (the `dists/<release>/Release` file answers 200). Then `apt update`,
  and confirm `apt-cache policy` now shows the vendor's URL as a candidate.

## "Not upgrading yet due to phasing" is working as intended

**Held-back packages under a phasing notice are a staged rollout, not a fault; security
updates are not phased.**

- **What it is:** Ubuntu releases most non-security updates to a growing share of machines
  over several days and halts the rollout if problems appear. `apt upgrade` reports the
  rest as "Not upgrading yet due to phasing".
- **Measured on:** Ubuntu 26.04, observed 2026-10.
- **Do:** wait. There is an option that skips phasing; on a server, letting other machines
  find a bad update first is the point of it.
- **Check:** `apt-cache policy <package>` shows a phased candidate with its percentage.
