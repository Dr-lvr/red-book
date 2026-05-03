# Container & VM Breakout

> Last updated: May 2026
> Trigger: spent ~1 hour on a box that was solved by a docker mount I should have seen in the first 15 minutes.

## Why this note exists

Three boxes in a row with container/virtualization layers in the path. On the third one I had everything — shell in the container, enumerated mounts, knew it was Docker — and still kept trying privilege escalation paths inside the container instead of looking outward.

The lesson isn't technical. The technical part I knew. The lesson is that **once I'm in a container, my mental model needs to flip**: the question stops being "how do I become root *here*" and becomes "what does *here* expose about *out there*". I kept asking the wrong question for an hour.

This note exists so next time the flip is automatic.

## The flip — first thing to ask after landing in any shell

Before any other enumeration, answer one question: **am I in a container/VM, and if so, where's the boundary leaking?**

```bash
# Quick container check — pick one, they all work
ls -la /.dockerenv 2>/dev/null
cat /proc/1/cgroup | grep -E "docker|lxc|kubepods"
cat /proc/self/status | grep -i container
systemd-detect-virt
```

If the answer is "yes, I'm in a container", the playbook isn't `linpeas` first. The playbook is **mount enumeration first**.

```bash
mount
cat /proc/mounts
findmnt
ls -la /  # any unusual top-level dirs? /host_root, /mnt, /data, anything mounted from outside
```

The container is just a process tree with namespaces. Anything mounted from the host into the container is the bridge. **Find the bridge before trying to break the walls.**

## The four breakout patterns I keep seeing

### 1. Privileged container

If the container was started with `--privileged`, capabilities are wide open. Check:

```bash
capsh --print
cat /proc/self/status | grep CapEff
```

`CapEff: 0000003fffffffff` (or similar all-ones) → privileged. From here `mknod` + raw device access on `/dev/sda` reads the host filesystem directly, no questions asked.

### 2. Docker socket exposed

The classic one. If `/var/run/docker.sock` is mounted into the container, the container can talk to the Docker daemon on the host. Check:

```bash
ls -la /var/run/docker.sock
# if it's there: game over, you control Docker on the host
docker -H unix:///var/run/docker.sock ps
docker -H unix:///var/run/docker.sock run -it -v /:/host alpine chroot /host /bin/bash
```

This was the third box. I should have seen the socket in the first mount enumeration. I didn't because I was looking at file permissions inside the container instead of looking at what came from outside.

### 3. Sensitive host paths mounted in

Anything from the host filesystem mounted as a volume is a potential read or write into the host. The dangerous ones:

- `/` mounted anywhere → trivial host access via the mount point
- `/etc` mounted → SSH keys, shadow, sudoers
- `/root` or `/home` mounted → SSH keys, history, secrets
- `/var/log` mounted → logs to poison or read
- `/proc` mounted from host → host process enumeration

Inverse pattern: a volume that *looks* internal but is actually a bind mount. `findmnt -T <path>` resolves where it actually lives.

### 4. Capabilities without full privilege

Not `--privileged` but with specific capabilities granted: `CAP_SYS_ADMIN`, `CAP_SYS_PTRACE`, `CAP_DAC_READ_SEARCH`, `CAP_NET_ADMIN`. Each has its own breakout path:

- `CAP_SYS_ADMIN` → mount filesystems, including the host's
- `CAP_SYS_PTRACE` → attach to host processes if PID namespace is shared
- `CAP_DAC_READ_SEARCH` → read any file on shared mounts regardless of permissions
- `CAP_NET_ADMIN` → manipulate network, potentially reach host services

The check is the same `CapEff` field as before, but you decode it instead of looking for all-ones. `capsh --decode=<hex>` does it for you.

## VM breakout — different problem, similar mindset

VM breakout is a different beast (hypervisor escapes, VENOM-class bugs, guest-tools exploitation) and outside HTB easy/medium scope 99% of the time. But the **mindset flip is the same**: once you know you're in a VM, the question becomes about the boundary, not about elevating inside.

Quick VM check:

```bash
systemd-detect-virt
dmidecode -s system-product-name 2>/dev/null  # VMware, VirtualBox, KVM, etc.
lscpu | grep -i hypervisor
```

If it's a VM on a box, the breakout is almost never the intended path. But guest-tools (vmtoolsd, VirtualBox Guest Additions) running with elevated privileges and exposed via an IPC channel have been historic vectors. Worth a `ps aux | grep -iE "vmtools|vbox|qemu"` just to log it.

## The rule that goes in the playbook

> **Rule 11 — When you land in a shell, identify the layer before enumerating the host.**
> Container vs VM vs bare metal changes which enumeration matters. Ten seconds of layer detection saves an hour of wrong-direction privesc. The mount table tells you more about your situation than the user's home directory does.

## What I want to drill

Three exercises for the next two weeks, in order:

1. **Re-do the third box from scratch**, this time forcing myself to enumerate mounts before anything else. Time goal: shell to host root in under 25 minutes.
2. **Build a small docker breakout lab locally** — three containers with the four patterns above (privileged, socket-mounted, host-path-mounted, cap-granted) and chain breakouts on each until the technique is muscle memory.
3. **Read one paper or write-up per week on real-world container escapes** — start with the runC CVE-2019-5736 write-up and the Docker copy-on-write breakout chain. Pattern recognition for what production breakouts look like vs CTF breakouts.

## References to come back to

- HackTricks: Docker Breakout / Privilege Escalation page (the canonical reference, dense)
- `linpeas` container module (good for confirming, bad for learning — never run it first)
- DEEPCE (Docker Enumeration, Escalation of Privileges and Container Escapes) — the focused tool, worth reading the source
- Trail of Bits "Understanding Docker container escapes" — clearer than HackTricks for the mental model
- runC CVE-2019-5736 write-up by Adam Iwaniuk — the breakout that changed how runtimes are written

---

*This entry is here because Box 3 cost me an hour I didn't need to spend. The technical knowledge was already in my head. The orientation wasn't. Writing this down is the orientation moving from passive to active.*
