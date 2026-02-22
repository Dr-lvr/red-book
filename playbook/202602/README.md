# Operational Playbook — Built on My Mistakes

> This is not a generic playbook. It's extracted from 7 HTB boxes in 9 days and the patterns I kept repeating.
> Every rule exists because I broke it at least once.

---

## Rule 0: The first 5 minutes decide the next 4 hours

The moment I land on a shell, before enumerating anything:

```
1. sudo -l
2. id && groups
3. uname -a (Linux) / systeminfo (Windows)
4. cat /etc/os-release (Linux) / [System.Environment]::OSVersion (Windows)
5. Versions of exposed services (sudo --version, exim --version, python3 --version, etc.)
```

**Why**: on Expressway I spent an hour on the email pipeline before checking sudo. The CVE was right there from minute one. On WingData I enumerated for 4 hours before going back to the hashes. Quick wins should be checked first, not last.

---

## Rule 1: Hashes found = hashcat in background, NOW

Not "I'll note it for later." Not "let me find a technical path first." I launch it immediately and keep enumerating.

```bash
# Template: the moment I find hashes
# 1. Identify format
hashcat --identify hash.txt 2>/dev/null || john --list=formats | grep <keyword>
# 2. Launch in background
hashcat -m <mode> hash.txt /path/to/rockyou.txt &
# or
john --wordlist=/path/to/rockyou.txt hash.txt &
# 3. Keep enumerating
```

**Why**: WingData. 4 hours of dead ends, hashes found at minute 30, cracked at minute 270. Had I launched immediately, I'd have had the password in 5 minutes and could have spent those 4 hours on root (which was the part that actually made me grow).

**Corollary**: if hashcat can't parse the format, go to John Jumbo. Don't waste 30 minutes reformatting (Expressway, IKE hash).

---

## Rule 2: Check software versions against CVEs BEFORE deep enumeration

```bash
# Linux
sudo --version
python3 --version
<service> --version  # exim, apache, nginx, etc.
dpkg -l | grep <package>

# Windows
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
(Get-ADDomain).DomainMode  # For AD
```

Then a quick search: `<software> <version> CVE privilege escalation`

**Why**: 
- Expressway: sudo 1.9.17 → CVE-2025-32463 → root in 2 minutes
- WingData: Python 3.x + tarfile filter="data" → CVE-2025-4517 → root
- Eighteen: Windows Server 2025 → BadSuccessor (not a classic CVE but version-specific)

The pattern is clear: HTB easy/medium boxes often have recent CVEs as the main vector. Searching for them early saves hours.

---

## Rule 3: The rabbit hole has a timer

If I've been enumerating a vector for more than 45 minutes with no concrete progress, I stop and reassess.

**Check at 45 minutes:**
- Have I found anything new in the last 15 minutes? If no → switch vector.
- Am I retrying the same thing with minor variations? If yes → switch vector.
- Do I have at least 2 alternative vectors to try? If no → move to broader enumeration.

**Why**: 
- Expressway: 1 hour on the VPN tunnel that wasn't needed. SSH with the same password was the path.
- WingData: 4 hours enumerating from wingftp. Every vector hit the same wall (filesystem permissions). I should have recognized the pattern after the second dead end, not the seventh.

**Exception**: if I'm learning something new and time isn't a constraint (practice, not engagement), the timer doesn't apply. But I need to be aware of it — "I'm exploring" is different from "I'm looking for the solution."

---

## Rule 4: When the wall is OS permissions, don't insist at the application level

If the service runs as user X and I need to access files owned by user Y, and filesystem permissions don't allow it:

- Changing the service configuration **doesn't change the process UID**
- Granting "permissions" in the service admin panel **doesn't change OS permissions**
- Executing code in the service context **is still user X**

The only way is: become user Y, or find something that runs as user Y or root.

**Why**: WingData. FTP admin panel, virtual directories, Lua console, webclient API — all useless because the FTP process was wingftp (uid 1000), and the target was wacky (uid 1001) with `drwxrwx---`. I understood the concept after the third attempt but kept going until the seventh "just to be sure." The certainty was an illusion.

---

## Rule 5: On Windows/AD, load tools BEFORE enumerating

Every time I land on a Windows shell:

```powershell
# 1. Upload PowerView (or SharpHound)
upload PowerView.ps1
. .\PowerView.ps1

# 2. If Evil-WinRM, verify cmdlets work
Get-DomainUser -Identity <my_user>

# 3. If it doesn't work, use manual LDAP or SharpHound
```

**Why**: Eighteen. I wasted time because PowerView wasn't loaded in the new session and commands were failing silently. On Windows, if I don't verify tools work immediately, I waste time on tooling errors, not methodology errors.

---

## Rule 6: Password spray and credential reuse ALWAYS

Every credential found gets tried everywhere:

