> **Lab Safety Notice**
>
> Only execute scans, traffic replays, and load tests against `127.0.0.1`, the isolated lab VMs, or dedicated Docker networks with explicit authorization from the thesis supervisor or the university network administrator. Running active scans or packet floods across live campus subnets can trigger institutional firewalls and IDS/IPS systems.

## 1. Installation & Isolated Environment Setup

Keep third-party tools outside the `netflow_soc_upgrade/` repository (for example, under `/opt/pentest_tools/`) to avoid nested `.git` directories and bloated commits.

```bash
# Create an external tools directory
sudo mkdir -p /opt/pentest_tools && sudo chown "$USER:$USER" /opt/pentest_tools
cd /opt/pentest_tools

# Clone Nikto directly from GitHub for the latest vulnerability database
git clone --depth 1 https://github.com/sullo/nikto.git
sudo ln -s /opt/pentest_tools/nikto/program/nikto.pl /usr/local/bin/nikto

# Install remaining networking, replay, load-testing, and monitoring tools via apt
sudo apt update && sudo apt install -y nmap wrk tcpreplay nfdump softflowd sysstat perl
```

Optional tools:
* **`hey`** (HTTP load generator with simple `POST` body support): `sudo apt install -y hey` or `go install github.com/rakyll/hey@latest`
* **`NFStream`** (Python `.pcap` / live interface flow extractor): `pip install nfstream` inside a virtual environment

### Docker Network Isolation

Attach the test containers to an internal Docker bridge so simulated attack traffic cannot route onto the university LAN or the internet:

```yaml
networks:
  isolated_lab:
    driver: bridge
    internal: true
```

---

## 2. Flow Capture & Traffic Replay

### Legacy Capture & Live Export (`nfcapd`, `nfdump`, `softflowd`)

```bash
# Start the legacy collector (UDP 9995, 5-minute file rotation)
sudo nfcapd -D -l /var/log/netflow -p 9995 -t 300

# Query legacy Top 10 Talkers as CSV (wrapped by flow_bridge.py)
nfdump -R /var/log/netflow -s srcip/bytes -n 10 -o csv

# Export live traffic from loopback or a lab NIC as NetFlow v9
sudo softflowd -i lo -v 9 -n 127.0.0.1:2055   # Modern prototype (GoFlow2)
sudo softflowd -i lo -v 9 -n 127.0.0.1:9995   # Legacy prototype (nfcapd)
```

### NetFlow File Replay (`nfreplay`)

Replays captured `nfcapd.*` binary files directly to either collector (`9995` or `2055`) for reproducible ingestion benchmarks.

| Flag | Purpose |
| :--- | :--- |
| `-r <file>` | Input `nfcapd` capture file |
| `-H <ip>` / `-p <port>` | Target host and UDP port (default `9995`) |
| `-v <5\|9>` | Output NetFlow version (`5` or `9`) |
| `-z <N>` | Preserve real time distribution multiplied by `N` (e.g., `-z 20` replays 5 min in 15 sec) |
| `-d <usec>` | Fixed delay in microseconds between records |
| `-c <num>` | Stop after sending `num` flow records |
| `-b <bytes>` | Set UDP send buffer size for high-rate transfers |

```bash
# Replay a 5-minute capture at 10x speed to GoFlow2
nfreplay -r /var/log/netflow/nfcapd.202611261200 -H 127.0.0.1 -p 2055 -v 9 -z 10

# Send exactly 100,000 flows with a 10µs delay between records
nfreplay -r /var/log/netflow/nfcapd.202611261200 -H 127.0.0.1 -p 2055 -v 9 -d 10 -c 100000
```

### Packet-Level Replay (`tcpreplay`) & Flow Extraction (`NFStream`)

Use when feeding `.pcap` captures through `softflowd` or `NFStream` to test the full exporter-to-collector pipeline.

| Flag | Purpose |
| :--- | :--- |
| `-i <iface>` | Network interface for packet injection (`lo` or isolated lab interface) |
| `-x <N>` / `--multiplier=<N>` | Replay at `N` times original capture speed |
| `--pps=<N>` / `--pps-multi=<M>` | Fixed packets/sec (`N`), sending bursts of `M` packets for smooth high-rate pacing |
| `-M <N>` / `--mbps=<N>` | Replay at fixed bandwidth (`N` Mbps) |
| `-t` / `--topspeed` | Send packets as fast as the NIC/CPU allows |
| `-l <N>` / `--loop=<N>` | Loop the capture `N` times (`0` = infinite) |
| `--unique-ip` | Rewrite IPs on each loop iteration to generate distinct flows (pair with `-l`) |
| `-K` / `--preload-pcap` | Preload the capture into RAM before starting for consistent timing |
| `--stats=<sec>` | Print live throughput stats every `sec` seconds |

