# Akvorado Lab Guide
### Deploying a Flow Collection Stack with Docker

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Understanding Flow Protocols](#2-understanding-flow-protocols)
3. [VM Prerequisites & Minimum Specs](#3-vm-prerequisites--minimum-specs)
4. [Install Docker & Docker Compose](#4-install-docker--docker-compose)
5. [Prepare the Working Directory](#5-prepare-the-working-directory)
6. [Configure Akvorado](#6-configure-akvorado)
7. [Start the Docker Stack](#7-start-the-docker-stack)
8. [Configure Network Devices to Send Flows](#8-configure-network-devices-to-send-flows)
   - 8.1 [MikroTik (NetFlow)](#81-mikrotik-netflow)
   - 8.2 [Arista (sFlow)](#82-arista-sflow)
9. [Verify Flow Receipt](#9-verify-flow-receipt)
10. [Troubleshooting Reference](#10-troubleshooting-reference)
11. [Appendix A: Removing Akvorado from the VM](#appendix-a-removing-akvorado-from-the-vm)
12. [Appendix B: Securing the Installation (HTTPS + Basic Auth)](#appendix-b-securing-the-installation-https--basic-auth)

---

## 1. Introduction

**Akvorado** is an open-source network flow collector, enricher, and visualiser. It receives flow telemetry exported from your network devices, enriches it with metadata (such as interface names and AS numbers), stores it in a ClickHouse columnar database, and presents it through a web-based UI for traffic analysis.

By the end of this lab you will have:

- A fully operational Akvorado stack running in Docker
- At least one network device exporting flows to your collector
- Verified traffic visibility in the Akvorado web interface

### Lab Topology

```
┌─────────────────────┐         Flow Export          ┌──────────────────────────────┐
│  Network Device(s)  │  ──────────────────────────► │         Lab VM               │
│                     │   UDP (NetFlow or sFlow)      │                              │
│  - MikroTik Router  │                               │  ┌────────────────────────┐  │
│  - Arista Switch    │                               │  │  Docker Compose Stack  │  │
└─────────────────────┘                               │  │                        │  │
                                                      │  │  akvorado  (inlet)     │  │
                                                      │  │  kafka / redpanda      │  │
                                                      │  │  clickhouse            │  │
                                                      │  │  akvorado  (console)   │  │
                                                      │  └────────────────────────┘  │
                                                      │                              │
                                                      │  Web UI → http://VM-IP:8081  │
                                                      └──────────────────────────────┘
```

---

## 2. Understanding Flow Protocols

Before configuring anything, it helps to understand what your network devices are actually sending.

### What is a Flow?

A **flow** is a summary of network traffic between two endpoints. Rather than capturing every packet (as a full packet capture would), your router or switch watches traffic passing through an interface and periodically exports summarised records. Each record typically contains:

| Field | Example |
|---|---|
| Source IP | 192.168.1.10 |
| Destination IP | 8.8.8.8 |
| Protocol | TCP |
| Source Port | 54321 |
| Destination Port | 443 |
| Bytes transferred | 142,400 |
| Packets | 98 |
| Interface (in/out) | ether1 |
| Timestamp | 2024-03-15 10:23:01 |

This makes flow data extremely useful for traffic analysis, capacity planning, and security investigation — at a fraction of the storage cost of packet capture.

---

### NetFlow (used by MikroTik)

**NetFlow** was developed by Cisco and has become the most widely implemented flow export standard. There are several versions:

| Version | Notes |
|---|---|
| v5 | Oldest, fixed record format, IPv4 only. Still widely supported. |
| v9 | Template-based, supports IPv6, MPLS, BGP fields. Flexible. |
| IPFIX | "NetFlow v10" — the IETF standard. Most extensible. |

**How it works:**

1. The router identifies a new flow on a monitored interface.
2. It tracks the flow in a local flow cache.
3. When the flow ends (TCP FIN/RST) or a timeout expires, the record is exported to the configured **collector** (your Akvorado VM).
4. Records are sent over **UDP** (connectionless — if a packet is dropped, the record is lost).

**Key timers to be aware of:**

- **Active timeout** — exports a record for long-lived flows that haven't ended (e.g. every 60 seconds).
- **Inactive timeout** — exports a record when a flow has been idle (e.g. for 15 seconds).

**MikroTik** implements NetFlow v5 and v9 via its Traffic Flow feature. We will use **v9** in this lab as it supports more fields and IPv6.

---

### sFlow (used by Arista)

**sFlow** (sampled flow) works differently from NetFlow. Rather than tracking every flow, sFlow takes **statistical packet samples** directly from the hardware forwarding plane.

**How it works:**

1. The switch samples 1 in every *N* packets passing through a monitored interface (e.g. 1 in 1024).
2. Each sampled packet is encapsulated along with interface counters and sent to the **sFlow collector** as a UDP datagram.
3. The collector statistically extrapolates traffic volumes from the samples.

**Key differences from NetFlow:**

| | NetFlow / IPFIX | sFlow |
|---|---|---|
| Method | Flow cache tracking | Packet sampling |
| Accuracy | Exact (for tracked flows) | Statistical (sampled) |
| Hardware requirement | Needs flow cache in software/ASIC | Very low — samples handled in ASIC |
| High-speed suitability | Can struggle at very high rates | Designed for high-speed interfaces |
| Header depth | Flow metadata only | Can include packet header payload |

**Arista** EOS has native sFlow support and can sample at high rates from hardware. We will configure sFlow on an Arista switch in Section 8.2.

---

### Understanding Sample Rates

Sample rate is one of the most important tuning decisions when deploying sFlow, and it has a direct impact on the router, the collector, and your storage. Getting the intuition right here will save a lot of head-scratching later.

#### What the sample rate number actually means

A sample rate of **1:1024** means the hardware forwards one packet out of every 1024 to the sFlow agent for export. It does **not** mean every 1024th packet in sequence — the selection is random, which is important for statistical validity.

The collector receives these samples and multiplies the observed traffic by the sample rate to estimate total volume. If a sampled packet shows 1,500 bytes and the sample rate is 1:1024, that one sample represents approximately **1.5 MB** of traffic on the wire.

#### Impact on the router / switch

The key point with sFlow is that the sampling itself happens in the **ASIC forwarding plane** — the hardware does the heavy lifting, not the CPU. This is why sFlow scales to 100G interfaces without breaking a sweat. However, the process of **encapsulating and exporting** the sample datagrams does involve the control plane to some degree:

| Sample Rate | Packets/sec at 1Gbps (avg 500B pkt) | Exported sFlow datagrams/sec |
|---|---|---|
| 1:64 | ~250,000 pkt/s | ~3,900 datagrams/s |
| 1:512 | ~250,000 pkt/s | ~490 datagrams/s |
| 1:1024 | ~250,000 pkt/s | ~245 datagrams/s |
| 1:4096 | ~250,000 pkt/s | ~61 datagrams/s |

> The line rate stays the same — only the number of exported samples changes. A low sample rate (e.g. 1:64) at high traffic volumes can generate enough export datagrams to stress the management plane. On most modern hardware 1:512 to 1:1024 is a safe starting range for a 1G interface.

#### Impact on the collector (Akvorado + Kafka)

Every sFlow datagram received by Akvorado's inlet is parsed, enriched (SNMP lookup, GeoIP, AS lookup), and pushed onto the Kafka/Redpanda queue. The collector CPU cost scales linearly with the number of datagrams per second, not with the underlying line rate.

- **Low sample rate (e.g. 1:64)** — high datagram rate, high CPU and memory pressure on the collector, more enrichment lookups per second.
- **High sample rate (e.g. 1:4096)** — low datagram rate, very low collector load, but coarser traffic visibility (small flows may be missed entirely).

For a lab environment with light traffic, a sample rate of **1:256 or 1:512** will give you enough data to see meaningful results quickly without overwhelming anything.

#### Impact on storage (ClickHouse)

Each enriched flow record that passes through Kafka gets written to ClickHouse as a row. Storage consumption is therefore directly proportional to the number of samples received:

- A lower sample rate = more rows per unit of time = faster disk growth.
- ClickHouse compresses columnar data very efficiently, but the write amplification from a very low sample rate can still be significant at scale.

A rough mental model for estimating storage:

```
rows/day = (avg packets/sec) × (86,400 sec/day) ÷ sample_rate
```

For example, at 50,000 packets/second average across all interfaces with a 1:1024 sample rate:

```
50,000 × 86,400 ÷ 1024 ≈ 4.2 million rows/day
```

ClickHouse typically stores flow rows at **50–150 bytes per row** after compression depending on field cardinality, so 4.2M rows/day ≈ **200–600 MB/day**. Easily manageable. At 1:64 that becomes 3–10 GB/day from the same traffic volume.

#### Choosing a sample rate — summary

| Scenario | Suggested Rate | Rationale |
|---|---|---|
| Lab / low traffic | 1:256 – 1:512 | See data quickly, minimal load |
| Production, 1G interfaces | 1:1024 | Good balance of visibility and load |
| Production, 10G interfaces | 1:2048 – 1:4096 | Prevents collector overload |
| Production, 100G interfaces | 1:8192 – 1:16384 | ASIC handles it; keep datagram rate sane |

> **Rule of thumb:** target roughly **200–500 sFlow datagrams per second per interface** as a comfortable collector load. Work backwards from your expected traffic rate to choose N accordingly.

---

### Which port does Akvorado listen on?

| Protocol | Default UDP Port |
|---|---|
| NetFlow v5 / v9 | 2055 |
| IPFIX | 4739 |
| sFlow | 6343 |

Akvorado's **inlet** component listens on these ports and handles all three protocol families.

---

## 3. VM Prerequisites & Minimum Specs

Your lab VM should meet the following minimum requirements:

| Resource | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| CPU | 2 vCPUs | 4 vCPUs |
| RAM | 4 GB | 8 GB |
| Disk | 20 GB | 40 GB |
| Network | 1 NIC, reachable from devices | Static IP assigned |

> **Note:** ClickHouse is storage-intensive in production. 20 GB is sufficient for this lab but should not be used as a sizing reference for production deployments.

### Pre-flight Checks

Before proceeding, confirm the following on your VM:

```bash
# Confirm your IP address
ip addr show

# Confirm you have sudo access
sudo whoami

# Check Ubuntu version
lsb_release -a

# Confirm internet connectivity (needed to pull images)
curl -I https://hub.docker.com
```

### Firewall / Port Requirements

Ensure the following ports are accessible on the VM from your network devices:

| Port | Protocol | Purpose |
|---|---|---|
| 2055 | UDP | NetFlow v5/v9 (MikroTik) |
| 6343 | UDP | sFlow (Arista) |
| 8081 | TCP | Akvorado Web UI |

If UFW is enabled on your VM:

```bash
sudo ufw allow 2055/udp
sudo ufw allow 6343/udp
sudo ufw allow 8081/tcp
sudo ufw status
```

---

## 4. Install Docker & Docker Compose

We install Docker Engine from the official Docker repository to ensure we have a current, supported version.

### Remove any old Docker packages

```bash
sudo apt-get remove -y docker docker-engine docker.io containerd runc
```

### Install dependencies

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

### Add Docker's GPG key and repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine and the Compose plugin

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Post-install configuration

```bash
# Add your user to the docker group (avoids needing sudo for every command)
sudo usermod -aG docker $USER

# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl start docker
```

> **Important:** Log out and back in (or run `newgrp docker`) for the group change to take effect.

### Verify the installation

```bash
docker version
docker compose version
docker run hello-world
```

---

## 5. Prepare the Working Directory

### Create a working directory

```bash
mkdir ~/akvorado
cd ~/akvorado
```

### Download the quickstart bundle

Akvorado publishes a quickstart tarball with each release that contains the Docker Compose file and a set of example configuration files. This gives us a complete, version-matched starting point to work from.

```bash
curl -sL https://github.com/akvorado/akvorado/releases/latest/download/docker-compose-quickstart.tar.gz | tar zxvf -
```

This will extract all files into the current directory. You should see output listing the files as they are extracted.

> **Lab pack users:** A pre-configured `docker-compose.yml` is included in your lab pack with all the changes from this guide already applied, including the TLS and basic auth configuration from Appendix B. Copy it into your working directory and follow the setup instructions in the file header before starting the stack:
> ```bash
> cp /path/to/lab-pack/docker-compose.yml ~/akvorado/
> ```

### Review what was extracted

```bash
ls -lh
```

Take a moment to look at the files before starting anything. You should see at minimum:

- `docker-compose.yml` — defines all the services
- One or more `.yaml` config files for Akvorado itself (e.g. `akvorado.yaml` or files in a `config/` subdirectory)

```bash
# Get a feel for the compose services
cat docker-compose.yml
```

> **Identify the following services in the compose file:**
> - `akvorado` — the main application (inlet + orchestrator + console)
> - `clickhouse` — the columnar database
> - `kafka` or `redpanda` — the message queue between inlet and ClickHouse
> - `zookeeper` — present if Kafka is used (not needed with Redpanda)

### Review the service relationships

```
flow packets
    │
    ▼
akvorado inlet
    │  (writes enriched flows)
    ▼
kafka / redpanda
    │  (consumed by)
    ▼
akvorado orchestrator ──► clickhouse
    │
    ▼
akvorado console ──► Web UI
```

---

## 6. Configure Akvorado

The quickstart bundle includes example configuration files that are already wired up to the compose stack. Rather than writing config from scratch, we edit the provided examples to add our flow sources.

### Locate the Akvorado config file

```bash
# The config file is likely akvorado.yaml or inside a config/ directory
find . -name "*.yaml" | grep -v docker-compose
```

Open the config file in your editor of choice:

```bash
nano akvorado.yaml
```

### Add flow listeners

Find the `inlet` section and confirm or add listeners for both NetFlow and sFlow. The example config may already have one or both — adjust the ports if needed:

```yaml
inlet:
  flow:
    inputs:
      - type: udp
        listen: "0.0.0.0:2055"   # NetFlow v9 (MikroTik)
        decoder: netflow
      - type: udp
        listen: "0.0.0.0:6343"   # sFlow (Arista)
        decoder: sflow
```

### Configure SNMP enrichment (optional but recommended)

Akvorado can poll your devices via SNMP to resolve interface indexes to human-readable names (e.g. `ether1` or `Ethernet1`). Add your device community string under the `inlet` section:

```yaml
inlet:
  snmp:
    communities:
      - community: public
```

> **Note:** The configuration schema can vary between Akvorado releases. The example files extracted from the tarball are the authoritative reference for the version you are running. If a key in the examples here doesn't match what you see in your extracted config, follow the extracted file's structure and refer to the [official documentation](https://akvorado.net/docs).

### Applying config changes without restarting the whole stack

After editing any config file (e.g. `config/outlet.yaml`, `config/inlet.yaml`), you only need to restart the individual container whose config changed — not the entire stack. The config files are bind-mounted from the host into the containers, so Docker already has the updated file; the container just needs to re-read it on startup.

```bash
# Restart a single service — note the full service names as defined in docker-compose.yml
docker compose restart akvorado-outlet
docker compose restart akvorado-inlet
docker compose restart akvorado-orchestrator
```

This takes a few seconds and avoids disrupting ClickHouse, Kafka, and any other services that don't need to be touched.

> **Important distinction:** `docker compose restart` only works for config file changes. If you ever modify `docker-compose.yml` itself (port mappings, environment variables, volume mounts), you need to recreate the affected container instead:
> ```bash
> docker compose up -d --force-recreate akvorado-outlet
> ```

---

## 7. Start the Docker Stack

### Pull images and start all services

```bash
cd ~/akvorado
docker compose up -d
```

This will download all required container images on first run. This may take a few minutes depending on your connection.

### Verify all containers are running

```bash
docker compose ps
```

All services should show a status of `running` or `healthy`. If any show `exited`, check the logs.

### Check logs

```bash
# All services
docker compose logs -f

# A specific service
docker compose logs -f akvorado
docker compose logs -f clickhouse
```

### Access the Web UI

Once all containers are healthy, open a browser and navigate to:

```
http://<YOUR-VM-IP>:8081
```

The UI will initially be empty — we need network devices to send flows before data appears.

---

## 8. Configure Network Devices to Send Flows

Replace `<AKVORADO-VM-IP>` in all examples below with the actual IP address of your lab VM.

---

### 8.1 MikroTik (NetFlow)

MikroTik implements flow export via the **Traffic Flow** feature, available on RouterOS.

#### Enable Traffic Flow via the CLI (SSH or Winbox terminal)

```routeros
# Enable Traffic Flow and set the version to IPFIX (v10) or v9
/ip traffic-flow
set enabled=yes interfaces=all active-flow-timeout=1m inactive-flow-timeout=15s

# PLACEHOLDER: confirm correct command for v9 vs IPFIX on your RouterOS version
/ip traffic-flow version set version=9

# Configure the export target (your Akvorado VM)
/ip traffic-flow target
add dst-address=<AKVORADO-VM-IP> port=2055 version=9
```

#### Verify on MikroTik

```routeros
/ip traffic-flow print
/ip traffic-flow target print
```

> **Expected:** You should see `enabled: yes` and your collector IP listed as a target.

> **Interface names in Akvorado:** MikroTik exports interface names using the **Comment** field set on each interface, not the interface identifier (e.g. `ether1`). If an interface has no comment set, the collector will receive a generic index or blank name, which makes traffic hard to read in the UI. Set a meaningful comment on every interface you want to monitor before sending flows:
> ```routeros
> /interface set ether1 comment="WAN-Upstream"
> /interface set ether2 comment="LAN-Core"
> ```
> These comments will appear as the interface description in Akvorado's flow records and dashboards.

> **RouterOS v7 + NetFlow v9 — missing sampling rate:** MikroTik RouterOS v7 does not include the sampling rate in its NetFlow v9 export data. Without this, Akvorado cannot correctly scale traffic volumes and flow counts will appear orders of magnitude lower than reality. To fix this, explicitly tell Akvorado the sampling rate by adding the following to your `config/outlet.yaml` file:
> ```yaml
> core:
>   default-sampling-rate: 1
>
> asn-providers:
    - flow-except-default-route
    - geo-ip
> ```
> A value of `1` means every packet is being tracked (no sampling), which is correct for NetFlow — MikroTik's Traffic Flow records every flow, it simply omits the sampling rate field in the export. After saving the file, apply the change with:
> ```bash
> docker compose restart akvorado-outlet
> ```

---

### 8.2 Arista (sFlow)

Arista EOS supports sFlow natively via hardware ASIC sampling.

#### Enable sFlow via the CLI (EOS)

```eos
# Enter configuration mode
configure terminal

# Set the sFlow sample rate (1 in 1024 packets)
sflow sample 1024

# Set the polling interval for interface counters (seconds)
sflow polling-interval 30

# Configure the collector destination
sflow destination <AKVORADO-VM-IP> 6343

# Enable sFlow globally
sflow run

# Enable sFlow on specific interfaces (repeat as needed)
interface Ethernet1
   sflow enable

# Save the configuration
end
write memory
```

Full EOS config example

```eos
sflow sample 4096
sflow sample truncate size 200
sflow polling-interval 20
sflow destination {ip_v4_address} 6343
sflow source-interface Loopback0
sflow extension bgp
no sflow extension router
sflow interface disable default
sflow run

interface Eth1/1
   sflow enable
   sflow egress enable
```


#### Verify on Arista

```eos
show sflow
show sflow interfaces
```

> **Expected:** `sFlow Status: enabled`, collector IP shown, and interface(s) listed as active.

### 8.3 Juniper (sFlow)

Juniper config example

```bash
set chassis fpc 0 sampling-instance flow0
set chassis fpc 0 inline-services flex-flow-sizing
set services flow-monitoring version-ipfix template ipv4 flow-active-timeout 60
set services flow-monitoring version-ipfix template ipv4 flow-inactive-timeout 60
set services flow-monitoring version-ipfix template ipv4 template-refresh-rate packets 1000
set services flow-monitoring version-ipfix template ipv4 template-refresh-rate seconds 60
set services flow-monitoring version-ipfix template ipv4 option-refresh-rate packets 1000
set services flow-monitoring version-ipfix template ipv4 option-refresh-rate seconds 60
set services flow-monitoring version-ipfix template ipv4 ipv4-template
set services flow-monitoring version-ipfix template ipv6 flow-active-timeout 60
set services flow-monitoring version-ipfix template ipv6 flow-inactive-timeout 60
set services flow-monitoring version-ipfix template ipv6 template-refresh-rate packets 1000
set services flow-monitoring version-ipfix template ipv6 template-refresh-rate seconds 60
set services flow-monitoring version-ipfix template ipv6 option-refresh-rate packets 1000
set services flow-monitoring version-ipfix template ipv6 option-refresh-rate seconds 60
set services flow-monitoring version-ipfix template ipv6 ipv6-template
set forwarding-options sampling instance flow0 input rate 128
set forwarding-options sampling instance flow0 family inet output flow-server {ip_v4_address} port 9996
set forwarding-options sampling instance flow0 family inet output flow-server {ip_v4_address}  version-ipfix template ipv4
set forwarding-options sampling instance flow0 family inet output inline-jflow source-address {source_ip_v4_address} 
set forwarding-options sampling instance flow0 family inet6 output flow-server {ip_v4_address} port 9996
set forwarding-options sampling instance flow0 family inet6 output flow-server {ip_v4_address} version-ipfix template ipv6
set forwarding-options sampling instance flow0 family inet6 output inline-jflow source-address {source_ip_v4_address}
```

### 8.4 Cisco (NetFlow)

Cisco Flexible netflow config

* Flow exporter → sends records to collector
* Flow record → defines fields to export
* Flow monitor → ties record + exporter together

#### Enable Netflow via the CLI (IOS-XE)

```cli
flow record Akvorado
 match ipv4 tos
 match ipv4 protocol
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 collect routing source as 4-octet
 collect routing destination as 4-octet
 collect routing next-hop address ipv4
 collect transport tcp flags
 collect interface output
 collect interface input
 collect counter bytes
 collect counter packets
 collect timestamp sys-uptime first
 collect timestamp sys-uptime last
!

!
flow monitor AkvoradoMonitor
 exporter AkvoradoExport
 cache timeout inactive 10
 cache timeout active 60
 record Akvorado
!

flow exporter AkvoradoExport
 destination {ip_v4_address}
 source Loopback0
 transport udp 9996
```

---

## 9. Verify Flow Receipt

### Check the Akvorado inlet logs

After configuring your devices, watch the inlet logs for incoming flow records:

```bash
docker compose logs -f akvorado | grep -i "flow\|inlet\|receive"
```

You should start seeing log lines indicating received flow packets within 1–2 minutes of configuring your device (depending on active flow timeout).

### Check the Web UI

Refresh `http://<YOUR-VM-IP>:8081` in your browser. Within a few minutes of flows arriving you should see:

- Source/destination IPs populating the sankey diagram
- Traffic volume graphs beginning to build
- Interface information (if SNMP enrichment is configured)

### Quick connectivity test

If you suspect flows aren't arriving, you can verify with `tcpdump` on the VM:

```bash
# Check for incoming NetFlow packets
sudo tcpdump -i any udp port 2055 -n

# Check for incoming sFlow packets
sudo tcpdump -i any udp port 6343 -n
```

If you see packets here but nothing in the Akvorado UI, the issue is within the container stack.

---

## 10. Troubleshooting Reference

### Container won't start

```bash
# Check for port conflicts
sudo ss -ulnp | grep -E "2055|6343|8081"

# Check container exit reason
docker compose logs <service-name>
```

### No flows visible in the UI

| Check | Command |
|---|---|
| Are packets reaching the VM? | `sudo tcpdump -i any udp port 2055` |
| Is the Akvorado inlet running? | `docker compose ps` |
| Is the config file mounted correctly? | `docker compose exec akvorado cat /etc/akvorado/akvorado.yaml` |
| Are there errors in the inlet log? | `docker compose logs akvorado` |

### ClickHouse not accepting data

```bash
docker compose logs clickhouse
docker compose exec clickhouse clickhouse-client --query "SHOW DATABASES;"
```

### Resetting the stack (full wipe)

> **Warning:** This will delete all collected flow data.

```bash
cd ~/akvorado
docker compose down -v
docker compose up -d
```

---

## Appendix A: Removing Akvorado from the VM

Use this procedure to completely remove the Akvorado stack, all collected data, and optionally Docker itself from the VM. This is useful at the end of a lab session or before a clean reinstall.

### Step 1 — Stop and remove all containers, networks, and volumes

This removes all running containers, the networks Docker Compose created, and the named volumes where ClickHouse stores its data.

```bash
cd ~/akvorado
docker compose down -v
```

> The `-v` flag is what removes the data volumes. Without it, containers are stopped and removed but the ClickHouse data persists on disk.

### Step 2 — Remove the working directory

```bash
cd ~
rm -rf ~/akvorado
```

### Step 3 — Remove unused Docker images

The above steps remove containers and volumes but leave the downloaded images cached on disk. To free that space:

```bash
# Remove images that are no longer associated with a running or stopped container
docker image prune -a

# Confirm disk space has been recovered
docker system df
```

If you want a single command that cleans up everything Docker-related in one shot (images, containers, volumes, networks, build cache):

```bash
docker system prune -a --volumes
```

> **Warning:** `docker system prune -a --volumes` removes **all** Docker data on the VM, not just Akvorado's. If other Docker projects are running on the same VM, omit this step and use the targeted commands above instead.

### Step 4 (optional) — Remove Docker entirely

Only do this if you want to return the VM to a completely clean state with no Docker installation.

```bash
# Stop the Docker service
sudo systemctl stop docker

# Remove Docker packages
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Remove Docker's data directory (images, volumes, networks stored on disk)
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd

# Remove the Docker apt repository
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/keyrings/docker.gpg

# Clean up
sudo apt-get autoremove -y
```

Verify Docker is gone:

```bash
docker version
# Expected: "command not found"
```

### Summary of what each step removes

| Step | What is removed |
|---|---|
| `docker compose down -v` | Containers, networks, data volumes (ClickHouse data) |
| `rm -rf ~/akvorado` | Config files, compose file, working directory |
| `docker image prune -a` | Cached container images |
| `docker system prune -a --volumes` | All Docker data on the VM |
| Remove Docker packages | Docker Engine, Compose plugin, containerd |
| `rm -rf /var/lib/docker` | All on-disk Docker state |

---

## Appendix B: Securing the Installation (HTTPS + Basic Auth)

The default Akvorado quickstart stack runs over plain HTTP on port 8081 with no authentication — fine for a lab on a trusted network, but not suitable for any internet-facing or production deployment. This appendix documents how to add:

- **TLS via Let's Encrypt** — automatic certificate issuance and renewal using the ACME HTTP challenge
- **Basic authentication** — a username and password protecting the console

The stack already includes **Traefik** as a reverse proxy in front of all Akvorado services. All the changes below are made to Traefik's configuration inside `docker-compose.yml` — no changes to the Akvorado containers themselves are needed.

> **Prerequisites:**
> - Your VM must have a **public DNS A record** pointing to its IP address (e.g. `flows.example.com → 1.2.3.4`). Let's Encrypt needs to reach your VM over HTTP on port 80 to complete the challenge.
> - Ports **80** and **443** must be open on your VM's firewall.
> - This will not work with a private/internal IP address that is not reachable from the internet.

---

### Overview of what changes

Comparing the default compose file to the secured version, there are four areas of change:

| Area | Default | Secured |
|---|---|---|
| Traefik entrypoints | HTTP :8081 only | HTTP :80 (redirect), HTTPS :443, private :8080 |
| TLS | None | Let's Encrypt ACME via HTTP challenge |
| Authentication | None | bcrypt basic auth on the `websecure` entrypoint |
| Console router | HTTP only | HTTPS with cert resolver and domain |
| Volumes | No cert storage | `akvorado-letsencrypt` volume for cert persistence |

---

### Step 1 — Add the Let's Encrypt volume

Open your `docker-compose.yml` and add the new volume under the `volumes:` section at the top of the file:

```yaml
volumes:
  akvorado-kafka:
  akvorado-geoip:
  akvorado-clickhouse:
  akvorado-run:
  akvorado-console-db:
  akvorado-letsencrypt:       # <-- add this line
```

This volume persists the issued certificate across container restarts. Without it, Akvorado would request a new certificate every time Traefik starts, quickly hitting Let's Encrypt rate limits.

---

### Step 2 — Generate a bcrypt password hash

Basic auth in Traefik requires passwords stored as bcrypt hashes. Generate one on your VM:

```bash
# Install htpasswd if not already present
sudo apt-get install -y apache2-utils

# Generate a bcrypt hash for your chosen password
# Replace 'yourpassword' with something strong
htpasswd -nbB flows yourpassword
```

This will output something like:

```
flows:$2y$05$LbR0WmEdIsh0Z8Mv2o2p1uGLN3ZViZV7b3NWAPnx/7zuaDgPP5V/2
```

Copy this output — you will need it in the next step.

> **Important:** When pasting the hash into `docker-compose.yml`, every `$` sign must be escaped as `$$`. For example:
> `flows:$2y$05$abc...` becomes `flows:$$2y$$05$$abc...`
>
> This is a Docker Compose requirement — `$` is used for variable interpolation, so it must be doubled to be treated as a literal character.

---

### Step 3 — Update the Traefik service

Rather than replacing the entire `traefik:` block, the changes below are broken into exactly what is being added or modified. Lines marked `# ✚ ADD` are new; lines marked `# ✎ CHANGE` replace an existing line.

**In the `environment:` section — add these lines:**

```yaml
      # ✚ ADD — change the public entrypoint to redirect to HTTPS instead of serving directly
      TRAEFIK_ENTRYPOINTS_public_HTTP_MIDDLEWARES: ""
      TRAEFIK_ENTRYPOINTS_public_HTTP_REDIRECTIONS_ENTRYPOINT_TO: websecure
      TRAEFIK_ENTRYPOINTS_public_HTTP_REDIRECTIONS_ENTRYPOINT_SCHEME: https

      # ✚ ADD — new HTTPS entrypoint with basic auth middleware applied
      TRAEFIK_ENTRYPOINTS_websecure_ADDRESS: ":8443"
      TRAEFIK_ENTRYPOINTS_websecure_HTTP_MIDDLEWARES: compress@docker,basic-auth@docker

      # ✚ ADD — Let's Encrypt ACME configuration
      TRAEFIK_CERTIFICATESRESOLVERS_letsencrypt_ACME_EMAIL: "email@example.com"   # <-- your email
      TRAEFIK_CERTIFICATESRESOLVERS_letsencrypt_ACME_STORAGE: "/letsencrypt/acme.json"
      TRAEFIK_CERTIFICATESRESOLVERS_letsencrypt_ACME_HTTPCHALLENGE_ENTRYPOINT: "public"
```

**In the `labels:` section — add this line** (alongside the existing label lines):

```yaml
      # ✚ ADD — basic auth middleware definition; paste your escaped bcrypt hash here
      - "traefik.http.middlewares.basic-auth.basicauth.users=flows:$$2y$$05$$your-hash-here"
```

**In the `ports:` section — add these two lines** (the existing `8081:8081/tcp` line stays):

```yaml
      - 80:8081/tcp      # ✚ ADD — standard HTTP port, used for ACME challenge and redirect
      - 443:8443/tcp     # ✚ ADD — standard HTTPS port
```

**In the `volumes:` section — add this line:**

```yaml
      - akvorado-letsencrypt:/letsencrypt   # ✚ ADD — persist Let's Encrypt certificates
```

---

### Step 4 — Update the console service for HTTPS

Only the `labels:` block of `akvorado-console` needs to change. Add the four lines marked `# ✚ ADD` and replace the one line marked `# ✎ CHANGE`:

```yaml
  akvorado-console:
    # ... (all other settings unchanged) ...
    labels:
      - traefik.enable=true
      # ✚ ADD — enable TLS on the console router
      - traefik.http.routers.akvorado-console.tls=true
      # ✚ ADD — use the Let's Encrypt cert resolver defined in the Traefik service
      - traefik.http.routers.akvorado-console.tls.certresolver=letsencrypt
      # ✚ ADD — your public domain name (must match your DNS A record)
      - traefik.http.routers.akvorado-console.tls.domains[0].main=flows.example.com
      # ✎ CHANGE — was entrypoints=public, now serves over the HTTPS entrypoint
      - traefik.http.routers.akvorado-console.entrypoints=websecure
      # (all remaining existing labels stay exactly as they are)
      - traefik.http.routers.akvorado-console-debug.rule=PathPrefix(`/debug`)
      - traefik.http.routers.akvorado-console-debug.entrypoints=private
      - traefik.http.routers.akvorado-console-debug.service=akvorado-console
      - traefik.http.routers.akvorado-console-metrics.rule=PathPrefix(`/api/v0/console/metrics`)
      - traefik.http.routers.akvorado-console-metrics.service=akvorado-console
      - traefik.http.routers.akvorado-console-metrics.observability.accesslogs=false
      - "traefik.http.routers.akvorado-console.rule=!PathPrefix(`/debug`)"
      - traefik.http.routers.akvorado-console.priority=1
      - traefik.http.routers.akvorado-console.middlewares=console-auth
      - traefik.http.services.akvorado-console.loadbalancer.server.port=8080
      - traefik.http.middlewares.console-auth.headers.customrequestheaders.Remote-User=alfred
      - traefik.http.middlewares.console-auth.headers.customrequestheaders.Remote-Name=Alfred Pennyworth
      - traefik.http.middlewares.console-auth.headers.customrequestheaders.Remote-Email=alfred@example.com
      - metrics.enable=true
      - metrics.path=/api/v0/metrics
```

> **Note on the Remote-User headers:** Akvorado's console uses these headers to identify the logged-in user for display purposes. In a basic auth setup these are static values — change them to match the username you created in Step 2. For a more complete multi-user setup, an upstream identity provider (e.g. Authelia, Keycloak) would set these headers dynamically, but that is outside the scope of this lab.

---

### Step 5 — Open firewall ports and apply the changes

```bash
# Open the standard HTTP and HTTPS ports
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Apply the compose changes — recreate only the traefik and console containers
cd ~/akvorado
docker compose up -d --force-recreate traefik akvorado-console
```

Watch the Traefik logs to confirm the certificate is issued:

```bash
docker compose logs -f traefik | grep -i "acme\|certificate\|tls\|error"
```

Within a minute or two you should see log lines confirming the ACME challenge completed and a certificate was obtained for your domain.

---

### Step 6 — Verify

Open a browser and navigate to your domain:

```
https://flows.example.com
```

You should be prompted for a username and password (the credentials from Step 2), and the connection should be secured with a valid Let's Encrypt certificate.

To verify the certificate details:

```bash
echo | openssl s_client -connect flows.example.com:443 2>/dev/null | openssl x509 -noout -dates -subject
```

---

### Troubleshooting HTTPS setup

| Symptom | Likely cause | Check |
|---|---|---|
| Certificate not issued | DNS not resolving to VM | `dig flows.example.com` from external host |
| Certificate not issued | Port 80 not reachable | Confirm firewall and ISP allow inbound port 80 |
| Browser shows "invalid cert" | Traefik still starting up | Wait 1–2 min, check `docker compose logs traefik` |
| 401 Unauthorized loop | `$$` not escaped correctly in hash | Re-check the basicauth label in compose |
| HTTP not redirecting | `public` entrypoint redirect not applied | Confirm `TRAEFIK_ENTRYPOINTS_public_HTTP_REDIRECTIONS_*` vars are present |

> **Let's Encrypt rate limits:** Let's Encrypt allows a maximum of 5 certificate requests per domain per week on the production endpoint. If you are testing repeatedly, use the staging endpoint first by adding `TRAEFIK_CERTIFICATESRESOLVERS_letsencrypt_ACME_CASERVER: "https://acme-staging-v02.api.letsencrypt.org/directory"` to Traefik's environment. Staging certificates are not trusted by browsers but confirm the ACME flow works before using a production certificate.

---

*Document version: draft — commands to be verified against Akvorado release and RouterOS/EOS versions in use.*
