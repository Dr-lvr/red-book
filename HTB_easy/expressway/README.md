# HTB Expressway

> Linux | IKE/IPsec VPN, Exim4, Sudo
>
> Two evenings after work, ~5 hours. Rated easy. This one taught me things.

---

## Recon

TCP scan: just SSH. The box doesn't show its hand until you scan UDP — port 500 (ISAKMP) and 4500 (NAT-T) are open. IKE/IPsec VPN running.

This was my first time touching IKE in a pentest context. Everything from here was new ground.

---

## Foothold

`ike-scan` in aggressive mode against the target reveals a full SA handshake with PSK authentication. The server responds with its ID (`ID_USER_FQDN` type) and a hash. Fed the capture through `ikescan2john.py` and cracked the PSK with John.

Hashcat couldn't parse the format — token length exception on mode 5300, 5400, and 15000. Spent 30 minutes reformatting the hash before switching to John Jumbo, which ate it in one second. Lesson: when hashcat chokes on exotic formats, don't waste time — go to John.

The cracked PSK doubled as the SSH password for the user whose ID was in the IKE handshake. Password reuse. User flag.

---

## The VPN Detour

Before discovering the SSH shortcut, I spent a solid hour trying to actually establish the IPsec tunnel. This was educational in the worst way.

**strongSwan 6.x** doesn't support IKEv1 aggressive mode. The server requires aggressive mode. I didn't know this going in and burned time on configs that silently failed or produced AUTH_FAILED because strongSwan was sending Main Mode.

**Switched to libreswan.** Libreswan and strongSwan conflict — different config syntax, different keywords, different services fighting over the same binary name. `charondebug` crashes libreswan. `ikev2=never` is deprecated. `modp1024` is blocked by default crypto policy. Every fix uncovered a new incompatibility.

**Never connected.** By the time I had libreswan almost configured, I'd already tried the PSK as an SSH password and was in. The VPN was a rabbit hole — but I now know the difference between strongSwan and libreswan, why aggressive mode matters, and what `ID_USER_FQDN` means in an IKE negotiation. That's not nothing.

---

## Privilege Escalation

On the box: SSH (22) and SMTP (25, localhost only). Exim 4.98.2 as MTA. An `/etc/aliases` entry redirecting root's mail to a deleted user. `pspy` shows a recurring `apt-get changelog` running as root — `apt-listchanges` with `frontend=pager` processing package updates.

Enumerated the email pipeline, tried command injection through Subject headers, watched `sed` get called on my input. Interesting, but the email processing runs as my user, not root.

Then checked sudo version: **1.9.17**.

CVE-2025-32463 — the chroot-to-root bug. Sudo 1.9.14 through 1.9.17 resolves paths via `chroot()` using a user-specified root directory while still evaluating sudoers. Create a fake chroot with a poisoned `/etc/nsswitch.conf` pointing to a malicious NSS shared library, and sudo loads it as root.

`gcc` was on the box. Compiled a `.so` with a constructor that calls `setreuid(0,0)` and spawns `/bin/bash`. Pointed `sudo -R` at the crafted chroot. Root shell.

---

## What I Learned

**IKE/IPsec is its own world.** I'd never touched it before this box. Aggressive vs Main mode, PSK authentication, SA proposals, transform sets, `ID_USER_FQDN` — all new vocabulary that I can now read and reason about. Not fluent yet, but the door is open.

**Hashcat isn't always the answer.** John Jumbo with `ikescan2john.py` handled the IKE hash format cleanly. Hashcat couldn't parse it across three different modes. Know your tools, and know when to switch.

**strongSwan ≠ libreswan ≠ IPsec.** They're different implementations with different capabilities and different config formats. strongSwan 6.x dropped IKEv1 aggressive mode. If the server requires it, you need libreswan or racoon. I learned this the hard way.

**Check sudo version early.** I went down the exim/email rabbit hole for a while before checking something basic. CVE-2025-32463 was sitting there the whole time. sudo version should be in the first five commands after landing a shell.

---

## Timeline

| Phase | Time |
|-------|------|
| Recon (TCP + UDP) | ~20 min |
| IKE scan + hash crack | ~40 min |
| VPN rabbit hole | ~1 hour |
| SSH + user flag | ~5 min |
| Exim/email enumeration | ~1 hour |
| Sudo CVE + root | ~30 min |
| **Total** | **~4 hours** |

---

## References

- [CVE-2025-32463 — Sudo chroot privilege escalation](https://www.sudo.ws/security/advisories/chroot_bug/)
- [PoC exploit](https://github.com/nflatrea/CVE-2025-32463)
