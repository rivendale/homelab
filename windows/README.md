# Windows

The Windows side of a desktop that also hosts WSL services: knowing when it will reboot, and why
a file every permission tool says is readable still refuses to open.

Measured on Windows 11, 2026-08 and 2026-09.

## Two registry keys do not prove no reboot is pending

**The two most-cited reboot-pending keys miss driver updates; check every signal, and treat the
Settings page as the authority.**

- **Failure (2026-09):** "no reboot pending" was reported from two keys, both False, while Windows
  Update showed "Restart required" for a driver update. The driver update had staged 78 entries in
  `PendingFileRenameOperations` and set neither key.
- **Check all of these before saying no:**
  - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending`
  - `...\Component Based Servicing\RebootInProgress` and `...\PackagesPending`
  - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired`
  - `...\WindowsUpdate\Auto Update\PostRebootReporting`
  - `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\PendingFileRenameOperations`
  - `HKLM\SOFTWARE\Microsoft\Updates\UpdateExeVolatile`

  Any one can be the only one set. If a probe and the Settings page disagree, the probe is
  incomplete. If you checked a subset, say so.
- **Why it matters with WSL:** if the distro starts at interactive logon, a reboot leaves every
  WSL service down until someone signs in. See
  [the distro lives only while a client is attached](../wsl/README.md#the-distro-lives-only-while-a-client-is-attached).

## An ACL grant cannot open an encrypted file

**On Windows, opening a file needs the ACL grant and, for an EFS-encrypted file, the key of the
account that encrypted it; `icacls` shows only the first.**

- **Failure (2026-08 and 2026-09):** files existed on one replica and nowhere else. `icacls`
  showed the sync service account holding inherited Modify, so permissions were ruled out and the
  cause declared unknown. The sync program's own failed-items list had said
  `hashing: open ...: Access is denied` the whole time. The files carried the Encrypted
  attribute; only the user who created them could decrypt them.
- **Check:** `cipher /c <file>` lists who can decrypt it. For a tree,
  `cipher /s:<root>` (one survey of about 70,000 files found exactly the 33 that were failing), or
  in PowerShell
  `Get-ChildItem -Recurse -Force | Where-Object { $_.Attributes -band [IO.FileAttributes]::Encrypted }`.
  Better still, try the open as the account in question.
- **Read the failing program's error log before theorizing.** Three hypotheses were tested against
  the filesystem before anyone read what the program said was wrong.

## An encrypted directory encrypts every new file

**EFS on a directory sets encrypt-on-create, so every file written into it later is encrypted;
decrypting the files alone makes the problem come back.**

- **Failure (2026-09):** encrypted files kept reappearing in a synced folder after they had been
  decrypted twice. Two of the encrypted entries were directories. The real source was further up:
  the user's Desktop folder and 329 directories under it, including the downloads folder, carried
  the encrypt-on-create bit. A download was born encrypted, and a move within the same volume keeps
  the attribute rather than taking the destination's, so the sync folder correctly reported "new
  files will not be encrypted" while encrypted files kept arriving in it.
- **Check:** ask which directory the file was created in, not where it ended up. Survey
  directories with `cipher /s:<root>` and look for `E` on directories. `cipher /d <dir>` clears the
  bit; verify by writing a new file in each affected directory and confirming it comes out
  unencrypted.

## A scheduled task's Last Result can have inverted polarity

**For a task whose job is to stay running, Last Result `0` is the failure and `267009` is
health.**

- **Failure (2026-08):** a logon task reported `0` while the thing it was meant to keep alive died
  behind it. See
  [the distro lives only while a client is attached](../wsl/README.md#the-distro-lives-only-while-a-client-is-attached).
- **Check:** `267009` is `0x41301`, "the task is currently running". Check the effect the task
  exists for, not its result code. Changing a task's action with `Set-ScheduledTask` needs an
  elevated shell; unelevated it returns "Access is denied".
