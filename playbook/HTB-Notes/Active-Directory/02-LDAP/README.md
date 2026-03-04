# 02 — Active Directory LDAP
> HTB Academy | AD Penetration Tester Path  
> red-book/modules/02-AD-LDAP.md

---

## Getting Started in Active Directory

**AD Structure:** Forest → Domain → OU → Objects (users, computers, groups, GPOs, trusts)

**Why enumerate AD?** AD misconfigs are the primary attack surface in enterprise networks. Even a low-priv domain user can query most of the directory by default.

**Rights vs Permissions:**
- *Rights* — system-level (e.g. SeDebugPrivilege, logon rights)
- *Permissions* — object-level ACLs (e.g. read/write on a user object)

**RSAT (Remote Server Administration Tools):** legitimate Windows admin tools. If installed on a compromised host, usable for stealthy enumeration via native binaries.

**NT AUTHORITY\SYSTEM:** full local privileges, authenticates to the domain as the machine account (`HOSTNAME$`). Useful for domain enum from a compromised machine without needing user creds.

---

## Leveraging Search Filters

### LDAP Overview
LDAP is the protocol used to query Active Directory. Every query needs:
- **Host** — DC IP or hostname
- **Base DN** — where to start the search (`DC=domain,DC=local`)
- **Filter** — what to look for (`(objectClass=user)`)
- **Attributes** — what fields to return

### LDAP Filter Syntax
| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | equal | `(samaccountname=administrator)` |
| `*` | wildcard | `(description=*)` |
| `&` | AND | `(&(objectClass=user)(adminCount=1))` |
| `\|` | OR | `(\|(objectClass=user)(objectClass=computer))` |
| `!` | NOT | `(!(userAccountControl:1.2.840.113556.1.4.803:=2))` |

**userAccountControl flags (common):**
| Flag | Value | Meaning |
|------|-------|---------|
| ACCOUNTDISABLE | 2 | account disabled |
| PASSWD_NOTREQD | 32 | no password required |
| DONT_EXPIRE_PASSWD | 65536 | password never expires |
| NO_PREAUTHENTICATION | 4194304 | AS-REP roastable |

### Active Directory Search Filters — Key Filters
```bash
# All users
(objectClass=user)

# Enabled users only
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Privileged accounts (adminCount=1)
(&(objectClass=user)(adminCount=1))

# AS-REP roastable
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))

# Kerberoastable (SPN set)
(&(objectClass=user)(servicePrincipalName=*))

# Accounts with description field set (often contains passwords)
(&(objectClass=user)(description=*))

# All groups
(objectClass=group)

# All computers
(objectClass=computer)

# GPOs
(objectClass=groupPolicyContainer)
```

### Enumerating AD with Built-in Tools
```powershell
# Native LDAP query via .NET (no tools needed)
$searcher = New-Object DirectoryServices.DirectorySearcher
$searcher.Filter = "(&(objectClass=user)(adminCount=1))"
$searcher.FindAll() | ForEach-Object { $_.Properties["samaccountname"] }

# ADSI via PowerShell
([adsisearcher]"(objectClass=user)").FindAll()

# net commands (noisy but native)
net user /domain
net group "Domain Admins" /domain
```

---

## LDAP Enumeration

### Anonymous Bind
Some DCs allow unauthenticated LDAP queries — always test first.
```bash
# Test anonymous bind
ldapsearch -H ldap://<IP> -x -b "DC=domain,DC=local" -s base

# Enumerate if allowed
ldapsearch -H ldap://<IP> -x -b "DC=domain,DC=local" "(objectClass=user)" samaccountname description

# With windapsearch
windapsearch --dc <IP> -m users
windapsearch --dc <IP> -m metadata       # domain info without creds
```

### Credentialed LDAP Enumeration
```bash
# ldapsearch
ldapsearch -H ldap://<IP> -x \
  -D "user@domain.local" -w 'pass' \
  -b "DC=domain,DC=local" \
  "(&(objectClass=user)(adminCount=1))" \
  samaccountname memberof pwdLastSet

# ldapdomaindump — outputs HTML + JSON, great for quick overview
ldapdomaindump -u 'domain\user' -p 'pass' <IP> -o ./ldap-dump/

# windapsearch
windapsearch --dc <IP> -d domain.local -u user -p pass --da        # domain admins
windapsearch --dc <IP> -d domain.local -u user -p pass --privileged-users
windapsearch --dc <IP> -d domain.local -u user -p pass -m computers

# NetExec
nxc ldap <IP> -u user -p pass --query "(adminCount=1)" "samaccountname"
nxc ldap <IP> -u user -p pass --users
nxc ldap <IP> -u user -p pass --groups
nxc ldap <IP> -u user -p pass --password-not-required
nxc ldap <IP> -u user -p pass --bloodhound -c All
```

**Key attributes to always pull:**
| Attribute | Why |
|-----------|-----|
| `samaccountname` | username |
| `description` | often contains plaintext passwords |
| `memberOf` | group membership |
| `adminCount` | 1 = was/is in privileged group |
| `userAccountControl` | account flags (disabled, no preauth, etc.) |
| `pwdLastSet` | 0 = never set / password not required |
| `msDS-KeyCredentialLink` | shadow credentials present |
| `servicePrincipalName` | kerberoastable if set on user |
| `mail` / `telephoneNumber` | useful for OSINT / spraying |

---

## Skills Assessment Checklist
- [ ] Test anonymous bind
- [ ] Enumerate users with `adminCount=1`
- [ ] Check for accounts with `description` populated
- [ ] Find AS-REP roastable accounts (`NO_PREAUTHENTICATION` flag)
- [ ] Find Kerberoastable accounts (`servicePrincipalName=*` on users)
- [ ] Dump all groups and memberships
- [ ] Check `pwdLastSet=0` accounts
- [ ] Look for `msDS-KeyCredentialLink` on any object

---

## Tools Reference
| Tool | Use |
|------|-----|
| `ldapsearch` | manual queries, precise filter control |
| `ldapdomaindump` | full dump → HTML/JSON output |
| `windapsearch` | fast targeted queries |
| `nxc ldap` | Swiss army knife, BloodHound collection |
| PowerShell ADSI | living off the land on Windows |

---

## Personal Notes
> *[fill during/after module]*
