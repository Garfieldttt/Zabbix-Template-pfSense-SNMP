# Zabbix Template pfSense by SNMP

This Zabbix template enables full monitoring of a pfSense firewall via SNMP (bsnmpd). No Zabbix agent is required. It collects system health, network interface statistics, pf firewall state, CPU and memory usage, storage utilization, disk I/O, TCP/IP statistics, load average, and version information.

Works with pfSense CE and pfSense Plus.

---

## Requirements

- Zabbix Server 7.0 or higher
- pfSense 2.6 or higher (CE or Plus)
- SNMP service enabled on pfSense (see setup below)

---

## 1. Enable SNMP on pfSense

1. Go to **Services → SNMP**
2. Enable the SNMP service
3. Set **Polling Port**: `161`
4. Set **Read Community String**: e.g. `public` (use a strong value in production)
5. **System Location** and **System Contact**: optional but recommended
6. Under **Modules**, enable at minimum:
   - `mibII` — system, interfaces, TCP/IP statistics
   - `hostres` — CPU cores, memory, storage (HOST-RESOURCES-MIB)
   - `ucd` — load average, memory details, disk I/O (UCD-SNMP MIB)
   - `pf` — pf firewall statistics (requires `snmp_pf.so` module)
7. **Bind Interfaces**: select the interface Zabbix can reach (e.g. LAN)
8. Click **Save**

> **Note for pf monitoring:** The `pf` module (`snmp_pf.so`) must be loaded. If pf items return "not supported", verify that `snmp_pf.so` is listed under Modules and that the SNMP service was restarted after enabling it.

---

## 2. Installation

1. Download `template_pfsense_snmp.yaml`
2. In Zabbix: **Data collection → Templates → Import**
3. Create a new host:
   - **Data collection → Hosts → Create host**
   - Host name: e.g. `pfsense01`
   - Template: `pfSense by SNMP`
   - Group: e.g. `Network appliances`
   - Interfaces: add **SNMP interface** with the pfSense IP and port `161`
4. Set the required macros on the host (see below)

---

## 3. Macros

### Required

| Macro | Example | Description |
|-------|---------|-------------|
| `{$SNMP_COMMUNITY}` | `public` | SNMP community string |

### Threshold Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$CPU.UTIL.WARN}` | `80` | CPU warning threshold (%) |
| `{$CPU.UTIL.CRIT}` | `90` | CPU critical threshold (%) |
| `{$MEMORY.UTIL.MAX}` | `90` | Memory warning threshold (%) |
| `{$MEMORY.SWAP.WARN}` | `80` | Swap usage warning threshold (%) |
| `{$STORAGE.UTIL.WARN}` | `80` | Storage warning threshold (%) |
| `{$STORAGE.UTIL.CRIT}` | `90` | Storage critical threshold (%) |
| `{$PF.STATE.UTIL.WARN}` | `80` | pf state table warning threshold (%) |
| `{$PF.STATE.UTIL.CRIT}` | `90` | pf state table critical threshold (%) |
| `{$IF.UTIL.MAX}` | `90` | Interface utilization warning threshold (%) |
| `{$IF.ERRORS.WARN}` | `2` | Interface error rate warning threshold (errors/s) |
| `{$PF.IF.BLOCK.WARN}` | `100` | pf block rate warning threshold (packets/s) |
| `{$PF.TABLE.COUNT.WARN}` | `100` | pf table entry count warning threshold |
| `{$LOAD.AVG.WARN}` | `4` | Load average warning threshold |
| `{$TCP.ESTAB.MAX}` | `10000` | Max established TCP connections before warning |
| `{$ICMP_LOSS_WARN}` | `20` | ICMP packet loss warning threshold (%) |
| `{$ICMP_RESPONSE_TIME_WARN}` | `0.15` | ICMP response time warning threshold (seconds) |
| `{$SYSTEM.USERS.MAX}` | `0` | Max logged-in users before alert (0 = disabled) |
| `{$PFSENSE.VERSION.EXPECTED}` | `2.8.1` | Expected pfSense CE version for update check |

### Filter Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$NET.IF.IFNAME.MATCHES}` | `^.*$` | Interface name inclusion filter (regex) |
| `{$NET.IF.IFNAME.NOT_MATCHES}` | `^(pflog\|pfsync\|enc\|lo)\d*$` | Interface name exclusion filter — internal pf interfaces |
| `{$DISK.IO.DEVICE.NOT_MATCHES}` | `^pass[0-9]` | Disk I/O device exclusion filter |

### Alert Enable/Disable Macros

Set to `0` to suppress a trigger globally. Interface/pf macros support context for per-instance suppression.

| Macro | Default | Description |
|-------|---------|-------------|
| `{$ENABLE_LINK_DOWN_ALERT}` | `1` | Interface link down trigger |
| `{$ENABLE_IF_UTIL_ALERT}` | `1` | Interface high utilization trigger |
| `{$ENABLE_IF_ERROR_ALERT}` | `1` | Interface error rate trigger |
| `{$ENABLE_CPU_CORE_ALERT}` | `1` | Per-core CPU high utilization trigger |
| `{$ENABLE_STORAGE_ALERT}` | `1` | Storage high utilization trigger |
| `{$ENABLE_PF_BLOCK_ALERT}` | `1` | pf interface block rate trigger |
| `{$ENABLE_PF_TABLE_ALERT}` | `1` | pf table entry count trigger |

**Context macro example** — suppress link-down alert for a specific interface:
- Create a host macro: `{$ENABLE_LINK_DOWN_ALERT:"em1"}` = `0`