```bash
# Replay a pcap at 10x speed on loopback, printing stats every 5s
sudo tcpreplay -i lo --multiplier=10 --stats=5 capture.pcap

# High-rate flow stress test: 10,000 pps, 5 loops, unique IPs per loop, preloaded in RAM
sudo tcpreplay -i lo --pps=10000 --pps-multi=10 -l 5 --unique-ip -K --stats=5 capture.pcap

# Offline .pcap flow counting with NFStream (Python)
python3 -c 'from nfstream import NFStreamer; print(sum(1 for _ in NFStreamer(source="capture.pcap")))'
```

---

## 3. Attack & Anomaly Simulation (Purple Team)

Keep all targets bound to `127.0.0.1` or an isolated Docker bridge (`internal: true`).

### Network & Transport Layer (`nmap`)

Operates at Layers 3 & 4 and generates clear anomalies in NetFlow metadata: vertical port scans (high `uniq(DstPort)` per `SrcAddr`), horizontal sweeps (high `uniq(DstAddr)`), and low-byte SYN bursts.

| Flag | Description |
| :--- | :--- |
| `-sS` / `-sT` / `-sU` | TCP SYN scan (`sudo`), TCP Connect scan, or UDP scan |
| `-sn` / `-Pn` | Host discovery only (ping sweep) / skip host discovery |
| `-p <ports>` / `-p-` / `--top-ports <n>` | Specific ports, all 65,535 TCP ports, or top *n* common ports |
| `-T0` to `-T5` / `--min-rate <n>` | Timing template (`-T4` standard fast lab) / minimum packets per second |
| `-sV` / `-sC` / `-O` / `-A` | Version detection, default NSE scripts, OS detection, aggressive bundle |
| `-oA <prefix>` | Save results in Normal, XML, and Grepable formats simultaneously |

```bash
# Vertical TCP SYN port scan (top 1,000 ports) -> high distinct DstPort count per SrcAddr
sudo nmap -sS -T4 127.0.0.1

# Aggressive full port sweep -> heavy flow/packet burst for alert latency testing
sudo nmap -sS -p- --min-rate 2000 --max-retries 1 127.0.0.1

# Service & default script enumeration on the SOC stack ports (Grafana, Bridge, ClickHouse)
nmap -sT -sV -sC -p 3000,8080,8123 127.0.0.1

# Horizontal subnet discovery -> high distinct DstAddr count across the lab network
nmap -sn 172.18.0.0/24

# UDP top-50 scan -> verifies non-TCP protocol visibility in the SOC dashboard
sudo nmap -sU --top-ports 50 -T4 127.0.0.1
```

### Application Layer (`nikto`)

Operates at Layer 7 to audit HTTP/HTTPS endpoints for misconfigurations, headers, and exposed paths, while generating a sustained **single-port TCP flow spike** in NetFlow.

| Flag | Description |
| :--- | :--- |
| `-h <target>` / `-p <ports>` | Target IP/URL and port(s) |
| `-ssl` / `-nossl` | Force HTTPS mode or disable SSL checks |
| `-id <user:pass>` | HTTP Basic Auth credentials |
| `-Tuning <codes>` | Filter check categories (`1` files, `2` misconfig, `3` info, `b` software; `x6` excludes DoS) |
| `-maxtime <time>` / `-Pause <sec>` | Hard stop duration (e.g., `60s`) / delay between requests |
| `-no404` | Skip 404 heuristic checks (faster against minimal APIs) |
| `-o <file> -Format <fmt>` | Export findings (`txt`, `csv`, `htm`, `xml`, `json`) |

```bash
# Time-boxed HTTP burst against the legacy Python bridge (port 8080)
nikto -h http://127.0.0.1 -p 8080 -nossl -maxtime 60s

# Safe misconfiguration & info-disclosure audit against Grafana (skips DoS tests)
nikto -h http://127.0.0.1 -p 3000 -nossl -Tuning 123b -maxtime 60s -o nikto_audit.csv -Format csv

# Run all checks except Denial of Service and export to HTML
nikto -h http://127.0.0.1 -p 3000 -nossl -Tuning x6 -o nikto_report.htm -Format htm
```

---

## 4. Dashboard & Query Load Testing (`wrk`, `hey`)

Populates `benchmarks/results/benchmark_query_latency.csv` (`requests_per_sec`, `latency_p50_ms`, `latency_p95_ms`, `latency_p99_ms`, `failed_requests`).

