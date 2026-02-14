# HTB Eighteen — A 3-Day Education in BadSuccessor

> This is not a clean walkthrough — it's what actually happened, including the dead ends, the wrong assumptions, and the moments where I stared at the screen wondering if I picked the wrong career. If you're here for a copy-paste speedrun, look elsewhere. If you want to understand the *thinking*, read on.

---

## The Setup

The box is a single Windows Server 2025 domain controller. One machine, one domain, no pivoting. Sounds simple. It isn't.

I got the user flag on day one. The root flag took me three more days. Not because the attack path is complicated — in hindsight it's almost elegant — but because I had to learn an entirely new attack class that didn't exist before May 2025, with tooling that's still catching up.

---

## Getting In

I already had a user list from earlier enumeration. Two candidate passwords from previous work on the box. Standard password spray with NetExec against WinRM:

```bash
nxc winrm <target> -u users.txt -p '<password>'
```

One hit. The `(Pwn3d!)` tag means I can get a shell.

```bash
evil-winrm -i <target> -u <user> -p '<password>'
```

User flag on the desktop. Now what.

---

## The Enumeration Spiral

This is where I spent most of my time, and honestly, where I learned the most.

My compromised user is a member of the **IT** group. Not Domain Admins, not anything obviously powerful. Just "IT." My privileges are minimal — `SeMachineAccountPrivilege`, `SeChangeNotifyPrivilege`, `SeIncreaseWorkingSetPrivilege`. Nothing exploitable directly.

I started checking the usual suspects. Here's my mental checklist and what each one returned:

**Kerberoastable SPNs?** Only `krbtgt`. Dead end.

**Unconstrained delegation?** Only DC01 itself. That's default, not exploitable from here.

**RBCD (Resource-Based Constrained Delegation)?** `msDS-AllowedToActOnBehalfOfOtherIdentity` is empty on DC01. Nothing pre-configured.

**Shadow Credentials?** `msDS-KeyCredentialLink` is empty. No Key Trust abuse possible.

**MSSQL?** There's a `mssqlsvc` account in the domain. Tried authenticating — login failed. No SPN set for Kerberoasting either.

**SYSVOL/GPP?** Two default GPOs, both clean. No `Groups.xml`, no cPassword, no leftover scripts with credentials.

**User descriptions?** Nothing beyond the built-in defaults.

**Managed Service Accounts?** `Get-ADServiceAccount -Filter *` returns absolutely nothing.

At this point I'd been at it for hours and I was running out of ideas. Everything looked default. Everything looked locked down. I was starting to think I'd missed something obvious in the initial foothold.

### The Pivot: Thinking Outbound

Here's where I made my first real mistake — or rather, where fixing a mistake unlocked the box.

My ACL enumeration was **inbound-focused**. I was asking "who has interesting permissions on Domain Admins? On the Administrators group? On DC01?" The answer was always the same: Domain Admins, Enterprise Admins, SYSTEM. Default stuff. Nothing custom, nothing abusable.

The question I should've asked from the start: **"What can MY user or MY group do on ANY object in the domain?"**

I uploaded PowerView and ran:

```powershell
Find-InterestingDomainAcl -ResolveGUIDs | ? {$_.IdentityReferenceName -match "IT|<myuser>"}
```

One result. My group has **CreateChild on an OU containing all the staff users.**

Now, in a normal Active Directory environment, CreateChild on an OU isn't particularly scary. You can create user objects, computer objects, maybe some group objects. Annoying for defenders, but not domain-compromise-level.

Unless you're running Windows Server 2025.

---

## Understanding the Attack

I need to explain what BadSuccessor is, because when I first read about it back when Akamai published the research, I thought "cool, but niche." Now I was staring at the exact scenario it was designed for.

Windows Server 2025 introduced a new object type: **delegated Managed Service Accounts (dMSA)**. The idea is that you can "migrate" an old service account to a new dMSA, and the dMSA inherits the old account's security context. The KDC (Key Distribution Center — the part of Active Directory that handles Kerberos authentication) sees the dMSA, checks which account it "succeeded," and grants it a ticket with that account's privileges.

