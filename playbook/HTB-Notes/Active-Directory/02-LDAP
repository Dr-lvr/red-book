## 02 — AD LDAP 🔄

### Getting Started in Active Directory
- **AD Overview:** hierarchical structure — Forest > Domain > OU > Objects (users, computers, groups, GPOs)
- **Why Enumerate AD?** — AD misconfigs are the #1 attack surface in enterprise environments
- **Rights & Privileges:** difference between rights (logon, shutdown) vs permissions (object-level ACLs)
- **RSAT:** Windows admin tools usable for enumeration if installed on a compromised host
- **NT AUTHORITY\SYSTEM:** local machine account — has full local rights but authenticates to the domain as the machine account (`HOSTNAME$`)

### Leveraging Search Filters

**LDAP Overview:** LDAP is the protocol used to query AD. Queries target a base DN and use filters.
```
Base DN example:  DC=domain,DC=local
Object filter:    (objectClass=user)
Attribute combo:  (&(objectClass=user)(adminCount=1))
```

**LDAP Search Filters — syntax:**
