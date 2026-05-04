---
title: 'Honeynet Suite: Multi-Service Honeypot and Attack Analysis Platform'
date: 2026-04-25
summary: 'A Python-based multi-service honeypot (SSH, HTTP, MySQL) deployed on DigitalOcean with structured JSON logging and a live attack-trend dashboard. Built as a defensive security portfolio project.'
links:
  - type: code
    url: https://github.com/Byrdsong-Portfolio/honeynet
tags:
  - Cybersecurity
  - Python
  - Network Security
  - Threat Intelligence
  - DigitalOcean
  - Honeypot
---

This project is a self-hosted honeynet running three emulated services on a DigitalOcean droplet. Every inbound connection attempt is captured, structured, and logged. The goal is twofold: build a realistic defensive security portfolio piece, and generate real attacker behavior data that feeds an analysis dashboard tracking attack trends over time.

The DigitalOcean deployment is currently being configured. This page will be updated with live dashboard screenshots and captured dataset summaries as the system accumulates data.

---

## Architecture

```
Internet
    |
    v
[ DigitalOcean Droplet: Ubuntu 22.04 ]
    |
    +--[ SSH Honeypot        :2222 ]  --> logs/ssh_events.jsonl
    |      Fake banner: OpenSSH 7.4
    |      Logs: IP, username, password, client version, timestamp
    |      Engine: Paramiko-based listener
    |
    +--[ HTTP Honeypot       :8080 ]  --> logs/http_events.jsonl
    |      Fake header: Apache/2.4.41
    |      Serves: fake login page, /admin, /phpmyadmin, /wp-login.php
    |      Logs: method, path, user-agent, POST body, IP
    |
    +--[ MySQL Honeypot      :3306 ]  --> logs/mysql_events.jsonl
    |      Emulates: MySQL 5.7 handshake sequence
    |      Logs: auth attempts, client version, capability flags
    |      Returns: ERR_ACCESS_DENIED to all connections
    |
    +--[ Dashboard           :5000 ]  --> Flask + Chart.js
           Attack volume by service (hourly / daily / weekly)
           Top attacking IPs with geo and ASN resolution
           Credential frequency table (SSH)
           HTTP path heatmap
           Live event feed, auto-refreshing
```

---

## Services

| Feature | SSH | HTTP | MySQL |
|---|---|---|---|
| Connection logging | Yes | Yes | Yes |
| Credential capture | Username + password | POST form fields | Handshake auth |
| Fake service banner | OpenSSH 7.4 | Apache 2.4.41 | MySQL 5.7.38 |
| Structured JSON output | Yes | Yes | Yes |
| Geo-IP resolution | Yes | Yes | Yes |
| Rate limiting | Yes | Yes | Yes |
| Dashboard integration | Yes | Yes | Yes |

---

## Log Format

Each service writes one JSON object per line. No raw packets are stored, only the parsed fields relevant to attacker behavior analysis.

**SSH event example:**
```json
{
  "timestamp": "2026-05-01T14:22:03Z",
  "service": "ssh",
  "source_ip": "198.51.100.23",
  "geo": { "country": "CN", "city": "Guangzhou", "asn": "AS4134" },
  "username": "root",
  "password": "admin123",
  "client_version": "SSH-2.0-libssh_0.9.6"
}
```

**HTTP event example:**
```json
{
  "timestamp": "2026-05-01T14:22:45Z",
  "service": "http",
  "source_ip": "203.0.113.77",
  "method": "POST",
  "path": "/wp-login.php",
  "user_agent": "Mozilla/5.0 (compatible; MJ12bot/v1.4.8)",
  "post_body": { "log": "admin", "pwd": "password1" },
  "geo": { "country": "RU", "city": "Moscow", "asn": "AS8359" }
}
```

**MySQL event example:**
```json
{
  "timestamp": "2026-05-01T14:23:10Z",
  "service": "mysql",
  "source_ip": "185.220.101.45",
  "username": "root",
  "auth_plugin": "mysql_native_password",
  "geo": { "country": "DE", "city": "Frankfurt", "asn": "AS396507" }
}
```