| Tool | Key Flags | Best Used For |
| :--- | :--- | :--- |
| **`wrk`** | `-t<threads> -c<conns> -d<dur> --latency` | Sustained high-concurrency `GET` load with latency percentiles |
| **`hey`** | `-z <dur> -c <conns> -m POST -d "<body>"` | Sending raw SQL queries via HTTP `POST` to ClickHouse without URL encoding |

```bash
# Legacy prototype (GET http://127.0.0.1:8080) - 4 threads, 50 concurrent clients, 30s
wrk -t4 -c50 -d30s --latency http://127.0.0.1:8080

# Modern prototype (ClickHouse POST via hey) - 50 concurrent clients, 30s
hey -z 30s -c 50 -m POST \
  -d "SELECT SrcAddr, count() AS flows, sum(Bytes) AS total_bytes FROM flows_db.flows GROUP BY SrcAddr ORDER BY total_bytes DESC LIMIT 10" \
  "http://admin:labpass@127.0.0.1:8123/"

# Modern prototype (ClickHouse GET via wrk)
wrk -t4 -c50 -d30s --latency \
  "http://admin:labpass@127.0.0.1:8123/?query=SELECT%20SrcAddr,count(),sum(Bytes)%20FROM%20flows_db.flows%20GROUP%20BY%20SrcAddr%20ORDER%20BY%20sum(Bytes)%20DESC%20LIMIT%2010"
```

---

## 5. System, Container, Storage & UDP Drop Monitoring

### UDP Packet Drops (`nstat`, `netstat`)

Populates `udp_packet_drops` and `drop_rate_percent` in `benchmarks/results/benchmark_ingestion_metrics.csv`.

```bash
# Reset delta baseline in nstat before a stress run
nstat -z UdpInDatagrams UdpInErrors UdpRcvbufErrors

# Print exact deltas after the stress run completes
nstat UdpInDatagrams UdpInErrors UdpRcvbufErrors

# Cumulative human-readable alternative since system boot
netstat -su | grep -E "packets received|receive buffer errors|packet receive errors"
```

### CPU, RAM & Disk I/O (`pidstat`, `iostat`, `docker stats`)

Populates `benchmarks/results/benchmark_system_resources.csv`.

```bash
# Legacy native processes: CPU (-u), RAM (-r), and Disk I/O (-d) every 1 second
pidstat -u -r -d -C "nfcapd|nfdump|python3" 1

# Host-wide disk I/O throughput and utilization every 1 second (skip idle devices)
iostat -xz 1

# Modern Docker stack: single CSV-ready snapshot (loop inside collect_metrics.sh)
docker stats --no-stream --format "{{.Name}},{{.CPUPerc}},{{.MemUsage}},{{.NetIO}},{{.BlockIO}}"
```

### Storage Footprint & Compression (`du`, ClickHouse `system.parts`)

Populates `benchmarks/results/benchmark_storage_footprint.csv`.

```bash
# Legacy disk usage and total stored flows
du -sh /var/log/netflow
nfdump -R /var/log/netflow -I | grep -i "Flows"

# Modern ClickHouse total flows, uncompressed size, on-disk size, and compression ratio
docker exec clickhouse clickhouse-client -u admin --password labpass -q "
SELECT
    sum(rows) AS total_flows_stored,
    round(sum(data_uncompressed_bytes) / 1048576, 2) AS raw_size_mb,
    round(sum(data_compressed_bytes) / 1048576, 2) AS disk_size_mb,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS compression_ratio
FROM system.parts
WHERE database = 'flows_db' AND table = 'flows' AND active
FORMAT CSVWithNames"
```

---

## 6. Standard Before vs. After Test Run Order

1. **Reset state:** Clear `/var/log/netflow/*` (legacy) or run `TRUNCATE TABLE flows_db.flows` (modern), then execute `nstat -z UdpInDatagrams UdpInErrors UdpRcvbufErrors` to zero the delta counters.
2. **Start monitoring:** Launch `benchmarks/monitoring/collect_metrics.sh` (`pidstat`, `docker stats`, and `iostat`) in the background.
3. **Run ingestion & attack workloads:** Replay the identical flow workload via `nfreplay` or `synthetic_netflow.py`, and execute the controlled `nmap` / `nikto` runs to verify SOC alert visibility.
4. **Capture ingestion & storage metrics:** Run `nstat` for UDP drops, record the data freshness delay, and collect `du` / ClickHouse `system.parts` storage numbers.
5. **Run query load tests:** Execute `wrk` or `hey` at 10, 50, and 100 concurrent clients (`-c 10`, `-c 50`, `-c 100`) and log the p50, p95, and p99 latencies.
6. **Stop monitoring & generate charts:** Terminate `collect_metrics.sh` and run `python3 benchmarks/results/plot_results.py` to produce the Before vs. After comparison graphs.