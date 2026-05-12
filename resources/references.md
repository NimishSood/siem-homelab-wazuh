# SIEM HomeLab References

This file lists the references, URLs, internal endpoints, configuration files, and validation artifacts that are explicitly used or referred to in Parts 1-6 of the SIEM HomeLab documentation.

It does not add outside reading or new sources. If something is listed here, it is because it appears in the completed deployment guides.

---

## 1. Local Deployment Guides

| Part | Guide | What It Covers |
|---|---|---|
| 1 | [Wazuh Indexer Deployment](../docs/part1-wazuh-indexer/wazuh-indexer-deployment.md) | Wazuh Indexer VM, certificate generation, OpenSearch tuning, service startup, cluster initialization, and Indexer verification. |
| 2 | [Wazuh Dashboard Deployment](../docs/part2-wazuh-dashboard/wazuh-dashboard-deployment.md) | Dashboard VM, certificate regeneration, Dashboard installation, `opensearch_dashboards.yml`, Wazuh Dashboard access, and initial no-server-connected state. |
| 3 | [Graylog Server Deployment](../docs/part3-graylog-server/graylog-server-deployment.md) | Graylog 7.0.5, MongoDB 8.0, Java 21, Wazuh Indexer backend connection, TLS truststore work, and backend compatibility fixes. |
| 4 | [Wazuh Manager Server Deployment](../docs/part4-wazuh-manager/wazuh-manager-deployment.md) | Wazuh Manager deployment, Graylog Raw/Plaintext TCP input, Fluent Bit forwarding, Wazuh Dashboard manager connection, manager hardening, agent groups, and SOCFortress rule notes. |
| 5 | [Wazuh Agent Deployment and Endpoint Telemetry](../docs/part5-wazuh-agents/wazuh-agents-deployment.md) | Debian and Windows Wazuh agents, Sysmon, Packetbeat, group-level log collection, endpoint validation, and Wazuh Manager troubleshooting. |
| 6 | [Graylog Extractors, Streams, and Wazuh Alert Routing](../docs/part6-graylog-routing/graylog-routing-deployment.md) | Graylog JSON extractor, Wazuh alert parsing, dedicated index set, Data Tiering, Wazuh Alerts stream, static `log_type=wazuh` field, and routing verification. |

---

## 2. External URLs and Download Sources

### 2.1 Operating System Source

| Source | URL | Used In |
|---|---|---|
| Ubuntu Server download page | `https://ubuntu.com/download/server` | Parts 1 and 2 host platform references. |

### 2.2 Wazuh Packages and Certificate Tooling

| Source | URL | Used For |
|---|---|---|
| Wazuh certificate tool | `https://packages.wazuh.com/4.14/wazuh-certs-tool.sh` | Generating Wazuh stack TLS certificates in Part 1. |
| Wazuh certificate configuration template | `https://packages.wazuh.com/4.14/config.yml` | Certificate node configuration in Part 1 and later certificate regeneration in Part 2. |
| Wazuh GPG key | `https://packages.wazuh.com/key/GPG-KEY-WAZUH` | Adding Wazuh package signing trust in Parts 1, 2, and 4. |
| Wazuh apt repository | `https://packages.wazuh.com/4.x/apt/ stable main` | Installing Wazuh Indexer, Dashboard, Manager, and Linux agent packages. |
| Debian Wazuh agent package | `https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.3-1_amd64.deb` | Debian 12 Wazuh agent installation in Part 5. |
| Windows Wazuh agent MSI | `https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.3-1.msi` | Windows Server 2025 Wazuh agent installation in Part 5. |

### 2.3 Graylog and MongoDB Package Sources

| Source | URL | Used For |
|---|---|---|
| MongoDB 8.0 GPG key | `https://www.mongodb.org/static/pgp/server-8.0.asc` | Installing MongoDB 8.0 for Graylog in Part 3. |
| MongoDB 8.0 apt repository | `https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse` | MongoDB package source in Part 3. |
| Graylog 7.0 repository package | `https://packages.graylog2.org/repo/packages/graylog-7.0-repository_latest.deb` | Installing Graylog 7.0.5 in Part 3. |

### 2.4 Fluent Bit

| Source | URL | Used For |
|---|---|---|
| Fluent Bit install script | `https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh` | Installing Fluent Bit on the Wazuh Manager in Part 4. |

### 2.5 Sysmon and Sysmon Configuration

| Source | URL | Used For |
|---|---|---|
| Sysmon ZIP package | `https://download.sysinternals.com/files/Sysmon.zip` | Downloading Sysmon for Windows Server 2025 in Part 5. |
| Olaf Hartong Sysmon Modular config | `https://raw.githubusercontent.com/olafhartong/sysmon-modular/master/sysmonconfig.xml` | Applying the Sysmon XML configuration in Part 5. |

