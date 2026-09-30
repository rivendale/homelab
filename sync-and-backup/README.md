# Sync and backup

Keeping files replicated across a Windows desktop, a Linux server and laptops with Syncthing,
and knowing whether a deletion can be recovered.

The thread through every Syncthing entry below: it reports health from somewhere other than
the effect. A config field that looks empty is not, the readable config is not the live one, a
folder the daemon cannot write still reads as connected, and a version store proves nothing
until a remote deletion lands in it.

Measured on Syncthing 2.1, with the Windows node running Syncthing as a service under its own
local account, 2026-08 and 2026-09.

## Syncthing versions only what Syncthing changed

**Syncthing archives a file into `.stversions` only when it applies a remote delete or replace;
a local delete or edit is never versioned.**

- **Failure (2026-09):** a probe wrote a file, deleted it locally, and found no version.
  "Versioning is dead" nearly went out as a finding. The probe exercised the one path versioning
  does not cover, so it could not have produced a version on a healthy folder either.
- **Check, with two hosts:** write a file on A, confirm it lands on B, delete it on A, then look
  in B's `.stversions`. Only then has Syncthing performed the deletion. Inspecting a version
  store tells you it is not empty; causing a deletion tells you it works.
- **Zero versions over a quiet period is normal** when the traffic was new files. New files
  never version.

## Read version age from the filename suffix

**A version's age is the `~YYYYMMDD-HHMMSS` suffix in its filename, in the creating host's local
time with no offset; the file's mtime is the original document's date.**

- **Failure (2026-09):** a recoverability metric reported the oldest version as 6,300 days old.
  It was a seventeen-year-old document; Syncthing preserves the original mtime. The fix then
  parsed the suffix as UTC and reported versions five hours old that were 61 minutes old, off by
  exactly the local offset. Both errors leaned in the reassuring direction.
- **Check:** parse the suffix, and apply the timezone of the host that holds the version.

## A whitespace encryption password is a real password

**Syncthing treats a whitespace-only `encryptionPassword` as a real password, which switches that
device to encrypted sharing and breaks the peer.**

- **Failure (2026-08):** one peer disconnected with "remote expects to exchange plain data, but
  local data is encrypted", which reads like a fault on the remote side. The value was a newline
  and spaces, most likely inherited from the config's `<defaults>` template when the device was
  added.
- **Check:** read the field's length from `GET /rest/config/folders`, never an emptiness test,
  which some parse paths pass for whitespace. A padlock beside a folder in the remote device panel
  means encrypted sharing. Fix live by setting it to `""` and `PUT` the folder back.

## A password on your own device row hides you

**An `encryptionPassword` on the local device's own row makes Syncthing leave itself out of the
cluster config it sends, so every peer rejects it.**

- **Failure (2026-09, observed):** tens of thousands of consecutive connection failures, each
  dying one or two seconds after TLS, against every peer, with "remote device missing in cluster
  config". Two passes chased a single peer pair and a version gap because the log was read from
  one side only. The failing node's own log named every peer, which made it a single-host fault.
- **The trap that hid it:** `config.xml` on disk said the field was empty; the running process
  held a nine-character value. Every inspection had read the file.
- **Measured on:** Syncthing 2.1.5, `generateClusterConfigRLocked` in `lib/model/model.go`.
- **Check:** ask the daemon (`/rest/config/folders`), never the file. Compare full device IDs,
  not seven-character prefixes.

## A whitespace versions path blocks every replace

**A versioning path containing only whitespace makes the versioner fail, and because archiving
is part of finalizing a replace, every replace in the folder fails while creates keep working.**

- **Failure (2026-09):** a script that correctly found versioning was not archiving set every
  folder to staggered versioning with a malformed path. From then on every replace failed and
  retried on backoff out to an hour, leaving full-size `~syncthing~<name>.tmp` files. Only one
  folder failed visibly, because only it had a replace to attempt; the others were equally broken
  and quiet. The folder state read `idle` with an empty `error`.
- **Check:** when replaces stall and creates work, suspect the versioner before the network. Read
  `/rest/folder/errors` and the log for `Failed to sync`; the folder-level error field carries
  folder-level faults only. Check path fields by length.
- **The lesson under it:** a fix that returns silently can turn a silent fault into a loud one.
  Verify a repair by the effect, not the command's exit.

## The readable config may not be the live one

**A Syncthing `config.xml` you can read may belong to an old install; the running daemon can be
on another port with a config you cannot read.**

- **Failure (2026-09):** a tool pointed at a readable config (months stale, naming a dead GUI
  port) found zero folders and reported "unreachable" honestly in its own terms, describing a
  daemon that no longer existed. The live service ran under its own account with its home
  directory closed to the user.
- **Check:** confirm the GUI port answers (`curl -fsS -m5 http://localhost:8384/rest/noauth/health`),
  then confirm the config you use names that port. Get the real home directory from the service
  registration (`--home` in its command line), not from a guess.

## A Windows service account needs a grant on every folder

**When Syncthing runs as a Windows service account, a folder without an explicit grant for that
account fails to scan, and a folder it cannot write still reads as connected to every peer.**

- **Failure (2026-08):** folders moved (not copied) into the sync tree kept their old ACL with
  inheritance disabled. Syncthing logged "Access is denied" and those subtrees never synced while
  siblings worked, which looked like a peer problem.
- **Failure (2026-09):** there was no grant at the parent at all; each working folder had its own.
  A new folder got nothing, so the daemon could not create `.stfolder` or write a byte. Index
  exchange needs no filesystem access, so every peer showed the folder connected and valid while
  it held nothing.
- **Check:** `(Get-Acl <path>).AreAccessRulesProtected` (True means inheritance is blocked) and
  whether the service account appears in `(Get-Acl <path>).Access` at all. If inheritance is on
  and there is still no grant, there is nothing to inherit; add one, or better, one inheritable
  grant at the parent. Verify by the daemon creating `.stfolder`, not by the command returning.
- **If the grant is right and reads still fail,** check for EFS encryption:
  [an ACL grant cannot open an encrypted file](../windows/README.md#an-acl-grant-cannot-open-an-encrypted-file).

## A sweep needs a positive control

**A search that finds zero of something prints the same as a search that walked nothing; print
how many items it examined.**

- **Failure (2026-08):** a sweep for encrypted files returned zero. The rerun with a count showed
  97,069 files traversed and 0 encrypted, and only that run meant anything. In another case a
  recursive listing under a directory the user could not read, with errors suppressed, returned a
  confident zero.
- **Check:** every sweep prints its traversal count, and does not suppress permission errors.

## Keep the evidence that authorizes a delete

**The file-by-file evidence that licenses deleting something is not working material; write it
somewhere durable the moment it is produced.**

- **Failure (2026-09):** a reboot erased per-file hash lists proving which files had verified
  copies elsewhere. The summary survived and read well; it could not authorize removing a single
  file, and the hashing had to be redone. See
  [nothing in /tmp survives a Windows restart](../wsl/README.md#nothing-in-tmp-survives-a-windows-restart).
- **Check:** before deleting, confirm the per-item evidence is on durable storage, and verify each
  deletion by size or inode rather than by the name you deleted with.
