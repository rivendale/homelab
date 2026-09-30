# Networking

Tailscale, SSH, DHCP and DNS on a home network: one consumer ISP gateway, a few machines on
DHCP, and a tailnet joining them.

## A Tailscale tag replaces the user identity

**A Tailscale device is owned by a user or by tags, never both, so applying a tag removes the
identity every existing rule was written against.**

- **Failure (2026-08):** the policy allowed traffic from a user to the same user. A tag was
  applied to most nodes in the admin console, and at once no rule matched: a tagged
  device is not that user, and it is not in `autogroup:member`, `autogroup:owner` or
  `autogroup:admin` either, because those describe user accounts. Every node went dark from
  every other. It happened twice in one day.
- **Measured on:** Tailscale 1.102, 2026-08, console-assigned tags.
- **Rules that would have prevented both outages:**
  1. Write the grant for the tag before applying the tag. The step that removes the old path
     goes last.
  2. Reachability and SSH are separate policy sections, and each needs the tag. With only the
     grants fixed, ping works, port 22 is open, and SSH is refused with "tailnet policy does
     not permit you to SSH to this node".
  3. In SSH rules, once a rule's `src` contains a tag, its `dst` must be tags only; Tailscale's
     docs say tagged devices can only SSH into other tagged devices. Expect one validation
     error per user-flavored SSH entry until they are all gone.
  4. Never put a server on a low-privilege tag meant for devices that must not receive SSH.
  5. Swapping tags is safe. Removing the last tag forced a re-authentication, and whoever
     re-authenticated became the owner (observed 2026-08, Tailscale 1.102, admin console).
     Tailscale's tags page says you cannot remove all tags from a device, so what was observed
     may be how that restriction surfaced. Add the new tag and remove the old in the same save.
- **Check:** `tailscale status` from each node after any policy change; a drop in the number of
  visible peers is the symptom. A console-assigned tag cannot be removed from the local CLI, so
  keep a way in that does not depend on the tailnet (see the next entry).

## Disable key expiry on servers you own

**Tailscale node keys expire by default, and on a server that is a scheduled outage rather than
credential rotation.**

- **Failure (2026-08):** a node key expired, the host fell off the tailnet, and a long-lived
  client on it gave up reconnecting before anyone re-authenticated.
- **Measured on:** Tailscale 1.102; disabling expiry was a console action with no CLI path.
- **Applies to user-owned devices.** Tagged devices have key expiry disabled by default, so a
  tagged server already has no expiry. The expiry setting can also be toggled through the
  Tailscale API, not only the console.
- **Check:** `tailscale status --json` and look for `KeyExpiry` on each server. Keep expiry on
  phones and laptops, which can be lost.

## Tailnet SSH and sshd are two doors

**Tailscale SSH is served by `tailscaled` and never reaches `sshd`; `sshd` is a separate door on
port 22. A control on one does nothing to the other.**

| door | handled by | governed by |
|---|---|---|
| tailnet | `tailscaled` | the Tailscale policy's SSH rules only |
| everything else | `sshd` on port 22, all interfaces | `sshd_config` only |

- **Failure (2026-08):** a session was about to harden `sshd_config` believing it would close
  logins over the tailnet. It would not have, and "we restricted sshd" would have been
  recorded as "we restricted tailnet logins". Listing tailnet addresses in `sshd_config`
  restricts nothing, because those connections never arrive there.
- **Measured on:** Tailscale 1.102 and OpenSSH 9.6, 2026-08; re-measured 2026-09.
- **Check which door served a session:** `ps -o comm,args -p $PPID`. If it shows
  `tailscaled be-child ssh`, sshd never saw the connection. `tailscale debug prefs` shows
  `"RunSSH": true` when the feature is on.
- **A host cannot test the tailnet door against itself.** SSH to a machine's own tailnet
  address from that machine is answered by sshd. Probe door one from another node, or you
  silently measure door two (observed 2026-08).
- **Keep door two as break-glass,** keys only and no root. When `tailscaled` broke, sshd over
  the LAN was the only way in. Test the break-glass key itself: tailnet SSH never checks keys,
  so a stale `IdentityFile` in your SSH config stays invisible until the day you need it
  (observed 2026-09).

## Some consumer gateways only accept a reservation inside the DHCP pool

**Some consumer gateways only accept a reservation inside the DHCP pool; a static address goes
outside it on every server.**