The vulnerability is absurdly simple: **the KDC doesn't verify that a real migration actually happened.** It just reads an attribute on the dMSA object (`msDS-ManagedAccountPrecededByLink`), sees a Distinguished Name pointing to, say, the Administrator account, and goes "ah yes, legitimate migration, here's a TGT with Administrator privileges."

No verification. No audit trail check. No confirmation from the source account. Just blind trust in an LDAP attribute.

And who can set that attribute? **Whoever creates the dMSA object.** Which means anyone with CreateChild on an OU can:

1. Create a dMSA
2. Point its predecessor to any account in the domain
3. Get a Kerberos ticket with that account's full privileges

91% of environments have non-admin users with CreateChild somewhere. Mine was one of them.

---

## The Execution (Where Everything Went Wrong Before Going Right)

Knowing the theory and executing the attack are two very different things. This is where my three days went.

### Round 1: Looking for a Tool That Doesn't Exist

My first instinct was to find a ready-made exploit. I cloned the Akamai BadSuccessor repo from GitHub. Opened the directory. One PowerShell script: `Get-BadSuccessorOUPermissions.ps1`. That's a **detection** script for defenders. Not an exploit. I should've read the README before getting excited.

### Round 2: Creating the Wrong Object Type

I tried `New-ADServiceAccount` from PowerShell. It created an object, but with the wrong class — `msDS-GroupManagedServiceAccount` (gMSA), not `msDS-DelegatedManagedServiceAccount` (dMSA). These are completely different things. A gMSA is the older type from Server 2012. A dMSA is the new Server 2025 type that BadSuccessor abuses.

I only realized this after inspecting the object properties and seeing the wrong ObjectClass. Wrong object, wrong attack.

### Round 3: Missing Mandatory Attributes

Switched to raw ADSI to create a proper dMSA. `SetInfo()` failed with a constraint violation. I was missing mandatory attributes. Queried the schema:

```powershell
$schema = [ADSI]"LDAP://CN=ms-DS-Delegated-Managed-Service-Account,$((Get-ADRootDSE).schemaNamingContext)"
$schema | fl systemMustContain
```

```
systemMustContain: {msDS-DelegatedMSAState, msDS-ManagedPasswordInterval}
```

Two required attributes I wasn't setting. Added them — object created successfully.

### Round 4: The One-Shot Rule

Object created. Now I needed to set the predecessor link pointing to the Administrator. Tried modifying the existing object:

```
Access is denied.
```

With CreateChild permissions, you have full control over attributes **during the initial creation**. But once the object exists, modifying certain sensitive attributes requires different permissions that I don't have.

The solution: pack **everything** into a single creation call. All attributes — including the predecessor link — set before the first and only `SetInfo()`. One shot is all you get.

This worked. Object created, predecessor pointing to Administrator, `msDS-DelegatedMSAState` set to 2 (active). Verified it all. The dMSA looked exactly like what BadSuccessor needs.

### Round 5: Can't Read the Password

The dMSA exists and points to Administrator. The KDC should honor it. But I need to actually *use* it — either read its managed password or request a Kerberos ticket.

I couldn't do either. `msDS-ManagedPassword` returned null because `msDS-GroupMSAMembership` (which controls who can read the managed password) wasn't set, and I couldn't set it due to the same access-denied issue.

I tried workarounds: scheduled tasks as SYSTEM (no permissions), direct Kerberos with impacket (principal unknown without credentials). Manual PowerShell was a dead end for the exploitation phase. I needed a tool that handles the full Kerberos flow.

### Round 6: Network Architecture Problems

Here's something that cost me more time than I'd like to admit: **LDAP and Kerberos are filtered from outside the box.** Only WinRM is accessible externally.

I didn't check this early enough. I installed bloodyAD on Kali, pointed it at the DC, and got connection timeouts. Then I realized with nmap that ports 88, 389, and 636 weren't reachable from my attack box.

Solution: reverse tunnel with chisel through Evil-WinRM. Forward Kerberos (88), LDAP (389), and LDAPS (636) back to Kali.

