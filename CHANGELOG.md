# Changelog

**Every entry names the failure that caused the change.** A line saying what moved is a
diff someone already has; the reason it moved is the part that does not survive anywhere
else, and it tells a reader whether the change still applies to them.

Nothing is listed until it is on `main`. Work in an open pull request belongs in the pull
request. Dates are the day the work landed.

## Unreleased

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
