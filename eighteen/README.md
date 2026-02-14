# HTB Easy - Eighteen — BadSuccessor on Server 2025

> I work in industrial OT/IT integration during the day while I'm teaching myself offensive security in the evenings. This is one of those — well, three of them.
> If you're here for a speedrun, this isn't it. If you want to see someone figure out a new attack class in real time, including all the dumb mistakes, keep reading^

---

## The Setup

Single Windows Server 2025 domain controller. One machine, one domain. EZ. I spent three evenings on that, after work — maybe 8/10 hours total.
The user flag took me minutes. The root flag took the other two and a half day.

The attack path itself is almost obvius, AD, controller... some kind of ticket, hash or something.

---

## Getting In

Had a user list from the early stage and two candidate passwords. Password spray with NetExec against WinRM:

```bash
nxc winrm <target> -u users.txt -p '<password>'
```

One hit. Shell with Evil-WinRM. User flag on the desktop. Cool.

---

## Hitting the Wall (hard)

My compromised user is in the **IT** group. No privileges, no admin, nothing actually obvious. Std-checklist:

- Kerberoastable SPNs → only `krbtgt`, nope
- Unconstrained delegation → only DC01 itself, default
- RBCD → empty
- Shadow Credentials → empty
- MSSQL → can't authenticate
- SYSVOL → clean, default GPOs only
- User descriptions → nothing
- Managed Service Accounts → none exist

Everything looked locked. First evening ended here. Wednesday, almost weekend.

### Second Evening: The Right Question

Came back the next evening and realized my ACL enumeration was backwards.
I'd been checking who has permissions *on* high-value targets — Domain Admins, Administrators, DC01.
That's the naive mindset from my day job leaking through.

Flipped it. Uploaded PowerView:

```powershell
Find-InterestingDomainAcl -ResolveGUIDs | ? {$_.IdentityReferenceName -match "IT|<myuser>"}
```

One result: **IT group has CreateChild on an OU.**

On a normal domain, whatever. On Windows Server 2025 — comes up that's BadSuccessor domain. Good. PWND. Not yet.

---

## BadSuccessor: The Short Version

Server 2025 introduced delegated Managed Service Accounts (dMSA).
You can "migrate" an old service account to a dMSA, and the dMSA inherits the old account's privileges through Kerberos.

The issue: seems likely that KDC doesn't verify that a real migration happened (?)
It reads an attribute on the dMSA (`msDS-ManagedAccountPrecededByLink`), sees it pointing to the Administrator account, and hands over a TGT with Administrator privileges.

No questions asked.

Anyone with CreateChild on an OU can create a dMSA and point it at whatever account they want. That's the whole attack. Good. PWND?

---

## Hitting the wall

Theory is super clear. Execution much less. Here's the condensed version of two evenings of trial and error.

**Wrong tool.** Cloned the Akamai BadSuccessor repo expecting an exploit. It's a detection script. Should've read the README :)

**Wrong object type.** Used `New-ADServiceAccount` which creates gMSAs, not dMSAs. Different object class, different attack. Wasted time before checking `ObjectClass` in the output.

**Missing attributes.** ADSI creation failed until I queried the schema and found two mandatory attributes: `msDS-DelegatedMSAState` and `msDS-ManagedPasswordInterval`.

**The one-shot thing.** With CreateChild permissions, you can set any attribute during the initial object creation. After that, modifying sensitive attributes is denied. I found this out the hard way — created the dMSA, then tried to set the predecessor link separately. Access denied. Had to delete and recreate with everything packed into a single `SetInfo()` call.

>Note on Permissions: This happens because CreateChild on an OU allows you to define the initial state of an object (including sensitive attributes like msDS-ManagedAccountPrecededByLink), but it does not grant WriteProperty or GenericAll over the object once it is "born." Unless the ACEs are set to inherit down to the new child object, you lose control the moment the creation is finalized.

**Network issues.** LDAP and Kerberos are filtered from outside the box. Only WinRM is exposed. I knew it, I ignored it. Set up a chisel reverse tunnel to forward ports 88, 389, and 636 through WinRM. Then LDAP signing kicked in and broke plain LDAP. Then the tunnel kept dying. 1 AM debugging spiral.

**What actually worked.** bloodyAD through the chisel tunnel:

```bash
bloodyAD -d <domain> -u '<user>' -p '<password>' \
  --host 127.0.0.1 add badSuccessor myDMSA --prepatch
```

The `--prepatch` flag skips writing new post-patch attributes on the target account (which requires privileges we don't have) and uses only the original `msDS-ManagedAccountPrecededByLink` on the dMSA.

bloodyAD created the dMSA, set the predecessor, got a TGT, and pulled the Administrator's NT hash from the "previous keys" in the TGS response. Pass-the-hash. Root.

---

## Notes to Self

- `Find-InterestingDomainAcl` filtered by my identity — always run this first, not last.
- Server 2025 + CreateChild on any OU = check BadSuccessor (immediately)
- gMSA ≠ dMSA. Different object classes, different cmdlets (!!!)
- CreateChild = one shot jic. Pack everything into the initial `SetInfo()`.
- Don't ignore notes on port accessibility while looking for tools. Would've saved me the whole tunnel debugging session.
- bloodyAD with `--prepatch` I think (but I've not enaugh cs) is currently the cleanest way to run BadSuccessor from Linux.
- When tunneling AD traffic, expect LDAP signing issues.

Ok. Next.

---

## Tools

| Tool | Purpose |
|---|---|
| NetExec | Password spraying |
| Evil-WinRM | Shell access |
| PowerView | ACL enumeration |
| Chisel | Reverse port forwarding |
| bloodyAD | BadSuccessor exploitation |

## References

- [Akamai — BadSuccessor Research](https://www.akamai.com/blog/security-research/abusing-dmsa-for-privilege-escalation-in-active-directory)
- [SpecterOps — Understanding & Mitigating BadSuccessor](https://specterops.io/blog/2025/05/27/understanding-mitigating-badsuccessor/)
- [Unit42 — When Good Accounts Go Bad](https://unit42.paloaltonetworks.com/badsuccessor-attack-vector/)
- [bloodyAD](https://github.com/CravateRouge/bloodyAD)
