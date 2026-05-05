# Incident Report — IR-2026-001

**Date:** May 1, 2026
**Analyst:** Kiante Nolen
**Severity:** Medium
**Status:** Resolved (Lab Environment)
**Report Type:** Security Lab Documentation — Portfolio Project 3

---

## 1. Incident Summary

Multiple failed authentication attempts were detected against the Windows victim machine (`windows-victim`, Agent ID: 001) monitored by the Wazuh EDR platform. The Wazuh manager, running on Parrot OS, successfully detected and alerted on repeated NTLM-based network logon failures targeting the local Administrator account. A total of 16 Rule 60122 alerts were generated over the course of the lab session.

Additionally, the Wazuh Security Configuration Assessment (SCA) module automatically ran a CIS Microsoft Windows 11 Enterprise Benchmark scan upon agent registration, identifying 261 failed controls and producing a 32% compliance score — providing a real-time posture baseline for the endpoint.

---

## 2. Environment

| Component | Details |
|---|---|
| EDR Platform | Wazuh v4.7.5 |
| Manager OS | Parrot OS (VirtualBox VM) |
| Manager IP | 192.168.56.101 |
| Agent Name | windows-victim |
| Agent ID | 001 |
| Agent OS | Microsoft Windows 11 |
| Agent IP | 192.168.56.102 |
| Network | VirtualBox Host-Only (vboxnet0) |
| Cluster Node | node01 |
| Registration Date | May 1, 2026 @ 10:35:44 |

---

## 3. Detection Source

| Field | Value |
|---|---|
| Platform | Wazuh SIEM/EDR v4.7.5 |
| Index | wazuh-alerts-4.x-2026.05.01 |
| Rule ID | 60122 |
| Rule Description | Logon failure — Unknown user or bad password |
| Rule Level | 5 (Medium) |
| MITRE Techniques | T1078 (Valid Accounts), T1531 (Account Access Removal) |
| MITRE Tactics | Defense Evasion, Persistence, Privilege Escalation, Initial Access, Impact |

---

## 4. Timeline of Events

| Time (CDT) | Event |
|---|---|
| 10:35:44 | Wazuh agent registered on windows-victim |
| 10:36:58 | SCA scan completed — CIS Windows 11 Benchmark (32% score) |
| 10:40:05 | Last confirmed agent keepalive |
| 10:44:29 | First Rule 60122 alert — failed logon attempt (Administrator) |
| 10:45:04 | Second failed logon attempt detected |
| 10:51:24 | Additional failed logon attempts detected |
| 10:52:00 | Continued brute force pattern confirmed |
| 10:56:41 | Failed logon burst — multiple alerts within seconds |
| 11:00:41 | Additional Rule 60122 alerts generated |
| 11:01:17 | Final detected logon failure in session |
| 11:08:58 | Dashboard confirmed 16 total Rule 60122 hits |

---

## 5. Indicators of Compromise (IOCs)

| IOC Type | Value |
|---|---|
| Targeted Account | Administrator (local) |
| Source Address | ::1 (IPv6 loopback — simulated internal) |
| Source Port | 63571 |
| Authentication Package | NTLM |
| Logon Type | 3 (Network) |
| Logon Process | NtLmSsp |
| Failure Status Code | 0xc000006d (wrong username or password) |
| Failure SubStatus | 0xc000006a (wrong password) |
| Failure Reason | %%2313 (Unknown user name or bad password) |
| Subject SID | S-1-0-0 (Null SID — unauthenticated source) |
| Total Alert Count | 16 (Rule 60122) |

---

## 6. Technical Analysis

The authentication failures match the behavioral pattern of a **brute-force or password spraying attack** targeting the local Windows Administrator account via NTLM network logon (Logon Type 3). Key forensic indicators support this assessment:

- **Status 0xc000006d** confirms the logon was rejected due to invalid credentials
- **SubStatus 0xc000006a** specifically indicates the password was incorrect (as opposed to an unknown username)
- **NtLmSsp** as the logon process confirms NTLM challenge-response was used, consistent with lateral movement tooling or `net use` commands
- **S-1-0-0 Subject SID** confirms the attempts originated from an unauthenticated security context
- The **loopback source address (::1)** indicates the attack was simulated locally using `net use \\localhost\IPC$` — consistent with controlled lab conditions

The SCA module independently identified 261 failed CIS benchmark controls on the endpoint, indicating significant security hardening gaps that would increase attack surface in a real environment.

No lateral movement, persistence mechanisms, or data exfiltration indicators were observed. This was a controlled test in an isolated lab environment.

---

## 7. Response Actions

- [x] Deployed Wazuh manager on Parrot OS (VirtualBox)
- [x] Enrolled Windows victim machine as Agent 001
- [x] Confirmed agent Active status in Wazuh dashboard
- [x] Enabled Windows audit policy for logon failure events (`auditpol /set /subcategory:"Logon" /failure:enable`)
- [x] Triggered and confirmed 16x Rule 60122 alerts via simulated brute force
- [x] Reviewed alert detail including MITRE tagging, failure codes, and targeted account
- [x] Documented SCA baseline scan results (32% CIS compliance)
- [x] Captured all evidence screenshots for documentation

---

## 8. Recommendations

1. **Implement account lockout policy** — lock after 5 failed attempts to stop brute force
2. **Enable Wazuh Active Response** — auto-block IPs triggering repeated Rule 60122 alerts
3. **Address CIS benchmark failures** — 261 failed controls represent significant hardening gaps
4. **Expand audit policy coverage** — enable Credential Validation and Account Logon subcategories
5. **Enable FIM on sensitive directories** — configure syscheck with shorter scan frequency (e.g., 3600s) for critical paths
6. **Configure alerting integrations** — set up email or Slack notifications for Level 5+ alerts
7. **Disable NTLM where possible** — enforce Kerberos authentication to reduce NTLM attack surface

---

## 9. Evidence Files

| File | Description |
|---|---|
| `wazuh-agent-active-windows-victim.png` | Terminal output confirming Agent 001 Active status |
| `wazuh-dashboard-agent-connected.png` | Dashboard agent page — Active, IP, SCA scan visible |
| `wazuh-dashboard-agent-connected-mid.png` | Security alerts table with Rule 60122 and MITRE tags |
| `wazuh-dashboard-auth-failure-alerts-top.png` | Dashboard summary — 796 events, 7 auth failures |
| `wazuh-rule-60122-logon-failures.png` | Events view — 8 hits filtered by rule.id:60122 |
| `wazuh-alert-detail-60122.png` | Expanded alert detail — Administrator, NTLM, status codes |
| `dashboard-top.png` | Wazuh modules overview |
| `dashboard-bottom.png` | Wazuh compliance and threat detection modules |

---

## 10. References

- Wazuh Rule ID 60122 — Windows Logon Failure
- MITRE ATT&CK T1078 — Valid Accounts
- MITRE ATT&CK T1531 — Account Access Removal
- Windows Event ID 4625 — An account failed to log on
- CIS Microsoft Windows 11 Enterprise Benchmark v1.0.0
- NIST 800-53 — AC-7 (Unsuccessful Logon Attempts)
- CompTIA Security+ SY0-701 — Domain 2: Threats, Vulnerabilities, and Mitigations

---

*Analyst: Kiante Nolen | Date: May 1, 2026*
