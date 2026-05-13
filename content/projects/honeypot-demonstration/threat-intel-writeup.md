# Threat Intelligence Report: HoneyNet Deployment
**Classification:** Public Portfolio · Sanitized  
**Author:** Isaiah Byrdsong  
**Platform:** DigitalOcean VPS · Ubuntu 22.04  
**Observation Period:** 30-day continuous deployment  

---

## Executive Summary

A multi-service honeynet was deployed on a public-facing cloud VPS to capture and analyze real-world opportunistic attack traffic. Three emulated services — SSH (port 2223), HTTP (port 80), and MySQL (port 3306) — were exposed to the open internet with no advertising or indexing. All connection attempts represent unsolicited, automated attacker activity.

**Key findings:**

- Automated SSH brute-force began within **11 minutes** of VPS provisioning
- Over **85% of credential attempts** used a concentrated dictionary of fewer than 200 common root/admin passwords
- HTTP scanners consistently probed `/wp-login.php`, `/.env`, and `/phpmyadmin` within the first 60 seconds of each new scanner session
- MySQL was targeted by clients advertising themselves as legitimate MySQL Workbench and connector builds, suggesting tooling reuse from legitimate software
- The top 5 source countries accounted for **72% of all connection attempts**
- Honey-token files (fake `credentials.txt`, `dump.sql`) were downloaded within 4 hours of first being served — subsequent SSH sessions used usernames matching the embedded fake credentials

---

## Methodology

### Infrastructure

A DigitalOcean Droplet (Ubuntu 22.04, 1 vCPU / 1 GB RAM) was provisioned with no firewall rules blocking inbound traffic on target ports. The real SSH management port (22) was restricted to a single trusted IP. No DNS records or public references were created for the VPS IP address.

### Services Deployed

| Service | Port | Technology | Purpose |
|---|---|---|---|
| SSH Honeypot | 2223 | Python / Paramiko | Capture brute-force credential attempts and interactive commands |
| HTTP Honeypot | 80 | Python / Flask | Capture scanner probes, web login attempts, file enumeration |
| MySQL Honeypot | 3306 | Python raw sockets | Capture DB auth attempts and client version fingerprints |
| Analytics Dashboard | 5000 | Flask / Chart.js / SocketIO | Real-time event visualization (restricted access) |

### Data Collection

Every inbound connection was logged to:
- A per-service **JSONL flat file** (`logs/{service}_events.jsonl`) for line-by-line streaming analysis
- A shared **SQLite database** (`logs/honeynet.db`) for aggregate queries and dashboard rendering

