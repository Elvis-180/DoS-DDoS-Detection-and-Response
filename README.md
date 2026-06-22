# DoS/DDoS Detection and Response

![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-T1498%20%7C%20T1499%20%7C%20T1595-red)
![Splunk](https://img.shields.io/badge/SIEM-Splunk%2010.2.2-orange)
![Snort](https://img.shields.io/badge/IDS-Snort%203.x-blue)
![pfSense](https://img.shields.io/badge/Firewall-pfSense-green)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

> A fully functional home SOC lab simulating, detecting, and responding to DoS/DDoS attacks using a layered defense architecture — Snort IDS + Splunk SIEM + pfSense Firewall.

---
##  Project Report

>  [Download Full SOC Lab Report (PDF)](DoS_DDoS_SOC_Lab_ILLUSTRATED.pdf)
---

##  Table of Contents

- [Project Overview](#project-overview)
- [Lab Architecture](#lab-architecture)
- [Detection Stack](#detection-stack)
- [Attack Simulations](#attack-simulations)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [Snort Detection Rules](#snort-detection-rules)
- [Splunk Dashboard](#splunk-dashboard)
- [Key Results](#key-results)
- [Incident Response](#incident-response)
- [Tools Used](#tools-used)

---

##  Project Overview

This project demonstrates a complete **DoS/DDoS detection and response pipeline** built in a home lab environment. The goal was to simulate real-world network attacks, detect them using custom IDS rules, and visualize the detections in a professional SOC dashboard.

**What was accomplished:**
- Designed and deployed a 4-zone segmented network using pfSense
- Performed active reconnaissance using Nmap (MITRE T1595)
- Simulated SYN Flood, HTTP Flood, and UDP Flood attacks using hping3
- Detected all attacks using custom Snort rules mapped to MITRE ATT&CK
- Built a real-time Splunk SIEM dashboard with severity classification
- Configured automated alerting triggering 18+ times during simulation
- Captured packet-level evidence using Wireshark

---

##  Lab Architecture

### Network Topology

```
                    INTERNET (WAN)
                          │
                   ┌──────────────┐
                   │   pfSense    │  ← Firewall / Router / DHCP
                   │  Firewall    │
                   └──────┬───────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
  LAN (192.168.1.x)   OPT1            Workstation
         │           (192.168.2.x)    (192.168.3.x)
   ┌─────┴──────┐         │                │
   │            │     Kali Linux       Windows 10
Windows      Ubuntu    192.168.2.5     192.168.3.2
Server       Server   [ATTACKER ]   [Domain PC]
192.168.1.1  192.168.1.9
[DC/TARGET]  [Snort+Splunk]
```

### Device Inventory

| Device | OS | IP Address | Role |
|--------|-----|-----------|------|
| pfSense | FreeBSD | 192.168.1.2 / 192.168.2.1 / 192.168.3.1 | Firewall / Router / DHCP |
| Windows Server 2025 | Windows Server | 192.168.1.1 | Domain Controller / IIS Target |
| Ubuntu Server | Ubuntu Linux | 192.168.1.9 | Snort IDS + Splunk SIEM |
| Windows 10 | Windows 10 | 192.168.3.2 | Domain Workstation |
| Kali Linux | Debian Linux | 192.168.2.5 | Attacker Machine  |

### Network Zones

| Interface | Subnet | Zone | Trust Level |
|-----------|--------|------|-------------|
| WAN | Dynamic | Internet |  Untrusted |
| LAN | 192.168.1.0/24 | Server Zone | Trusted |
| OPT1 | 192.168.2.0/24 | Attacker Zone |  Untrusted |
| Workstation | 192.168.3.0/24 | User Zone |  Semi-Trusted |

---

##  Detection Stack

```
LAYER 1 — Perimeter Defense
pfSense firewall enforces strict zone-based rules.
Only legitimate traffic passes. Everything else is
blocked, dropped, and logged to Splunk.

LAYER 2 — Intrusion Detection  
Snort IDS monitors all LAN traffic using 8 custom
rules. Any DoS/DDoS pattern triggers an alert which
is written to snort.alert.fast and forwarded to Splunk.

LAYER 3 — SIEM Visibility & Response
Splunk ingests all logs from Snort and pfSense,
displays real-time severity levels, and fires
automated alerts when thresholds are breached.
```

### Log Flow

```
Snort (Ubuntu 192.168.1.9)
  └─► /var/log/snort/snort.alert.fast
        └─► Splunk UF (inputs.conf)
              └─► index=security | sourcetype=fast_alert
                    └─► SOC Dashboard + Alerts

pfSense ──► Syslog UDP:5514
              └─► index=pfsense | sourcetype=syslog
                    └─► Perimeter Firewall Panel

Windows Server/Win10 ──► Splunk UF
                           └─► WinEventLog:Security
                                 └─► Brute Force + AD Panels
```

---

##  Attack Simulations

### Phase 1 — Reconnaissance (T1595)

```bash
# Aggressive network scan from Kali Linux
nmap -A 192.168.1.0/24
```

**Findings:**
- Target identified: 192.168.1.1 (Windows Server 2025)
- Port 80 open: Microsoft IIS httpd 10.0
- Domain: cis.net (Default-First-Site-Name)
- 256 IPs scanned in 48.9 seconds
- Snort detected scan: 15 ICMP PING NMAP alerts in Splunk

### Phase 2 — SYN Flood (T1499.001)

```bash
sudo hping3 -S --flood -p 80 192.168.1.1
```

- **138,186 packets transmitted**
- 100% packet loss on target
- Snort SID 1000002 triggered continuously

### Phase 3 — HTTP Flood (T1499.002)

```bash
sudo hping3 -S -p 80 --flood --rand-source 192.168.1.1
```

- **144,995 detection events** in Splunk
- Sequential port usage pattern detected
- Snort SID 1000008 triggered

### Phase 4 — UDP Flood (T1498)

```bash
sudo hping3 --udp --flood -p 80 192.168.1.1
```

- **610,013 events in 15 minutes**
- Highest volumetric attack of the simulation
- Snort SID 1000004 triggered

---

##  MITRE ATT&CK Mapping

| Tactic | Technique | ID | Tool | Evidence |
|--------|-----------|-----|------|---------|
| Reconnaissance | Active Scanning | T1595 | Nmap | 15 ICMP alerts in Splunk |
| Reconnaissance | Scanning IP Blocks | T1595.001 | Nmap -A | 256 IPs in 48.9s |
| Impact | Network DoS | T1498 | hping3 | 610,013 UDP flood events |
| Impact | Direct Network Flood | T1498.001 | hping3 --flood | 138,186 SYN packets |
| Impact | Reflection Amplification | T1498.002 | Snort rule | Rule SID 1000007 deployed |
| Impact | Endpoint DoS | T1499 | hping3 | 100% packet loss confirmed |
| Impact | SYN Flood | T1499.001 | hping3 -S | Snort SID 1000002 triggered |
| Impact | Service Exhaustion | T1499.002 | hping3 HTTP | 144,995 HTTP flood events |

---

##  Snort Detection Rules

**File:** `/etc/snort/rules/local.rules`

```snort
# HTTP GET Detection - T1595
alert tcp any any -> any 80 (msg:"HTTP GET Request Detected"; \
  flow:to_server,established; content:"GET"; http_method; \
  classtype:attempted-recon; sid:1000001; rev:1;)

# SYN Flood Detection - T1499.001
alert tcp any any -> $HOME_NET any (flags:S; \
  msg:"Potential DoS/DDoS SYN Flood Detected"; \
  detection_filter:track by_dst, count 100, seconds 5; \
  sid:1000002; rev:1;)

# ICMP Flood Detection - T1498
alert icmp any any -> $HOME_NET any \
  (msg:"Potential ICMP Flood Detected"; \
  detection_filter:track by_dst, count 50, seconds 5; \
  sid:1000003; rev:1;)

# UDP Flood Detection - T1498
alert udp any any -> $HOME_NET any \
  (msg:"Potential UDP Flood Detected"; \
  detection_filter:track by_dst, count 50, seconds 5; \
  sid:1000004; rev:1;)

# Port Scan Detection - T1595.001
alert tcp any any -> $HOME_NET any (flags:S; \
  msg:"Potential Port Scan Detected"; \
  detection_filter:track by_src, count 20, seconds 5; \
  sid:1000005; rev:1;)

# SSH Brute Force - T1110
alert tcp any any -> $HOME_NET 22 \
  (msg:"Potential SSH Brute Force Detected"; \
  detection_filter:track by_src, count 5, seconds 60; \
  sid:1000006; rev:1;)

# DNS Amplification - T1498.002
alert udp any any -> $HOME_NET 53 \
  (msg:"Potential DNS Amplification Attack"; \
  detection_filter:track by_dst, count 100, seconds 5; \
  sid:1000007; rev:1;)

# HTTP Flood Detection - T1499.002
alert tcp any any -> $HOME_NET 80 \
  (msg:"Potential HTTP Flood Detected"; \
  detection_filter:track by_dst, count 100, seconds 10; \
  sid:1000008; rev:1;)
```

---

##  Splunk Dashboard

### Key Detection Queries

**DoS/DDoS Master Detection:**
```spl
index=security sourcetype=fast_alert
| rex field=_raw "\{(?<proto>\w+)\}\s+(?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}):(?<src_port>\d+)\s+->\s+(?<dest_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}):(?<dest_port>\d+)"
| bucket _time span=1m
| stats count as alert_count, dc(dest_port) as ports_targeted, values(proto) as protocols by _time, src_ip, dest_ip
| eval severity=case(
    alert_count > 50, " HIGH",
    alert_count > 20, " MEDIUM",
    alert_count > 5,  " LOW",
    true(), "INFO"
  )
| where severity!="INFO"
| sort -alert_count
```

**Top Attacker IPs:**
```spl
index=security sourcetype=fast_alert
| rex field=_raw "(?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}):\d+\s+->"
| stats count as attack_count by src_ip
| sort -attack_count
| head 10
```

### Dashboard Panels

| Panel | Description | MITRE |
|-------|-------------|-------|
| Total Events KPI | Count of all security events (24h) | — |
| DoS/DDoS Attacks KPI | Active attack sessions detected | T1498/T1499 |
| Brute Force KPI | Failed login attempts | T1110 |
| AD Changes KPI | Group modification events | T1098 |
| Real-Time Incident Timeline | Line chart of events over time | — |
| Top Attacker IPs | Bar chart of highest volume sources | T1595 |
| DoS/DDoS Attack Types | Pie chart with severity labels | T1498/T1499 |
| Brute Force Accounts | Targeted user accounts pie chart | T1110 |
| Perimeter Firewall | pfSense interface activity | T1562.004 |
| HIGH/MEDIUM Severity | Single value panels with color coding | T1498/T1499 |
| Live Threat Feed | Real-time last 20 Snort alerts | — |
| Active Directory | Group modification events table | T1098/T1069 |

---

##  Key Results

| Metric | Value |
|--------|-------|
| Total Security Events | 1,528,725 |
| Active DoS/DDoS Attacks | 4 (HIGH Severity) |
| Brute Force Attempts | 44 |
| SYN Flood Packets | 138,186 |
| HTTP Flood Events | 144,995 |
| UDP Flood Events (15 min) | 610,013 |
| Splunk Alerts Triggered | 18+ |
| Snort Rules Deployed | 8 |
| MITRE Techniques Covered | 8 |
| Wireshark Packets Captured | 800 |

---

##  Incident Response 

| Phase | Actions Taken |
|-------|--------------|
| **Preparation** | Snort rules deployed, Splunk alerts configured, pfSense policies set |
| **Identification** | Snort SID 1000002/1000004/1000008 fired, Splunk dashboard confirmed HIGH severity |
| **Containment** | Block 192.168.2.5 at pfSense OPT1, apply rate limiting on SYN packets |
| **Eradication** | Verify IIS service restored, tighten pfSense firewall rules |
| **Recovery** | Confirm normal traffic baseline in Splunk, resume domain operations |
| **Lessons Learned** | Tune Snort thresholds, pre-configure pfSense rate limiting |

---

##  Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| Oracle VirtualBox | Latest | Hypervisor for all VMs |
| pfSense | CE | Firewall, router, DHCP server |
| Snort IDS | 3.x | Network intrusion detection |
| Splunk Enterprise | 10.2.2 | SIEM, dashboards, alerting |
| Splunk Universal Forwarder | 9.x | Log shipping from Ubuntu |
| Kali Linux | Latest | Attacker machine |
| hping3 | Latest | DoS/DDoS attack simulation |
| Nmap | 7.9x | Network reconnaissance |
| Wireshark | Latest | Packet capture and analysis |
| Windows Server 2025 | 10.0.26100 | Domain controller / IIS target |
| Windows 10 | Latest | Domain workstation |
| Ubuntu Server | Latest | IDS + SIEM host |

---
## Screenshots
Dashboard
<img width="1920" height="1080" alt="Screenshot 2026-06-21 214017" src="https://github.com/user-attachments/assets/5076fd48-634f-4e2c-9228-bcbfaeb5215a" />
Wireshark I/O Graph
<img width="1920" height="1080" alt="Screenshot 2026-06-21 215202" src="https://github.com/user-attachments/assets/50156e24-dad8-4b77-b488-85cdb9042286" />
Dos/DDoS Alerts
<img width="1920" height="1080" alt="Screenshot 2026-06-21 214506" src="https://github.com/user-attachments/assets/ff3d3d3a-4375-4c96-b90c-c0593268beb9" />
Wireshark Detection
<img width="1920" height="1080" alt="Screenshot 2026-06-21 214711" src="https://github.com/user-attachments/assets/08d3822d-89cb-48b9-bb8d-3bc895d4ae0a" />

##  License

This project is for educational purposes only. All attack simulations were conducted in an isolated lab environment with no external systems targeted.

---

*Date: June 21, 2026*
