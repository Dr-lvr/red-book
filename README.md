# red-book

I'm a full-stack developer in industrial OT/IT integration — PLCs, SCADA systems, Modbus TCP, factory floors across Pakistan, Brazil, Colombia, Turkey, and Canada. Now I'm learning how to break in on order to protect.

This repository is the honest record of that transition not a portfolio of clean solutions. It's what actually happened: the wrong turns, the walls I hit, the moments I stared at a terminal for 45 minutes before realizing I'd been looking at the answer the whole time.

I only write up what made me think. If a box took an hour and I just executed known steps, it's not here. If it took eight hours and I had to learn something I didn't know existed, it is.

---

## Structure

```
red-book/
├── HTB_easy/           # HackTheBox machines — easy tier
│   ├── eighteen/       # Windows Server 2025, BadSuccessor, 3-day education in AD
│   ├── wingdata/       # Wing FTP Server, 7 dead ends before the obvious answer
│   ├── expressway/     # IKE/IPsec VPN, the rabbit hole that taught me UDP exists
│   └── .../
│
├── playbook/           # Operational methodology — built on mistakes
│   ├── 202602/         # February 2026 snapshot (7 boxes, 9 days)
│   └── .../            # Updated monthly as new patterns emerge
│
└── README.md
```

**HTB_easy/** — Writeups of HackTheBox machines. Each one tells the real story: what I tried, what failed, what I learned. Spoiler-free while boxes are active, published after retirement. The value isn't in the flags — it's in the methodology.

**playbook/** — My operational playbook, extracted from recurring mistakes. Every rule exists because I broke it at least once. Versioned monthly so I can track how my methodology evolves. The February 2026 edition has 10 rules from 7 boxes. By the time I sit for the CPTS exam, it should have 30.
