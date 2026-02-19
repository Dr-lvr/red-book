# HTB WingData

> Linux | Wing FTP Server
>
> Two evenings after work, ~8 hours. Rated easy.

---

## Foothold

Public pre-auth RCE on the FTP server. Null byte injection in Lua session files — the server stores sessions as Lua scripts and runs them with `dofile()`. C-level `strlen` stops at the null byte, the Lua interpreter doesn't. Public exploit, shell in minutes as the FTP service account.

---

## The Service Account

The service account runs the FTP process. It can read configs, user databases, logs. It cannot access the target user's home directory (`drwxrwx---`, wrong user, wrong group) or the privesc path in `/opt/` (`drwxr-x---`, root-owned, target user's group). No sudo, no SUID, no capabilities, no cron.

Here's everything I tried before going back to the hashes.

**FTP credentials.** User passwords stored as salted SHA-256 in readable XML files. Config documents the exact format. Noted for later.

**Password manipulation + server restart.** Config files are writable. Changed FTP passwords, admin password, master password. Server restart required — kill the process, lose the shell, wait for systemd to restart, re-exploit. Got into the webclient and admin panel with new credentials.

**Admin panel.** Chisel tunnel to the internal admin port. Full control over user configuration — virtual directories, quotas, permissions, groups. Gave the target user every write permission available. Doesn't matter: the FTP process runs as the service account at the OS level. FTP permissions say "yes", the filesystem says "no."

**Lua console.** Both the admin panel and the CLI tool (`wftpconsole`) expose a Lua scripting environment. Arbitrary code execution — as the service account. Enumerated every API function the server exposes. Tried programmatic virtual directory manipulation, task scheduling, database operations. Same UID, same wall.

**Internal services.** Curl against FTP locally, webclient API, SFTP on alternate ports. Every write to the target user's directories: permission denied.

**System sweep.** Apache config (static site), bookmark database (empty), audit logs (timestamps only), grepped for credentials system-wide, checked SSH keys, hidden files. LinPEAS confirmed what I already knew.

**The upload paradox.** The webclient lets the target FTP user "upload" files with the permissions I enabled from the admin panel. But the upload handler is the FTP process, which is the service account, which can't write to the target user's home. An FTP user with full write permissions who cannot write.

---

## The Pivot

Cracked one of the five FTP hashes against rockyou. Hashcat mode 1420, `sha256(salt + password)`, hex-encoded salt. Password reused for SSH. User flag.

---

## Root

`sudo -l` reveals a Python script runnable as root. Extracts tar archives with `tarfile.extractall(filter="data")`. The `data` filter uses `os.path.realpath()` to validate symlinks, but `realpath()` silently stops resolving when the path exceeds PATH_MAX (4096 bytes). Crafted a tar that inflates the resolution path past that limit using long directory names paired with short symlinks, then chained through the unresolved symlink to write outside the extraction directory as root.

Took a few attempts — naive approaches (deep short-name chains, oversized directory names) failed for different reasons. The key from the original Google Security Research advisory is the asymmetry: short tar member names, long `realpath()` expansions.

---

## Takeaways

Launch hashcat in background from the moment you find hashes. Don't wait. I found the hashes in the first 30 minutes and cracked them after four hours of dead ends. That's a workflow gap.

But the question I'm taking home is: if that password had been strong, what path did I miss?

---

## Timeline

| Phase | Time |
|-------|------|
| Recon + foothold | ~15 min |
| Service account enumeration | ~4 hours |
| Hash crack + pivot | ~30 min |
| Root | ~3 hours |

---

## References

- [GHSA-hgqp-3mmf-7h8f](https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f)

