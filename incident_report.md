# Security Incident Report
**SSH Brute Force Attack & Credential Compromise**

Prepared by: [Kiante Nolen](https://github.com/CodeBroKinty) | CodeBroKinty  | April 30, 2026

---

## 1. Incident Summary

| Field | Detail |
|-------|--------|
| **Incident ID** | INC-2026-001 |
| **Date Detected** | April 30, 2026 — 02:54 AM |
| **Detection Method** | SPL query on Splunk SIEM — failed SSH authentication spike |
| **Severity** | HIGH |
| **Status** | Simulated / Lab Investigation — Closed |
| **Analyst** | Kiante Nolen (CodeBroKinty) |
| **Environment** | Homelab — Splunk Enterprise Free Trial |

---

## 2. Attack Timeline

| Time | Event |
|------|-------|
| Apr 29, 2026 14:00 | 2 events from `194.8.74.23` — initial reconnaissance probe against SSH port 22 |
| Apr 29, 14:01 – Apr 30, 01:59 | 12+ hours of silence — attacker waits for off-hours window |
| Apr 30, 2026 02:00 | 132 events in a single hour — full brute force assault launched |
| Apr 30, 02:54 | `nsharpe`: Accepted password from `10.2.10.163` — credential compromise confirmed |
| Apr 30, 02:54 | `nsharpe` executes `su` to root — privilege escalation confirmed |
| Apr 30, 02:54 | `djohnson`: Accepted password from `10.3.10.46` — second account compromised |
| Apr 30, 02:54 | Root opens multiple sessions for `djohnson` — lateral movement indicator |

---

## 3. Technical Findings

### 3.1 Attacker Profile

| Field | Detail |
|-------|--------|
| **Source IP** | `194.8.74.23` (external) |
| **Target** | `mailsv1`, `www1`, `www2`, `www3` — SSH port 22 |
| **Total Events** | 313 events across 7 log sources |
| **Attack Type** | SSH dictionary brute force using Linux system account wordlist |
| **Attack Window** | Concentrated burst — 132 events at 02:00 AM |

---

### 3.2 Targeted Usernames (Top 10)

| Username | Attempts | Notes |
|----------|----------|-------|
| `local` | 6 | |
| `system` | 6 | |
| `irc` | 5 | |
| `root` | 4 | ⚠️ HIGH RISK — full system access if successful |
| `nagios` | 4 | |
| `games` | 4 | |
| `admin` | 3 | |
| `apache` | 3 | |
| `bin` | 3 | |
| `vpxuser` | 3 | |

> **Assessment:** Attacker used a standard Linux/Unix credential wordlist targeting default system accounts. This is a non-targeted, opportunistic attack. The presence of `root`, `apache`, and `nagios` in the wordlist indicates intent to gain either direct root access or service account access for persistence.

---

### 3.3 Compromised Accounts

| Account | Finding |
|---------|---------|
| `nsharpe` | Multiple `Accepted password` events from `10.2.10.163` at 02:54 AM |
| `nsharpe` | Executed `su` to root — `uid=0` session opened — **PRIVILEGE ESCALATION CONFIRMED** |
| `djohnson` | Multiple `Accepted password` events from `10.3.10.46` at 02:54 AM |
| `djohnson` | Root (`uid=0`) opened multiple sessions on their behalf — lateral movement |
| Internal IPs | Two separate internal IPs (`10.2.10.163`, `10.3.10.46`) — possible pivot from external attack |

---

## 4. SPL Queries Used

### Query 1 — Attack Scope
```spl
index=main "194.8.74.23" | stats count by source
```
**Purpose:** Quantify total attack events across all log sources to determine infrastructure-wide impact.

---

### Query 2 — Brute Force Username Enumeration
```spl
index=main "Failed password" "194.8.74.23"
| rex "Failed password for (invalid user )?(?<username>\S+) from"
| stats count by username
| sort -count
```
**Purpose:** Extract and rank all usernames attempted by the attacker to identify wordlist type and high-risk targets.

---

### Query 3 — Attack Timeline
```spl
index=main "194.8.74.23" earliest=-1d | timechart span=1h count
```
**Purpose:** Visualize attack volume over time to identify probe vs. burst pattern and confirm off-hours timing.

---

### Query 4 — Compromised Account Activity
```spl
index=main "djohnson" OR "nsharpe"
| table _time, source, _raw
| sort _time
```
**Purpose:** Investigate user activity during and after the attack window to confirm credential compromise and privilege escalation.

---

## 5. Incident Assessment

### What Happened
An external threat actor at IP `194.8.74.23` conducted a two-phase SSH brute force attack against the Homelab infrastructure. The attacker performed an initial reconnaissance probe at 14:00 on April 29, then launched a concentrated credential stuffing assault at 02:00 AM on April 30 — a deliberate off-hours timing strategy. The attack successfully compromised two user accounts (`nsharpe` and `djohnson`) on `www1`, with `nsharpe` subsequently escalating privileges to root.

### Attack Chain
```
194.8.74.23 (external attacker)
    └── Probes SSH port 22 across mailsv, www1, www2, www3
    └── Brute force wordlist targets 20+ Linux system and service accounts
    └── Credentials for nsharpe and djohnson successfully guessed
         └── nsharpe authenticates from 10.2.10.163 → escalates to root via su
         └── djohnson authenticates from 10.3.10.46 → root opens sessions on their behalf
         └── Two separate internal IPs suggest lateral movement or prior internal compromise
```

---

## 6. Escalation & Recommendations

### What I Would Escalate

**Immediate:**
- Block `194.8.74.23` at the firewall perimeter
- Disable and lock `nsharpe` and `djohnson` accounts pending investigation
- Audit all root sessions opened by `uid=0` on `www1` for persistence mechanisms (cron jobs, SSH keys, new accounts)

**Urgent:**
- Investigate internal IPs `10.2.10.163` and `10.3.10.46` — determine if these are legitimate workstations or already compromised hosts
- Review all sudo and su logs across all servers for the full 24-hour window

---

### Preventive Controls

| Control | Purpose |
|---------|---------|
| Disable password-based SSH auth — enforce key pairs only | Eliminates brute force as an attack vector entirely |
| Implement `fail2ban` — auto-block after 3-5 failed attempts | Kills attack speed before wordlist can be exhausted |
| Restrict SSH to known internal IPs via firewall rules | Prevents external attackers from reaching SSH port |
| Set `PermitRootLogin no` in `sshd_config` | Prevents direct root login even with valid credentials |
| Enable MFA for all privileged accounts | Single stolen credential is not enough to authenticate |
| Splunk alert: >5 failed SSH attempts from single IP within 5 min | Real-time detection instead of after-the-fact discovery |

---

*CodeBroKinty Portfolio Lab | [github.com/CodeBroKinty/splunk-siem-lab](https://github.com/CodeBroKinty/splunk-siem-lab) | Splunk SIEM Project*