- **This is firmware, not DHCP (observed 2026-08, one consumer ISP gateway).** dnsmasq and ISC
  dhcpd do not require it. dnsmasq's man page says reservation addresses "are not constrained to
  be in the range given by the --dhcp-range option", only to be in the same subnet as some valid
  range, and ISC dhcpd host declarations behave the same way.
- **Failure (2026-08):** hours went into a gateway's reservation dialog, which silently reverted
  an entry three times even with the correct MAC and an in-pool address. The requirement itself
  surfaced only when an out-of-pool address was tried: "Reserved IP Address is not in valid
  range". A reservation is the wrong tool when what you want is an address the gateway never
  issues.
- **The better option:** shrink the pool and assign statics below it. An out-of-pool address is
  one the gateway cannot hand to anything else; a reservation is a promise it keeps. Some
  gateways show the pool only behind an edit button, not on the summary page.
- **Check:** read the pool range from the gateway, then `ip -4 addr` and
  `ip route show default` on each server. `proto dhcp` on the default route means the host is
  still asking.

## A DHCP lease is a fuse

**A host on DHCP whose renewal breaks keeps working until the lease runs out, so the outage
arrives days after its cause.**

- **Failure (2026-08):** a server's network manager marked its interface failed and stopped
  renewing. The lease it had just taken ran its full two days, and the server dropped off the
  LAN to the minute. A planned move to static addresses stalled and nothing recorded that it
  had; the plan read as the current layout for days.
- **Do not just lengthen the lease.** That lengthens the fuse and makes the cause harder to
  connect to the outage.
- **Applying a static remotely:** use `netplan try`, never `netplan apply`; it reverts after
  about two minutes unless confirmed, so a wrong gateway undoes itself. Set DNS servers
  explicitly, because DHCP was supplying them. Leave IPv6 alone.
- **Check egress, not reachability:** `ip route show default` and a request to a public address
  from the host itself. Tailnet reachability stayed green through the whole outage.

## The tailnet name is the identity

**Address a host by its tailnet name, not its LAN address; on DHCP, every LAN address written
down has an expiry nothing will announce.**

- **Why:** when a host on DHCP becomes unreachable, "it moved" and "it is down" look identical.
  A LAN address in a document is a claim about one moment.
- **Check:** use `<host>.<tailnet-name>` (MagicDNS) in scripts and docs. If you must record a
  LAN address, write the date it was measured next to it.
- **Behind a reverse proxy:** a proxy that routes on the `Host` header needs the right name
  even when the address is right. Without the header you reach a different virtual host, and a
  200 from the wrong one looks like success. `curl -H 'Host: <service-name>' http://<host>/`.

## Test DNS by asking a resolver directly

**To learn whether your network resolves a name, query a resolver by address; asking from any
one machine answers only for that machine.**

- **Failure (2026-08):** service names that worked on one desktop existed only in that desktop's
  hosts file. No resolver on the network knew them, and nothing on the server ran DNS. See
  [WSL resolves names the way Windows does](../wsl/README.md#wsl-resolves-names-the-way-windows-does).
- **Check:** `dig @<resolver-address> <name>` against each resolver clients actually use: the
  router, and the tailnet resolver if MagicDNS is on. NXDOMAIN from all of them is the answer,
  whatever your own machine says.

## A tailnet-first resolver fails new connections only

**If a machine's first DNS resolver is reachable only over the tailnet, losing the tailnet
breaks every new connection while established ones survive.**

- **Failure (2026-08, observed):** on a WSL host using the Windows network stack, Tailscale DNS
  put a tailnet-only resolver first. When the tailnet dropped, a long-lived client's streaming
  connection stayed up while its heartbeats, which opened new connections, all failed. It gave
  up and never recovered.
- **Check:** `resolvectl status` (or `ipconfig /all` on Windows) and note which resolver is
  first. Test resolution with the tailnet down before relying on it. Do not install a second
  `tailscaled` inside WSL under mirrored networking; it fights the Windows client for the same
  identity.

## Losing the tailnet takes tailnet-only services with it

**Anything published only over the tailnet, such as a notification server behind
`tailscale serve`, disappears in a tailnet outage, and messages sent during it are lost.**

- **Failure (2026-08):** during the tag outage above, a message bus exposed only through
  `tailscale serve` was unreachable, Prometheus lost every cross-host scrape target, and local
  checks stayed green.
- **Check:** list what your alerting depends on and ask which of it survives the tailnet being
  down. The alert that says "the tailnet is down" cannot travel over the tailnet.
