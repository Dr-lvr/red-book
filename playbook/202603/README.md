# Monthly Bulletin — [Month Year]
> Boxes completed: X | New rules: X | Rules reinforced: X

---

## Boxes This Month
| Box | OS | Difficulty | Days | Key Vector |
|-----|----|-----------|------|------------|
| BoxName | Linux/Windows | Easy/Medium/Hard | X | e.g. CVE-XXXX, Kerberoast |

---

## New Rules / Updates
> Only add a rule if you broke it at least once this month.

### Rule N: [Short title]
[One sentence: what the rule is.]
```bash
# Template / command if applicable
```

**Why**: [The specific box and mistake that generated this rule.]

---

## Patterns Reinforced
> Rules from the existing playbook that came up again this month.

| Rule | Box | How it showed up |
|------|-----|-----------------|
| Rule 1: Hashes → hashcat NOW | BoxName | Found hash at min 10, waited 2h |
| Rule X | BoxName | ... |

---

## Dead Ends Worth Documenting
> Vectors that wasted significant time — for future reference.

- **[Box] — [Vector]:** [What I tried, why it failed, how long it took]

---

## CVEs / Techniques Encountered
| Name | CVE / Ref | Context | Notes |
|------|-----------|---------|-------|
| BadSuccessor | — | Windows Server 2025 AD | dMSA DACL abuse |

---

## Checklist Updates
> Any additions to the landing checklist based on this month's boxes.

- [ ] New item for Linux checklist: ...
- [ ] New item for Windows checklist: ...

---

## Stats
- Total boxes: X (Easy: X / Medium: X / Hard: X)
- Avg time to user: Xh
- Avg time to root: Xh
- Most common failure mode this month: [e.g. "forgot UDP", "didn't spray creds"]

---

*Next month focus: [e.g. "AD boxes only", "medium difficulty minimum", "ADCS practice"]*