### 2.6 SOCFortress and Packetbeat Sources

| Source | URL | Used For |
|---|---|---|
| SOCFortress Wazuh-Rules repository | `https://github.com/socfortress/Wazuh-Rules` | Optional advanced Wazuh rules in Part 4 and Packetbeat configuration source in Part 5. |
| SOCFortress Wazuh rules installer script | `https://raw.githubusercontent.com/socfortress/Wazuh-Rules/main/wazuh_socfortress_rules.sh` | Optional Wazuh rule installation workflow in Part 4. |
| SOCFortress Packetbeat YAML | `https://raw.githubusercontent.com/socfortress/Wazuh-Rules/main/Packetbeat/packetbeat.yml` | Packetbeat configuration downloaded by the Part 5 install script. |
| Packetbeat 7.16.3 RPM | `https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-7.16.3-x86_64.rpm` | Packetbeat install script path for yum-based systems in Part 5. |
| Packetbeat 7.16.3 DEB | `https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-7.16.3-amd64.deb` | Packetbeat install script path for Debian-based systems in Part 5. |

---

## 3. Lab Hosts and Internal Endpoints

| Component | Hostname | IP / URL | Notes |
|---|---|---|---|
| Wazuh Indexer | `wazuh-indexer` | `192.168.71.100` | OpenSearch-based Wazuh storage and search layer. REST API uses `9200/TCP`. |
| Graylog Server | `graylog-server` | `http://192.168.71.101:9000` | Graylog web interface and REST API in the lab. Receives Wazuh alerts on TCP `5555`. |
| Wazuh Manager | `wazuh-manager` | `192.168.71.102` | Receives agent events, writes `alerts.json`, and forwards alerts to Graylog through Fluent Bit. |
| Wazuh Dashboard | `wazuh-dashboard` | `https://192.168.71.103` | Wazuh web UI used for agent deployment, group editing, and management workflows. |
| Debian endpoint | `debian12-1` | `192.168.71.201` | Linux Wazuh agent and Packetbeat endpoint. |
| Windows endpoint | `windows-server-2025-1` | `192.168.71.202` | Windows Wazuh agent and Sysmon endpoint. |

---

## 4. Port Reference

| Port | Component / Flow | Purpose |
|---|---|---|
| `443/TCP` | Browser to Wazuh Dashboard | Wazuh Dashboard web interface. |
| `9000/TCP` | Browser to Graylog | Graylog web interface and REST API in this lab. |
| `9200/TCP` | Dashboard or Graylog to Wazuh Indexer | Wazuh Indexer REST API. |
| `9300-9400/TCP` | Wazuh Indexer cluster traffic | Indexer cluster communication. |
| `55000/TCP` | Wazuh Dashboard to Wazuh Manager API | Wazuh Server/Manager API connection. |
| `27017/TCP` | Graylog to MongoDB | MongoDB metadata database for Graylog. |
| `5555/TCP` | Fluent Bit to Graylog | Raw/Plaintext TCP input for Wazuh alerts. |
| `1515/TCP` | Wazuh agents to Wazuh Manager | Agent registration through `wazuh-authd`. |
| `1514/TCP` | Wazuh agents to Wazuh Manager | Agent event transport through `wazuh-remoted`. |

---

## 5. Important Files and Paths

### 5.1 Wazuh Indexer and Dashboard

| Path | Purpose |
|---|---|
| `/etc/wazuh-indexer/opensearch.yml` | Wazuh Indexer configuration, including certificate DN and compatibility settings. |
| `/etc/wazuh-indexer/certs/` | Indexer certificate storage. |
| `/etc/wazuh-indexer/jvm.options` | JVM heap configuration for the Indexer. |
| `/etc/wazuh-dashboard/opensearch_dashboards.yml` | Wazuh Dashboard listener, Indexer connection, and TLS certificate settings. |
| `/etc/wazuh-dashboard/certs/` | Dashboard certificate storage. |
| `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml` | Wazuh Dashboard configuration used to point the Dashboard to the Wazuh Manager/API. |

### 5.2 Graylog

| Path | Purpose |
|---|---|
| `/etc/graylog/server/server.conf` | Graylog server configuration, including backend host settings and password values. |
| `/etc/default/graylog-server` | Graylog Java options and runtime configuration. |
| `/var/log/graylog-server/server.log` | Graylog server logs used during backend troubleshooting. |

### 5.3 Wazuh Manager and Agents