---

## Project File Structure

```
honeynet/
├── ssh_honeypot.py          # Paramiko SSH listener
├── http_honeypot.py         # Flask HTTP trap
├── mysql_honeypot.py        # Raw socket MySQL emulator
├── dashboard/
│   ├── app.py               # Flask dashboard server
│   ├── templates/index.html # Chart.js dashboard UI
│   └── static/charts.js     # Chart configuration
├── utils/
│   ├── logger.py            # Shared structured JSON logger
│   ├── geoip.py             # IP geolocation helper
│   └── alerting.py          # Webhook/email alert hooks
├── fake_data/               # Runtime bait data (gitignored)
├── logs/                    # Captured events (gitignored)
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Quick Start

```bash
git clone https://github.com/Byrdsong-Portfolio/honeynet.git
cd honeynet
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Start individual services
nohup python3 ssh_honeypot.py &
nohup python3 http_honeypot.py &
nohup python3 mysql_honeypot.py &

# Start dashboard
python3 dashboard/app.py
```

**DigitalOcean firewall rules required before launch:**

| Port | Protocol | Open To | Purpose |
|---|---|---|---|
| 2222 | TCP | Any | SSH honeypot |
| 3306 | TCP | Any | MySQL honeypot |
| 8080 | TCP | Any | HTTP honeypot |
| 5000 | TCP | Your IP only | Dashboard (keep private) |
| 22 | TCP | Your IP only | Real admin SSH |

---

## Overtime Dashboard Analysis

Once the DigitalOcean deployment is live, the dashboard will track the following metrics continuously and surface trends as data accumulates.

**Volume metrics (per service, per time window):**
- Total connection attempts per hour, day, and week
- New unique IPs per day vs. repeat offenders
- Service-level comparison (which service attracts the most attempts)

**Credential analysis (SSH and HTTP):**
- Top 25 usernames attempted, ranked by frequency
- Top 25 passwords attempted, ranked by frequency
- Most common username/password pairs (credential combos)
- New credentials not seen in prior 7-day window (emerging wordlists)
- Credential attempt velocity spikes (burst detection)

**Geographic and network analysis:**
- Top 10 source countries by connection volume
- Top autonomous systems (ASNs) responsible for attack traffic
- Tor exit node vs. residential vs. datacenter IP classification
- Geolocation shift alerts (unusual new origin countries)

**HTTP-specific analysis:**
- Most probed paths ranked by hit count (path heatmap)
- User-agent breakdown (automated scanners vs. browser strings)
- Most common vulnerability scan signatures (Nuclei, Shodan, Masscan patterns)
- POST body keyword frequency (what credentials/payloads are being submitted)

**Trend indicators I will be watching:**
- Week-over-week change in total attempt volume by service
- Correlation between global CVE disclosures and HTTP path spikes
- Credential wordlist evolution (are attackers using new lists vs. rockyou-derived?)
- SSH client version distribution over time (what tools are attackers using?)

**Dashboard update schedule:** Screenshots and dataset summaries will be posted here monthly as the deployment accumulates data.

---

## Deployment Status

| Component | Status |
|---|---|
| GitHub repository | Live |
| SSH honeypot (code) | Complete |
| HTTP honeypot (code) | Complete |
| MySQL honeypot (code) | Complete |
| Dashboard (code) | Complete |
| DigitalOcean droplet | Pending provisioning |
| Live data collection | Pending deployment |
| First analysis post | Pending first 30 days of data |

---

## Legal and Ethics

This system is deployed on infrastructure I own and operate. All captured data reflects unsolicited inbound connection attempts initiated by external actors against exposed service ports.

- No active scanning, exploitation, or unauthorized access is performed by this system
- Captured credentials are stored locally and are not re-used, shared, or published in identifiable form
- IP addresses in logs are retained for research and analysis purposes only
- This project operates as a passive listener and complies with applicable computer fraud and abuse law
- Do not deploy this system on infrastructure you do not own or have explicit authorization to run research tools on

<!--more-->