IP geolocation used the [ip-api.com](http://ip-api.com) free tier (45 req/min) with an in-process LRU cache to avoid redundant lookups.

Discord webhooks delivered real-time alerts for every hit via a synchronous HTTP POST — no cooldown, no queuing, ensuring zero missed events.

---

## Key Findings

### 1 — SSH Brute Force (T1110.001)

**Observation window:** Continuous  
**Volume:** ~2,400 unique credential pairs captured over 30 days

| Username | Attempt Count |
|---|---|
| root | 1,842 |
| admin | 391 |
| ubuntu | 287 |
| deploy | 104 |
| pi | 76 |

| Password | Attempt Count |
|---|---|
| 123456 | 312 |
| admin | 287 |
| password | 241 |
| root | 198 |
| 1234 | 176 |

**Top source countries:** CN (34%), RU (18%), US (11%), NL (5%), DE (4%)

The overwhelming majority of attackers used `libssh` or automated tools (Masscan, custom scanners) identifiable by their SSH client version string. A smaller subset opened interactive shells and issued reconnaissance commands: `id`, `uname -a`, `cat /etc/passwd`, `history`.

### 2 — HTTP Scanner Activity (T1046 / T1190 / T1083)

**Volume:** ~8,700 HTTP requests logged

**Top probed paths:**

| Path | Hits | Category |
|---|---|---|
| /wp-login.php | 1,241 | WordPress credential attack |
| /.env | 892 | Secret/config exfiltration |
| /admin | 784 | Generic admin panel probe |
| /phpmyadmin | 661 | Database admin panel probe |
| /.git/config | 498 | Source code exfiltration |
| /wp-config.php | 417 | WordPress config exfiltration |
| /backup/credentials.txt | 203 | Honey-token download |
| /admin/dump.sql | 187 | Honey-token download |

Notable: scanners that downloaded the honey-token `credentials.txt` file subsequently appeared in SSH logs attempting the embedded fake username (`deploy`) and password within the same 24-hour window — demonstrating credential-reuse behavior across attack phases.

### 3 — MySQL Auth Attempts (T1078)

**Volume:** ~340 unique connection attempts

Most MySQL clients identified themselves as legitimate software (MySQL Workbench, MySQL Connector/Python, MySQL Connector/J) — indicating attackers reusing standard tooling rather than custom exploits. The vast majority attempted authentication as `root` with no password (empty auth response), consistent with scanning for misconfigured default installs.

| Username | Count |
|---|---|
| root | 298 |
| admin | 24 |
| mysql | 11 |
| (blank) | 7 |

### 4 — Attack Timeline

- **T+0:11** — First SSH brute-force attempt (automated scanner, CN origin)
- **T+0:43** — First HTTP probe (`/wp-login.php`)
- **T+1:02** — First MySQL connection attempt
- **T+4:17** — First honey-token file download (`credentials.txt`)
- **T+4:51** — SSH attempt using honey-token username (`deploy`)
- **T+6:00** — First interactive SSH shell session (attacker executed `id`, `uname -a`, `ls`)
- **T+24:00** — ~800 credential attempts logged across all services

---

## MITRE ATT&CK Technique Mapping

| Technique ID | Name | Tactic | Observed In |
|---|---|---|---|
| T1595.001 | Active Scanning: Scanning IP Blocks | Reconnaissance | All services — sequential probes from single IPs |
| T1046 | Network Service Discovery | Discovery | Full port sweeps preceding targeted probes |
| T1110 | Brute Force | Credential Access | SSH, HTTP |
| T1110.001 | Password Guessing | Credential Access | SSH — dictionary attack on `root` account |
| T1110.003 | Password Spraying | Credential Access | HTTP — common creds sprayed across `/admin`, `/wp-login.php` |
| T1190 | Exploit Public-Facing Application | Initial Access | HTTP — probing known CVE paths (`/phpmyadmin`, `/wp-config.php`) |
| T1083 | File and Directory Discovery | Discovery | HTTP — systematic enumeration of config/backup paths |
| T1078 | Valid Accounts | Defense Evasion | SSH — reuse of credentials from honey-token file download |
| T1059.004 | Command and Scripting Interpreter: Unix Shell | Execution | SSH — interactive shell commands (`id`, `uname`, `cat /etc/passwd`) |

---

## Defensive Recommendations

Based on observed attacker behavior, the following controls would prevent or significantly impede the techniques observed:

1. **Disable password authentication for SSH.** Replace with ed25519 key-based auth. This eliminates the entire T1110.001 attack surface.

2. **Rename or restrict SSH port.** Moving SSH from port 22 to a high-numbered unpublished port reduces automated scanner hits by ~95% (measured across comparable deployments).

3. **Enforce fail2ban or equivalent.** A 5-attempt lockout with 1-hour ban eliminates slow brute-force scans. Most observed scanners used 1–3 attempts per session to avoid rate limits.

4. **Remove public HTTP admin surfaces.** `/wp-admin`, `/phpmyadmin`, and similar paths should be IP-restricted or removed if not needed. Apache/Nginx access controls block the entire T1190 class observed.

5. **Never expose MySQL port 3306 to the internet.** Bind to `127.0.0.1` or use a VPN/SSH tunnel for remote DB access. All MySQL hits observed were opportunistic scans, not targeted attacks.

6. **Rotate secrets immediately if `.env` or config files are ever web-accessible.** The 4-hour window from honey-token exposure to credential reuse demonstrates how quickly scraped credentials enter attacker tooling.

7. **Implement egress filtering.** Many observed SSH sessions attempted to `wget`/`curl` secondary payloads. Blocking outbound connections from servers to unknown IPs limits post-exploitation reach.

---

## Conclusion

This deployment confirms that public-internet exposure of common service ports triggers automated attack activity within minutes, not hours. The attack patterns observed are overwhelmingly automated and opportunistic — not targeted — which means standard hardening controls (key-only SSH, restricted admin surfaces, no exposed DB ports) would have prevented nearly every observed intrusion attempt.

The honey-token credential reuse finding is the most operationally interesting result: it demonstrates that attackers actively collect and pipeline credentials from HTTP-accessible config files into credential brute-force campaigns, often within hours of initial download.

---

## Appendix: Tool Stack

| Component | Technology | Notes |
|---|---|---|
| SSH Honeypot | Python 3.10 / Paramiko | Full ServerInterface implementation, fake interactive shell |
| HTTP Honeypot | Python 3.10 / Flask | Apache/PHP fake headers, honey-token file generation |
| MySQL Honeypot | Python 3.10 / raw sockets | MySQL 5.7 HandshakeV10, full capability flag parsing |
| Logging | Python stdlib / SQLite | Shared JSONL + SQLite event store |
| Geo-IP | ip-api.com REST / MaxMind GeoLite2 | In-process LRU cache |
| Alerting | Discord Webhooks | Synchronous HTTP POST per event |
| Dashboard | Flask / Flask-SocketIO / Chart.js | Real-time SocketIO event push, REST API |
| Infrastructure | DigitalOcean Droplet | Ubuntu 22.04, 1 vCPU / 1 GB RAM |
| Deployment | bash start.sh / systemd | 30s watchdog loop, nohup process management |

---

*Isaiah Byrdsong · [LinkedIn](https://www.linkedin.com/in/ibyrdsong/) · Cybersecurity & Intelligence Professional*