| Path | Purpose |
|---|---|
| `/var/ossec/etc/ossec.conf` | Wazuh Manager configuration. Incorrect ownership on this file caused the Part 5 manager startup failure. |
| `/var/ossec/etc/authd.pass` | Agent registration password file used after registration password support was enabled. |
| `alerts.json` | Wazuh alert output consumed by Fluent Bit. |
| `/var/ossec/logs/archives/archives.json` | Wazuh archive output used to confirm Windows endpoint events on the manager. |

### 5.4 Fluent Bit, Sysmon, and Packetbeat

| Path | Purpose |
|---|---|
| `/etc/fluent-bit/fluent-bit.conf` | Fluent Bit configuration that forwards Wazuh alert output to Graylog. |
| `/var/log/td-agent-bit.log` | Fluent Bit log used to confirm worker startup. |
| `C:\Temp\SysmonInstall\Extracted\Sysmon64.exe` | Sysmon binary path used during Part 5 installation. |
| `C:\Temp\SysmonInstall\sysmonconfig.xml` | Downloaded Sysmon XML configuration. |
| `Microsoft-Windows-Sysmon/Operational` | Windows Event Channel collected by Wazuh after Sysmon installation. |
| `/etc/packetbeat/packetbeat.yml` | Packetbeat configuration downloaded from SOCFortress. |
| `/tmp/packetbeat/packetbeat` | Packetbeat JSON output collected by the Linux Wazuh agent. |

---

## 6. Key Commands Preserved Across the Guides

### 6.1 Service and Port Validation

| Command | Context |
|---|---|
| `ss -ltnpd` | Used to confirm listening services such as Graylog TCP input port `5555`. |
| `ss -ltnpd \| grep wazuh` | Used to confirm Wazuh Manager registration and event ports `1515` and `1514`. |
| `sudo systemctl status wazuh-indexer` | Wazuh Indexer service verification. |
| `sudo systemctl status wazuh-manager.service` | Wazuh Manager service verification after permission fixes. |
| `tail -f /var/log/graylog-server/server.log` | Graylog troubleshooting and backend connection review. |
| `tail -f /var/log/td-agent-bit.log` | Fluent Bit worker startup validation. |

### 6.2 Certificate and Backend Troubleshooting

| Command / Setting | Context |
|---|---|
| `openssl s_client -connect 192.168.71.100:9200 -showcerts` | Used to inspect the Wazuh Indexer certificate and SAN values in Part 3. |
| `compatibility.override_main_response_version: true` | Setting that caused Graylog 7 to detect the backend as Elasticsearch 7.10.2 instead of OpenSearch 2.x. |
| `# compatibility.override_main_response_version: true` | Corrected backend setting used after Part 3 troubleshooting. |
| `sudo bash -c 'chmod 400 /etc/wazuh-indexer/certs/*'` | Correct pattern for applying permissions when shell wildcards require elevated context. |
| `sudo bash -c 'chmod 400 /etc/wazuh-dashboard/certs/*'` | Same wildcard-permission lesson applied to Dashboard certificates. |

### 6.3 Endpoint Telemetry Validation

| Command | Context |
|---|---|
| `Get-Service Sysmon64` | Checks whether Sysmon is installed as a Windows service. |
| `wevtutil el \| findstr /i sysmon` | Checks whether the Sysmon Event Log channel exists. |
| `Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10` | Reviews recent Sysmon events. |
| `sudo tail -f /var/ossec/logs/archives/archives.json \| grep -Ei "002\|windows-server-2025-1\|Sysmon\|Microsoft-Windows-Sysmon"` | Confirms Windows endpoint events arriving at the Wazuh Manager. |
| `sudo tail -f /tmp/packetbeat/packetbeat` | Confirms live Packetbeat JSON output on the Debian endpoint. |

---

## 7. Graylog Routing Values From Part 6

| Item | Value |
|---|---|
| Extractor type | JSON extractor |
| Extractor condition | `Always try to extract` |
| List item separator | `,` |
| Key separator | `_` |
| Key/value separator | `:` |
| Index set name | `WAZUH ALERTS` |
| Index prefix | `wazuh-alerts-nimish` |
| Search pattern | `wazuh-alerts-nimish*` |
| Data Tiering minimum lifetime | 7 days |
| Data Tiering maximum lifetime | 14 days |
| Stream name | `Wazuh Alerts` |
| Stream target index set | `WAZUH ALERTS` |
| Remove matches from default stream | Enabled |
| Static field | `log_type=wazuh` |
| Stream rule | `log_type` match exactly `wazuh` |
| Verified routed index | `wazuh-alerts-nimish_0` |

---

## 8. Notes on Source Boundaries

This reference file is intentionally scoped to Parts 1-6. It does not include general vendor documentation pages, best-practice articles, or third-party learning resources unless they were explicitly present in the guides.

References that may be added later should be tied to the part of the lab where they are actually used.
