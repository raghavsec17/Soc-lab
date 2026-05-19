# 🔍 SOC Home Lab — OpenSearch + Apache NiFi + Threat Analysis

A personal Security Operations Centre (SOC) home lab built on Ubuntu 22.04 LTS, demonstrating real-time log ingestion, centralized storage, dashboard visualization, and Windows Event Log threat analysis.

---

## 🧱 Stack

| Component | Version | Role |
|---|---|---|
| Ubuntu 22.04.5 LTS | Jammy | Host OS (VirtualBox VM) |
| OpenSearch | 2.19.1 | Log storage and full-text search |
| OpenSearch Dashboards | 2.19.1 | Visualization and threat hunting UI |
| Apache NiFi | 1.28.1 | Data pipeline and log routing |
| Python 3 + python-evtx | — | EVTX log conversion and parsing |

---

##  Architecture

```
Syslog Source (UDP 5514)
        │
        ▼
┌──────────────────┐
│  Apache NiFi     │  ListenSyslog → PutFile
│  (port 8080)     │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  OpenSearch      │  REST API (port 9200)
│  syslog-logs     │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  OpenSearch      │  Dashboards UI (port 5601)
│  Dashboards      │
└──────────────────┘
```

---

## 📁 Repository Structure

```
Soc-lab/
├── report/
│   └── Assessment_Report.pdf
├── evtx-logs/
│   ├── attack_1.evtx       # Local group enumeration
│   ├── attack_2.evtx       # Kerberos enumeration + log wipe
│   └── attack_3.evtx       # MSSQL brute force
└── screenshots/
    └── *.png               # Setup, pipeline, dashboards, analysis
```

---

## ⚙️ What Was Built

### 1. Environment Setup
- Deployed Ubuntu 22.04.5 LTS on Oracle VirtualBox
- Installed and configured OpenSearch 2.19.1 with HTTPS and admin authentication
- Installed OpenSearch Dashboards, accessible at `http://10.0.2.15:5601`
- Installed Apache NiFi 1.28.1 running in HTTP mode on port 8080

### 2. Syslog Ingestion Pipeline (NiFi)
- **ListenSyslog** processor configured on UDP port 5514
- Connected via **success** queue to **PutFile** processor writing to `/tmp/syslogs`
- Test messages sent using `logger` utility and confirmed received end-to-end
- Data indexed into OpenSearch `syslog-logs` index and verified via REST API

### 3. OpenSearch Dashboards
- Created `syslog-logs*` and `evtx-logs*` index patterns
- Built Metric visualization showing real-time event count
- Used Discover view with DQL queries to hunt through Windows Event Logs

---

## 🔴 Threat Analysis — 3 Attack Patterns Identified

### Attack 1 — Local Group Membership Enumeration
- **File:** `attack_1.evtx`
- **Event ID:** 4799
- **Actor:** `jbrown` (domain: 3B)
- **Tool:** `net1.exe` (net localgroup)
- **What happened:** Account jbrown queried the Administrators, Remote Desktop Users, and Users local groups 5 times within 3 minutes across multiple logon sessions — classic post-exploitation reconnaissance to identify privilege escalation paths.

### Attack 2 — Kerberos User Enumeration + Audit Log Wipe
- **File:** `attack_2.evtx`
- **Event IDs:** 4768, 4771, 1102
- **Attacker IP:** `172.16.66.1`
- **What happened:** Attacker first cleared the Windows Security audit log (1102) to destroy evidence, then sent automated Kerberos AS-REQ packets to enumerate valid domain accounts. 7 usernames returned Status `0x6` (not found); `Administrator` and a pre-planted `backdoor` account returned `0x18` (exists, wrong password). All 12 events occurred within 70ms — confirming automated tooling (Kerbrute/Rubeus).
- **Key finding:** The `backdoor` account has a valid domain SID, confirming a prior separate compromise.

### Attack 3 — MSSQL Brute Force Login Attack
- **File:** `attack_3.evtx`
- **Event ID:** 18456
- **Attacker IP:** `10.0.2.17`
- **Target:** `MSEDGEWIN10` — Microsoft SQL Server (`sa` account)
- **What happened:** 10 failed SQL Server login attempts from a single host within 110ms, targeting the `sa` (system administrator) account, `root`, and 8 internal certificate accounts using an automated credential wordlist tool (Hydra/Metasploit mssql_login). A successful `sa` login would have granted full database control and potential OS command execution via `xp_cmdshell`.

---

## 🛡️ Key Takeaways

- NiFi provides a flexible, low-code way to build real-time security data pipelines
- OpenSearch + Dashboards gives powerful log search and visualization without licensing cost
- Windows Event IDs 4799, 4768, 4771, 1102, and 18456 are high-value indicators for detecting reconnaissance, credential attacks, and anti-forensic behaviour
- A single compromised account (jbrown) performing group enumeration is often an early indicator of a larger intrusion chain

---

## 📄 Full Report

The complete technical report (with screenshots, configurations, and detailed analysis) is available in the [`report/`](./report/) folder.

---

## 🚀 Future Plans

- [ ] Add Alerting rules in OpenSearch for automatic detection of brute force patterns
- [ ] Expand NiFi pipeline to ingest Windows Event Logs in real time via Winlogbeat
- [ ] Add more attack scenarios (Pass-the-Hash, DCSync, Kerberoasting)
- [ ] Build a proper SIEM-style dashboard with multiple panels
- [ ] Integrate threat intelligence feeds via NiFi HTTP processors

---

*Built as a personal SOC home lab project — continuously evolving.*