```bash
# Template
nxc ssh <target> -u <user> -p '<password>'
nxc winrm <target> -u <user> -p '<password>'
nxc smb <target> -u <user> -p '<password>'
# If I have a user list:
nxc <protocol> <target> -u users.txt -p '<password>'
nxc <protocol> <target> -u users.txt -p passwords.txt --no-bruteforce
```

**Special case**: NT_STATUS_PASSWORD_MUST_CHANGE on spray → credentials are valid, account is in expired state.

```bash
Tool: smbpasswd -r <target> -U <user> (rpcclient and netexec don't handle this state)
```

**Why**: Expressway. The VPN PSK was the SSH password. I didn't try it right away — I spent an hour on the IPsec tunnel first. On Eighteen, the cracked password from the spray gave direct access.

---

## Rule 7: Document dead ends, not just successes

For every box, I keep a scratch file:

```
## Tried
- [x] FTP admin panel → OS permissions block (wingftp uid)
- [x] Lua console → RCE as wingftp, same shell I already have
- [x] VPN tunnel → strongswan doesn't support IKEv1 aggressive mode
- [ ] SSH with PSK as password → TO TRY

## To try
- ...
```

**Why**: prevents retrying the same thing 45 minutes later. On WingData I retried virtual directories twice before marking it as a dead end. And documented dead ends are gold for writeups — they show the method, not just the result.

---

## Rule 8: UDP exists

```bash
nmap -Pn -sU --top-ports 100 -v <target>
```

**Why**: Expressway. SSH only on TCP. The entire box depends on IKE on UDP 500. If I hadn't scanned UDP, I'd have found nothing.

---

## Rule 9: Hash format matters more than the wordlist

Before launching a crack:

```
1. Identify the EXACT format (algorithm + salt + encoding)
2. Find the correct hashcat mode / john format
3. If unsure, try john --identify or hashcat --identify
4. If hashcat fails with "token length exception" → John Jumbo
5. If john fails → search <service> + hashcat mode online
```

**Why**: 
- WingData: sha256 with salt "WingFTP" prepended, hex-encoded. Mode 1420, hex salt. I had to read the server docs to figure out the format.
- Expressway: IKE PSK hash. Hashcat couldn't parse it across 3 different modes. John with ikescan2john.py cracked it in 1 second.

---

## Rule 10: Time sync on AD

```bash
# BEFORE any Kerberos operation
sudo timedatectl set-time "$(date -d "$(curl -s -I http://<DC_IP> | grep -i '^Date:' | cut -d' ' -f2-)" '+%Y-%m-%d %H:%M:%S')"
```

**Why**: Eighteen. Kerberos tolerates a maximum 5-minute skew. If the clock is off, tickets fail silently. I learned this through trial and error.

---

## Landing Checklist

### Linux — first 5 minutes
```
□ id && groups
□ sudo -l
□ cat /etc/os-release
□ uname -a
□ Service versions (sudo, python, apache, exim, etc.)
□ ss -tlnp (internal ports)
□ cat /etc/passwd (real users)
□ cat /etc/crontab && ls /etc/cron.d/
□ find / -perm -4000 2>/dev/null (SUID)
□ Hashes found? → hashcat/john in background NOW
□ Versions → search CVEs
```

### Windows — first 5 minutes
```
□ whoami /all
□ net user <me>
□ systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
□ Upload PowerView + verify it works
□ Get-DomainUser -Identity <me> | select memberof
□ Find-InterestingDomainAcl (if AD)
□ Sudo version? (if present)
□ Internal ports: netstat -ano
□ Credentials found? → spray everywhere
□ Domain functional level (if AD)
```

---

## Recurring Error Patterns (monitor these)

| Pattern | Trigger | Action |
|---------|---------|--------|
| Insisting on a vector that hits the same wall | Third attempt with same result | STOP. Mark as dead end. Switch vector. |
| Chasing the "elegant" path over the effective one | Hashes/credentials available but untried | Launch crack/spray NOW, keep enumerating |
| Not checking software versions | Landing and enumerating services without versions | Versions FIRST, CVE search FIRST |
| Temporal rabbit hole | >45 min on a vector with no progress | Timer. Reassess. Switch. |
| Broken tooling on Windows | Commands failing silently | Verify tools loaded BEFORE enumerating |
| Forgetting UDP | TCP-only scan | UDP top-100 is always in the first nmap |
| Password found, not tried everywhere | New credential → only tried where I found it | Spray across ALL protocols |
| SAM hash on a DC → PtH fails | Got local Admin hash, domain PtH rejected | On DCs, SAM = local accounts only. Domain hashes live in ntds.dit — dump that instead |

---

*Last updated: Feb 2026 — 7 boxes, 9 days*
*Update after every box that reveals a new pattern*