---

## 4. Discovery Rules

| Rule | Source | Discovers |
|------|--------|-----------|
| `net.if.discovery` | IF-MIB ifTable | Network interfaces with traffic, errors, status |
| `cpu.discovery` | HOST-RESOURCES-MIB | CPU cores with per-core utilization |
| `disk.io.discovery` | UCD-SNMP diskIOTable | Block devices with read/write throughput |
| `storage.discovery` | HOST-RESOURCES-MIB hrStorageTable | Filesystems with capacity and utilization |
| `pf.if.discovery` | BEGEMOT-PF-MIB pfInterfacesTable | pf interfaces with pass/block packet counters |
| `pf.table.discovery` | BEGEMOT-PF-MIB pfTablesTable | pf tables (e.g. sshguard, snort2c) with entry counts |
| `pf.label.discovery` | BEGEMOT-PF-MIB pfLabelsTable | pf firewall rules with traffic byte/packet counters |

---

## 5. Triggers

### Host-Level

| Trigger | Severity | Description |
|---------|----------|-------------|
| Unavailable by ICMP ping | Disaster | No ICMP response |
| High ICMP packet loss | Average | Loss > `{$ICMP_LOSS_WARN}`% |
| High ICMP response time | Warning | Response > `{$ICMP_RESPONSE_TIME_WARN}` s |
| System version has changed | Info | sysDescr changed since last check |
| Update available | Info | Installed version differs from `{$PFSENSE.VERSION.EXPECTED}` |
| Device has been restarted | Warning | Uptime < 10 minutes |
| pf firewall is not running | Disaster | pf reports not running |
| pf state table critically full | High | State table > `{$PF.STATE.UTIL.CRIT}`% |
| pf state table high utilization | Average | State table > `{$PF.STATE.UTIL.WARN}`% |
| pf dropping packets due to memory limits | High | pf memory limit drops detected |
| High inbound packet drop rate | Warning | High packet drop rate on pf input |
| Malformed packets detected (bad offset) | Warning | pf bad offset counter increasing |
| High system load average | Warning | Load average > `{$LOAD.AVG.WARN}` |
| CPU utilization critically high | High | CPU > `{$CPU.UTIL.CRIT}`% |
| High CPU utilization | Warning | CPU > `{$CPU.UTIL.WARN}`% |
| High memory utilization | Warning | Memory > `{$MEMORY.UTIL.MAX}`% |
| High swap utilization | Warning | Swap > `{$MEMORY.SWAP.WARN}`% |
| IP forwarding disabled | Disaster | IP forwarding is off — firewall cannot route |
| User logged in | Warning | Logged-in user count > `{$SYSTEM.USERS.MAX}` |
| High TCP established connections | Warning | TCP established > `{$TCP.ESTAB.MAX}` |
| High TCP retransmit rate | Warning | TCP retransmit segments increasing rapidly |
| High TCP connection failure rate | Warning | TCP connection failures increasing |
| Elevated UDP input error rate | Warning | UDP input errors increasing |
| IP inbound datagrams being discarded | Warning | IP discard counter increasing |

### Interface Prototypes

| Trigger | Severity |
|---------|----------|
| Interface link down | Average |
| High inbound traffic utilization | Warning |
| High outbound traffic utilization | Warning |
| High inbound error rate | Warning |
| High outbound error rate | Warning |

### CPU / Storage Prototypes

| Trigger | Severity |
|---------|----------|
| CPU core critically high utilization | Average |
| Storage critically full | High |
| Storage usage over warning threshold | Average |

### pf Firewall Prototypes

| Trigger | Severity |
|---------|----------|
| pf interface: High inbound block rate | Warning |
| pf interface: High outbound block rate | Warning |
| pf table: Entry count over threshold | Warning |

---

## 6. Dashboard

The template includes a pre-built dashboard **"pfSense Overview"** with the following pages:

| Page | Contents |
|------|----------|
| Overview | Hostname, location, uptime, OS description, IP forwarding, pf status, pf state table, CPU%, memory%, active problems |
| Interfaces | Traffic graphs per interface (LLD), link status |
| CPU & Storage | Per-core CPU utilization, storage utilization, disk I/O throughput |
| pf Firewall | Pass vs block packets per pf interface (inbound + outbound) |
| pf Tables | Match rate per pf table (e.g. sshguard blocked IPs) |
| pf Rules | Traffic counters per named pf firewall rule |

---

## 7. Notes

- **pf module not loaded:** If all pf items show "not supported", the `snmp_pf.so` module is not active. Enable it under Services → SNMP → Modules and restart the SNMP service.
- **pf Labels (rule counters):** Only rules with a `label` in `pf.conf` are discoverable. Rules without labels do not appear in the BEGEMOT-PF-MIB pfLabelsTable. Named rules in pfSense can be set under Firewall → Rules → Advanced → Label.
- **Interface filter:** `pflog`, `pfsync`, `enc`, and `lo` interfaces are excluded by default via `{$NET.IF.IFNAME.NOT_MATCHES}`. Add additional interfaces to suppress if needed.
- **Version check:** `{$PFSENSE.VERSION.EXPECTED}` must be updated manually when a new pfSense CE release is available. The template fetches the available version from the Netgate docs page.
- **pfSense Plus:** The version check is tuned for pfSense CE (`2.x.y` format). For pfSense Plus, adjust the `{$PFSENSE.VERSION.EXPECTED}` macro and the version parsing regex accordingly.
- **CPU temperatures:** Not available via SNMP on pfSense. Requires a custom agent or script.
