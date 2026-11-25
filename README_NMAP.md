# Nmap Cheatsheet & Lab Reference

> **Lab Safety Notice**
>
> Only run `nmap` against `127.0.0.1`, the isolated lab VMs, or dedicated Docker networks with explicit authorization from the supervisor or university network administrator. Running scans across live campus subnets can trigger institutional firewalls and IDS/IPS systems.

## Overview & NetFlow SOC Context

**Nmap** (Network Mapper) operates at **Layers 3 & 4** (Network and Transport layers) to discover hosts, map open TCP/UDP ports, fingerprint operating systems, and identify running services.

Because NetFlow records connection metadata (`SrcAddr`, `DstAddr`, `SrcPort`, `DstPort`, `Proto`, `Bytes`, `Packets`), `nmap` is ideal for testing network-layer anomaly detection on the ClickHouse + Grafana SOC dashboard:

* **Vertical Port Scans:** A single `SrcAddr` contacting hundreds or thousands of distinct `DstPort` values on one `DstAddr` within seconds.
* **Horizontal Host Sweeps:** A single `SrcAddr` probing the same port (or ICMP) across many distinct `DstAddr` hosts in a subnet.
* **High Flow-Rate Spikes:** Rapid, low-byte packets (`Packets = 1` or `2` per flow) that stress both collector ingestion (`nfcapd` / `GoFlow2`) and alert queries.

## Installation (Debian / Ubuntu)

```bash
sudo apt update && sudo apt install -y nmap
```

## Core Scan Types

| Flag | Description | Notes |
| :--- | :--- | :--- |
| `-sS` | TCP SYN (stealth / half-open) scan | Fast default; requires `sudo` |
| `-sT` | TCP Connect (full 3-way handshake) | Works without root; noisier in logs |
| `-sU` | UDP port scan | Slower; combine with `--top-ports` |
| `-sn` | Host discovery only (ping sweep) | Skips port scanning |
| `-Pn` | Skip host discovery | Treats target as online (bypasses ICMP blocks) |

## Port Selection & Timing

| Flag | Description | Example |
| :--- | :--- | :--- |
| `-p <ports>` | Specific ports or ranges | `-p 22,80,443` or `-p 1-1000` |
| `-p-` | All 65,535 TCP ports | `sudo nmap -sS -p- 127.0.0.1` |
| `-F` | Fast mode (top 100 common ports) | `nmap -F 127.0.0.1` |
| `--top-ports <n>` | Scan the *n* most common ports | `sudo nmap -sU --top-ports 50 127.0.0.1` |
| `-T0` to `-T5` | Timing template (Paranoid to Insane) | `-T4` is the standard fast lab setting |
| `--min-rate <n>` | Send at least *n* packets per second | `--min-rate 1000` (great for flow spikes) |
| `--max-retries <n>` | Cap probe retransmissions | `--max-retries 1` (speeds up filtered scans) |

## Enumeration, NSE Scripts & Output

| Flag | Description |
| :--- | :--- |
| `-sV` | Probe open ports to detect service and version info |
| `-O` | OS fingerprinting (requires `sudo`) |
| `-sC` | Run default safe Nmap Scripting Engine (NSE) scripts |
| `-A` | Aggressive bundle (`-O -sV -sC --traceroute`) |
| `--script <name>` | Run a specific NSE script or category (e.g., `vuln`, `http-enum`, `banner`) |
| `-v` / `-vv` | Increase verbosity during execution |
| `-oN <file>` | Save output in Normal human-readable format |
| `-oX <file>` | Save output in XML format |
| `-oG <file>` | Save output in Grepable format |
| `-oA <prefix>` | Save output in all three formats at once (`<prefix>.nmap`, `.xml`, `.gnmap`) |

## Ready-to-Run Lab Commands

```bash
# 1. Fast TCP SYN scan of top 1,000 ports (classic vertical port-scan signature in NetFlow)
sudo nmap -sS -T4 127.0.0.1

# 2. Full 65,535-port sweep at high packet rate (heavy flow spike for SOC alert testing)
sudo nmap -sS -p- --min-rate 2000 --max-retries 1 127.0.0.1

# 3. Service & default script enumeration on the SOC stack ports (Grafana, Bridge, ClickHouse)
nmap -sT -sV -sC -p 3000,8080,8123 127.0.0.1

# 4. Subnet host discovery (horizontal sweep across an isolated Docker network)
nmap -sn 172.18.0.0/24

# 5. UDP scan of top 50 ports (generates UDP flow records)
sudo nmap -sU --top-ports 50 -T4 127.0.0.1

# 6. Full baseline audit saved in all output formats
sudo nmap -sS -sV -sC -T4 -p 1-10000 -oA nmap_lab_baseline 127.0.0.1
```

## What to Look for in ClickHouse / Grafana

When running `nmap` while `softflowd` or `NFStream` exports flows to `GoFlow2` + `ClickHouse`:

* **High `uniq(DstPort)` per `SrcAddr`:** Strongest indicator of a vertical port scan.
* **High flow count with low `Bytes / Packets` ratio:** Typical of SYN scans (`-sS`) that never complete the handshake.
* **High `uniq(DstAddr)` per `SrcAddr`:** Strongest indicator of a horizontal host sweep (`-sn`).