# homelab

Practice from running a small home lab: a Windows desktop with WSL2, a small Linux server,
laptops, on one tailnet, with systemd user services, Prometheus alerting, push notifications
and file sync between the machines.

**Every entry names the failure behind it.** A rule without the failure that produced it
cannot tell you whether it still applies to your setup; the failure can. Each entry gives
the rule in one bold line, what went wrong (dated to the month), the product or version it
was measured on, and how to check your own machines. Anything not re-verified since it was
first seen is marked **observed** with its date.

## Who this is for

Anyone running a few machines at home who has had a service report healthy while doing
nothing. Most of what follows is one shape: a check that answered a narrower question than
the one being asked, and read green while the thing it guarded was broken. The specific
products (WSL, Tailscale, Syncthing, ntfy, Prometheus) are where it happened, not the
point.

It is not a setup guide. It assumes you already run these tools and want to know where
they lie to you.

## Contents

| page | covers |
|---|---|
| [wsl/](wsl/README.md) | mirrored networking, borrowed DNS, PID 1 and the OOM killer, keeping a distro alive, what `/tmp` survives, calling Windows binaries |
| [systemd/](systemd/README.md) | user units, oneshot state, memory limits that hang instead of kill, parent slice caps, long jobs, services that die silently |
| [networking/](networking/README.md) | Tailscale tags and ACLs, the two SSH doors, DHCP reservations and statics, names versus addresses, testing DNS |
| [sync-and-backup/](sync-and-backup/README.md) | Syncthing versioning, whitespace config values, live versus readable config, Windows service accounts, keeping evidence durable |
| [alerting/](alerting/README.md) | ntfy's 4 KB limit, label matchers, suppression, watchdogs that die with their host, volatile signals |
| [windows/](windows/README.md) | detecting a pending reboot, EFS encryption that ACL tools cannot see, encrypted directories |
| [traps.md](traps.md) | one-line index of every entry |

[AGENTS.md](AGENTS.md) is the working agreement for anyone, human or agent, adding to this
repo. [CHANGELOG.md](CHANGELOG.md) records each change and the failure that caused it.

## Sibling repos

- [rivendale/hsi-operator](https://github.com/rivendale/hsi-operator): operating AI coding
  agents; the Claude Code specifics of background jobs live in its
  [docs/tools.md](https://github.com/rivendale/hsi-operator/blob/main/docs/tools.md).
- [rivendale/local-ai](https://github.com/rivendale/local-ai): running private AI on your
  own hardware.
- [rivendale/opensource](https://github.com/rivendale/opensource): a watch list of tools,
  and [lessons](https://github.com/rivendale/opensource/blob/main/lessons/README.md) that
  already cover several monitoring rules this repo links to rather than repeats.

## License

[MIT](LICENSE).