But even the tunnel had issues. The DC enforces LDAP signing, so plain LDAP through the tunnel gets rejected. And chisel sessions kept dying, requiring restarts at the worst possible moments. My `/etc/hosts` had stale entries pointing to old IPs from previous box spawns. Each of these things individually takes 5 minutes to fix. Together, in a frustration spiral at 1 AM, they eat hours.

### Round 7: The Finish Line

With a stable tunnel and bloodyAD connected, the actual exploitation was one command:

```bash
bloodyAD -d <domain> -u '<user>' -p '<password>' \
  --host 127.0.0.1 add badSuccessor myDMSA --prepatch
```

The `--prepatch` flag matters. Without it, bloodyAD tries to write new attributes (`msDS-SupersededManagedAccountLink`) on the *target* account — that's post-patch behavior that requires elevated privileges. With `--prepatch`, it uses only the original mechanism on the dMSA itself, which we control.

bloodyAD created the dMSA, set the predecessor, requested a TGT, and extracted the keys. In the "previous keys" field of the TGS response was the Administrator's NT hash. The KDC just handed it over because it saw the predecessor link and assumed everything was legitimate.

Pass-the-hash with Evil-WinRM. Root flag on the Administrator's desktop.

---

## What Three Days Taught Me

The optimal path for this box, from first scan to root, is maybe 16 minutes. I took three days. That ratio feels bad until I think about what those three days actually bought me.

**Enumerate outbound, not just inbound.** My instinct from years of sysadmin/development work was to protect high-value assets — check who has access to the admin groups. Offensive thinking is the opposite: check what your compromised identity can touch, anywhere, on anything. `Find-InterestingDomainAcl` filtered by my own identity should've been my first command. It would've given me the answer in seconds.

**Server 2025 changes the game.** CreateChild on an OU used to be a minor finding. Now it's a domain compromise primitive. Any environment with even one Server 2025 DC and a KDS Root Key is potentially vulnerable. As someone coming from OT where industrial controllers run software from 2008, this is a reminder that the cutting edge of AD security moves fast.

**New attacks have messy tooling.** BadSuccessor was published in May 2025. Nine months later, the tooling ecosystem is still fragmented. Akamai's repo is detection-only. Impacket has support but with different syntax. bloodyAD has the cleanest implementation. In another year this will all be standardized. Right now, you need to understand the underlying attack well enough to troubleshoot the tools themselves.

**The one-shot rule for CreateChild.** This one is going in my permanent notes. When you have CreateChild-only permissions, you get one chance during the initial `SetInfo()`. Post-creation modifications on sensitive attributes will fail. Design your object creation to include everything upfront.

**Check port accessibility before choosing tools.** I wasted time running tools from Kali that needed LDAP and Kerberos, only to discover those ports were filtered. Five minutes of nmap upfront would've told me I needed a tunnel, and I could've had it running before I started the exploitation phase.

**I understand dMSA internals now.** Not just "run this tool" — I know what `msDS-DelegatedMSAState` does, why `msDS-ManagedAccountPrecededByLink` is dangerous, how the KDC processes these attributes during ticket issuance, and why the "previous keys" field in the TGS contains the predecessor's NT hash. That understanding doesn't come from reading a blog post. It comes from failing six times and figuring out why each time.

I came into this box as an OT developer who's learning pentesting. I came out of it understanding a Server 2025-specific AD attack class at a level that would let me explain it, detect it, and exploit it. Three days well spent.

---

## Tools

| Tool | Purpose |
|---|---|
| NetExec | Password spraying |
| Evil-WinRM | Shell access |
| PowerView | ACL enumeration |
| Chisel | Reverse port forwarding (88, 389, 636) |
| bloodyAD | BadSuccessor exploitation |

## References

- [Akamai — BadSuccessor Research](https://www.akamai.com/blog/security-research/abusing-dmsa-for-privilege-escalation-in-active-directory)
- [SpecterOps — Understanding & Mitigating BadSuccessor](https://specterops.io/blog/2025/05/27/understanding-mitigating-badsuccessor/)
- [Unit42 — When Good Accounts Go Bad](https://unit42.paloaltonetworks.com/badsuccessor-attack-vector/)
- [bloodyAD](https://github.com/CravateRouge/bloodyAD)
