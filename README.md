# Week 11 - SOC Threat Detection, Alert Correlation & Incident Response

**Cyber Security Internship | DG Interns Hub**
**Author:** Mah Noor | **Date:** 5 November 2026 | **Incident:** DG-SOC-001

A SOC lab where alerts from **Snort** (network IDS) and **Wazuh** (host SIEM) are correlated to investigate failed SSH logins followed by a successful login. All testing was done in my own isolated VirtualBox lab.

---

## Repository Contents

```
Week11/
├── README.md
├── DG_Week11_SOC_Report.pdf            (25-page report)
├── DG_Week11_SOC_Presentation.pptx     (10 slides)
└── Week11_Screenshots/
    ├── 01_Lab_Setup/
    ├── 02_Snort_Monitoring/
    ├── 03_SSH_Investigation/
    ├── 04_Wazuh_Monitoring/
    ├── 05_Week10_Reference/
    └── 06_Troubleshooting/
```

## Lab Environment

| Machine | Role | IP |
|---|---|---|
| Windows host (PowerShell) | Test / attacker machine | 192.168.56.1 |
| soc-snort (Ubuntu 24.04 LTS) | Target, Snort IDS + Wazuh agent (shown as `snort-agent`) | 192.168.56.102 (enp0s8) |
| wazuh-server (Wazuh 4.14.9 OVA) | Manager, indexer, dashboard | 192.168.56.103 |

Time: Linux logs are in **UTC**. Pakistan time = UTC+5. Wazuh timestamps are about 22 seconds behind the Linux logs (VM clock drift after resume).

## Tasks Covered

| Task | Where |
|---|---|
| 1. Alert correlation notes | Report, Section 2 |
| 2. Wazuh monitoring | Report, Section 3 |
| 3. Snort + Wazuh comparison | Report, Sections 4-5 |
| 4. SOC investigation | Report, Section 7 |
| 5. IOC identification | Report, Section 8 |
| 6. Alert classification (16 alerts) | Report, Section 6 |
| 7. Incident response | Report, Section 9 |
| 8. Incident ticket DG-SOC-001 | Report, Section 10 |
| 9. Three playbooks | Report, Section 11 |

## Test Steps (own lab only)

From Windows PowerShell:

```powershell
ping 192.168.56.102 -n 10
nmap -sS -p 1-1000 192.168.56.102
ssh wronguser@192.168.56.102        # wrong password several times
ssh snortadmin@192.168.56.102       # wrong password, then the correct one
```

On soc-snort:

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i enp0s8
sudo grep -E "Failed password|Invalid user|Accepted password" /var/log/auth.log | tail -20
```

## Key Findings

- **Snort:** 10 pings detected (SID `1:1000001`). The nmap scan was detected indirectly through default SNMP signatures `1:1418`, `1:1420`, `1:1421`. The custom port-scan rule (SID 1000002) did not fire. No Snort alert for the encrypted SSH logins was observed.
- **Wazuh:** 11 level-5 alerts (rule `5710` x9, `5760` x2), one level-10 brute-force alert, and correlation rule **`40112` (level 12)** for failures followed by a success. MITRE: Brute Force, Valid Accounts.
- **Incident:** source `192.168.56.1`, accounts `wronguser` (invalid) and `snortadmin` (valid), 8 failed passwords, then **Accepted password at 09:10:59 UTC**. After login only `whoami`, `ls`, `exit`.
- **Verdict:** True Positive detection of an authorized lab simulation. Severity: High.
- **Other:** unexpected open port 80/tcp, 16 pending Ubuntu updates.

## IOC Summary

| Type | Value |
|---|---|
| IP | 192.168.56.1 (source), 192.168.56.102 (target) |
| Usernames | wronguser, snortadmin |
| Files | /var/log/auth.log, /tmp/fim_test.txt (Week 10 FIM) |
| Ports | 22, 80, 161, 162, 705 |
| Protocols | ICMP, TCP, SSH |
| Wazuh rules | 5710, 5760, 40112 |
| Snort SIDs | 1:1000001, 1:1418, 1:1420, 1:1421 |
| Window | 2026-11-05, 09:03:42 - 09:10:59 UTC |

## Limitations (honest notes)

- No new file-integrity alert (rules 550/553/554) appeared in this run. The Week 10 FIM alert is included as reference (`05_Week10_Reference`).
- Rule `5715` returned no result; correlation rule `40112` covered the successful login.
- The rule ID of the level-10 brute-force alert was not captured in a screenshot.
- Entries dated 28 Oct 2026 in `auth.log` are earlier tests and are not part of this incident.

## Tools

Snort, Wazuh 4.14.9, OpenSSH, nmap, Windows PowerShell, VirtualBox, Ubuntu 24.04 LTS.

## Disclaimer

All activity was performed on machines I own, inside an isolated host-only network, for learning purposes.
