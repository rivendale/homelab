# AGENTS.md

Working agreement for anyone, human or agent, changing this repo. It is the one instruction
file; there is no second name for it, because two instruction files drift and the one a tool
loads automatically wins the contradiction.

## What this repo is

Practice for running a small home lab and home servers, published so a stranger can use it.
Changes arrive by pull request. Nothing is committed to `main` directly.

## What an entry must have

- **The rule in one bold line.** If it takes a paragraph to state, it is two entries.
- **The failure it came from**, described generically and dated to the month. Practice, not
  the story: what broke and why, never who was affected.
- **The product and version measured** where the behavior could change between versions.
  "Tailscale 1.102" or "Syncthing 2.1" is useful; "Tailscale" alone is not.
- **How to check**: a command, a field or a test a reader can run on their own machine. A
  rule with no check is a belief.
- **Observed, with a date,** on anything not re-verified since it was first seen. Do not
  upgrade an observation to a fact because it sounds right.

## Hard lines for public text

- No person names, households, employers or firms.
- No hostnames, tailnet names, IP addresses of any kind, MAC addresses, home-directory paths
  or account numbers. Use placeholders: `<server>`, `<desktop>`, `<tailnet-name>`, `<user>`.
- No router model together with where it is installed.
- No secrets, and no command that prints one. Report where a credential lives, never what it
  is.
- American English. No em dashes.

## Before you open a pull request

- Every relative link resolves. Script it; do not eyeball it.
- Grep the tree for hostnames, addresses and home paths from your own environment. The
  placeholders exist so nothing real has to be written.
- If a rule already lives in a sibling repo
  ([opensource lessons](https://github.com/rivendale/opensource/blob/main/lessons/README.md),
  [hsi-operator](https://github.com/rivendale/hsi-operator)), link to it and add only what is
  new. One fact in two repos is two places to correct.
- Add the entry to [traps.md](traps.md) and a line to [CHANGELOG.md](CHANGELOG.md) naming the
  failure behind the change.

## Verification habits

- Verify against live state, not a document. Say plainly what is unverified.
- A check that returned nothing may not have run. Ask whether it could have found anything.
- Prove a check in the denying direction, with an input it must refuse.
- Test from the consuming end. A producer reporting success is not a consumer receiving it.
- A correction lands where the claim lives, and everywhere it was copied.
