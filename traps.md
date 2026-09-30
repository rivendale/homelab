# Traps index

Every entry in this repo, one line each. Each links to the rule, the failure behind it, and how
to check. Rules that live in a sibling repo are linked there.

## [WSL2](wsl/README.md)

- [Mirrored networking destroys every socket at once](wsl/README.md#mirrored-networking-destroys-every-socket-at-once)
- [WSL resolves names the way Windows does](wsl/README.md#wsl-resolves-names-the-way-windows-does)
- [A check that passes from home can fail from CI](wsl/README.md#a-check-that-passes-from-home-can-fail-from-ci)
- [PID 1 inside WSL can be killed by the OOM killer](wsl/README.md#pid-1-inside-wsl-can-be-killed-by-the-oom-killer)
- [The distro lives only while a client is attached](wsl/README.md#the-distro-lives-only-while-a-client-is-attached)
- [Nothing in `/tmp` survives a Windows restart](wsl/README.md#nothing-in-tmp-survives-a-windows-restart)
- [Call Windows binaries by full path from WSL](wsl/README.md#call-windows-binaries-by-full-path-from-wsl)

## [systemd](systemd/README.md)

- [Ask the bus the unit actually lives on](systemd/README.md#ask-the-bus-the-unit-actually-lives-on)
- [A oneshot reads activating while it runs](systemd/README.md#a-oneshot-reads-activating-while-it-runs)
- [The user manager has its own PATH](systemd/README.md#the-user-manager-has-its-own-path)
- [MemoryCurrent counts page cache](systemd/README.md#memorycurrent-counts-page-cache)
- [MemoryHigh turns a crash into a hang](systemd/README.md#memoryhigh-turns-a-crash-into-a-hang)
- [The cap that binds may belong to a parent](systemd/README.md#the-cap-that-binds-may-belong-to-a-parent)
- [High load with an idle CPU means paging](systemd/README.md#high-load-with-an-idle-cpu-means-paging)
- [Run long jobs in their own transient unit](systemd/README.md#run-long-jobs-in-their-own-transient-unit)
- [A wrapper's exit is not the work finishing](systemd/README.md#a-wrappers-exit-is-not-the-work-finishing)
- [StandardOutput file does not truncate](systemd/README.md#standardoutput-file-does-not-truncate)
- [A process can survive its own death](systemd/README.md#a-process-can-survive-its-own-death)
- [A prompt on discarded stdout blocks silently](systemd/README.md#a-prompt-on-discarded-stdout-blocks-silently)
- [Find a process without matching yourself](systemd/README.md#find-a-process-without-matching-yourself)
- [Decide what unattended upgrades may restart](systemd/README.md#decide-what-unattended-upgrades-may-restart)

## [Networking](networking/README.md)

- [A Tailscale tag replaces the user identity](networking/README.md#a-tailscale-tag-replaces-the-user-identity)
- [Disable key expiry on servers you own](networking/README.md#disable-key-expiry-on-servers-you-own)
- [Tailnet SSH and sshd are two doors](networking/README.md#tailnet-ssh-and-sshd-are-two-doors)
- [Some consumer gateways only accept a reservation inside the DHCP pool](networking/README.md#some-consumer-gateways-only-accept-a-reservation-inside-the-dhcp-pool)
- [A DHCP lease is a fuse](networking/README.md#a-dhcp-lease-is-a-fuse)
- [The tailnet name is the identity](networking/README.md#the-tailnet-name-is-the-identity)
- [Test DNS by asking a resolver directly](networking/README.md#test-dns-by-asking-a-resolver-directly)
- [A tailnet-first resolver fails new connections only](networking/README.md#a-tailnet-first-resolver-fails-new-connections-only)
- [Losing the tailnet takes tailnet-only services with it](networking/README.md#losing-the-tailnet-takes-tailnet-only-services-with-it)
- [No third-party LLM routers between agents and providers](networking/README.md#no-third-party-llm-routers-between-agents-and-providers)

## [Sync and backup](sync-and-backup/README.md)

- [Syncthing versions only what Syncthing changed](sync-and-backup/README.md#syncthing-versions-only-what-syncthing-changed)
- [Read version age from the filename suffix](sync-and-backup/README.md#read-version-age-from-the-filename-suffix)
- [A whitespace encryption password is a real password](sync-and-backup/README.md#a-whitespace-encryption-password-is-a-real-password)
- [A password on your own device row hides you](sync-and-backup/README.md#a-password-on-your-own-device-row-hides-you)
- [A whitespace versions path blocks every replace](sync-and-backup/README.md#a-whitespace-versions-path-blocks-every-replace)
- [The readable config may not be the live one](sync-and-backup/README.md#the-readable-config-may-not-be-the-live-one)
- [A Windows service account needs a grant on every folder](sync-and-backup/README.md#a-windows-service-account-needs-a-grant-on-every-folder)
- [A sweep needs a positive control](sync-and-backup/README.md#a-sweep-needs-a-positive-control)
- [Keep the evidence that authorizes a delete](sync-and-backup/README.md#keep-the-evidence-that-authorizes-a-delete)

## [Alerting](alerting/README.md)

- [ntfy turns a long message into an attachment](alerting/README.md#ntfy-turns-a-long-message-into-an-attachment)
- [Filter on an optional label with negation](alerting/README.md#filter-on-an-optional-label-with-negation)
- [Test a suppression on release](alerting/README.md#test-a-suppression-on-release)
- [A watchdog on the same host cannot report its death](alerting/README.md#a-watchdog-on-the-same-host-cannot-report-its-death)
- [One sample of a volatile signal is not evidence](alerting/README.md#one-sample-of-a-volatile-signal-is-not-evidence)
- [A textfile metric can be stale](alerting/README.md#a-textfile-metric-can-be-stale)

## [Windows](windows/README.md)

- [Two registry keys do not prove no reboot is pending](windows/README.md#two-registry-keys-do-not-prove-no-reboot-is-pending)
- [An ACL grant cannot open an encrypted file](windows/README.md#an-acl-grant-cannot-open-an-encrypted-file)
- [An encrypted directory encrypts every new file](windows/README.md#an-encrypted-directory-encrypts-every-new-file)
- [A scheduled task's Last Result can have inverted polarity](windows/README.md#a-scheduled-tasks-last-result-can-have-inverted-polarity)

## Linked from the opensource lessons

- [Identical alerts destroy the signal](https://github.com/rivendale/opensource/blob/main/lessons/README.md#monitoring-and-alerts)
- [An alert whose wait outlasts the outage never fires](https://github.com/rivendale/opensource/blob/main/lessons/README.md#monitoring-and-alerts)
- [Assert what the response contains](https://github.com/rivendale/opensource/blob/main/lessons/README.md#deploying-small-games)
- [Compare against a shared ref, not the checkout](https://github.com/rivendale/opensource/blob/main/lessons/README.md#testing-prove-each-check-can-fail)
