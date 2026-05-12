# SIEM HomeLab Learning Notes

These notes summarize what was learned while building Parts 1-6 of the SIEM HomeLab. They are intentionally limited to the content already documented in the deployment guides. The goal is not to create a generic SIEM theory page, but to capture the practical lessons that came out of this specific build.

---

## 1. Architecture Lessons

### 1.1 The Lab Is Not the Default Wazuh Data Path

The biggest architecture decision in this project is that Wazuh alerts are not sent directly from the Wazuh Manager to the Wazuh Indexer with the default Filebeat-style path.

The working flow after Part 6 is:

```text
Wazuh agents
  -> Wazuh Manager
  -> alerts.json
  -> Fluent Bit
  -> Graylog Raw/Plaintext TCP input on port 5555
  -> Graylog JSON extractor
  -> Wazuh Alerts stream
  -> WAZUH ALERTS index set
  -> wazuh-alerts-nimish_0 and future matching indices
```

This design makes Graylog the handling layer for ingestion, parsing, routing, buffering, and later enrichment. Wazuh still performs endpoint collection, ruleset evaluation, and alert generation, but Graylog controls how those alert records are parsed and stored.

That separation matters. It means future sources such as firewall logs or Office 365 logs can use their own inputs, static fields, streams, and index sets instead of being mixed into one default bucket.

### 1.2 Graylog Ingestion and Graylog Parsing Are Separate Problems

Part 4 proved that Wazuh alerts were reaching Graylog. Part 6 showed that this alone was not enough. The first Graylog messages arrived as raw blocks where the JSON payload was mostly trapped inside the `message` field.

The useful mental model is:

- Ingestion answers: did the event arrive?
- Parsing answers: can I search and dashboard the important fields?
- Routing answers: did the event land in the right stream and index set?

Part 6 completed that path for Wazuh alerts by adding a JSON extractor, a `log_type=wazuh` static field, a stream rule, and the dedicated `WAZUH ALERTS` index set.

### 1.3 Dedicated Index Sets Keep the Lab Manageable

Putting Wazuh alerts into `wazuh-alerts-nimish` is more than a naming preference. It keeps Wazuh fields, retention, and searches separate from other future sources. This helps avoid field-growth problems when different log families produce many unique key/value pairs.

The original plan was to rotate at around 10 GB with a legacy size-based rotation model. In Part 6, that plan was corrected because the Graylog interface being used favored Data Tiering. The final lab setting keeps Wazuh alerts for a minimum of 7 days and a maximum of 14 days.

For this HomeLab, that is enough. Recent alerts stay searchable for testing and troubleshooting, but the lab is not pretending to be a long-term archive.

---

## 2. Component Notes

### 2.1 Wazuh Indexer

The Wazuh Indexer is the OpenSearch-based storage and search layer. In this lab, it runs as a single node at `192.168.71.100`. The same VM acts as master, data, and ingest node.

That is acceptable for learning, but it is not production-resilient. A single-node Indexer has no automatic failover. The Part 1 notes explicitly call out that production should use multiple nodes, replica shards, SSD-backed storage, and tighter network segmentation.

The Indexer also introduced the first recurring lesson in the lab: certificate and permission details matter. Certificate files under `/etc/wazuh-indexer/certs/` must be owned and protected correctly, and wildcard commands such as `sudo chmod 400 /etc/wazuh-indexer/certs/*` can fail because the shell expands `*` before `sudo` applies.

The reliable pattern used in the lab was:

```bash
sudo bash -c 'chmod 400 /etc/wazuh-indexer/certs/*'
```

### 2.2 Wazuh Dashboard

The Dashboard runs separately at `192.168.71.103` and provides the web interface for Wazuh management, monitoring, and investigation. It connects to the Wazuh Indexer over HTTPS and later connects to the Wazuh Manager API.

Part 2 exposed a planning mistake: the `wazuh-dashboard` node was not included in the original certificate generation configuration from Part 1. Because the Wazuh certificate tool generates the set from `config.yml`, adding the Dashboard later required regenerating the certificate set and redeploying the updated certificates.

The practical lesson is simple: define the planned Wazuh stack components before running certificate generation. Even if a component has not been deployed yet, adding it to the certificate configuration up front avoids avoidable certificate churn later.

Part 2 also corrected a Common Name mismatch in `opensearch.yml`. The certificate CN and the `plugins.security.nodes_dn` / `plugins.security.authcz.admin_dn` values must line up. If they do not, the system may install correctly but fail later in less obvious ways.

### 2.3 Graylog Server

Graylog runs at `192.168.71.101` and is reachable in the lab at:

```text
http://192.168.71.101:9000
```

In this build, Graylog 7.0.5 is paired with Java 21 and MongoDB 8.0. MongoDB stores Graylog configuration and metadata. Graylog itself acts as the log handling layer: inputs, streams, pipelines, enrichment, routing, buffering, and index behavior.

The important troubleshooting lesson from Part 3 was that name resolution is not the same as certificate validation. Adding `wazuh-indexer` to `/etc/hosts` made the name resolve, but the Indexer certificate was valid for `IP Address:192.168.71.100`, not for the DNS name `wazuh-indexer`. Graylog therefore had to connect to the backend using the IP address:

```text
https://username:password@192.168.71.100:9200
```

Part 3 also showed that `compatibility.override_main_response_version: true` can break Graylog 7 backend detection in this architecture. With that setting enabled, Graylog detected the backend as Elasticsearch 7.10.2 instead of OpenSearch 2.x. Commenting it out allowed Graylog to detect the Wazuh Indexer correctly.

### 2.4 Wazuh Manager

The Wazuh Manager runs at `192.168.71.102`. It receives endpoint data, applies Wazuh rules, generates alerts, and writes matching alerts to `alerts.json`.

Part 4 intentionally avoided Filebeat and used Fluent Bit instead. Fluent Bit reads Wazuh alert output and forwards it to Graylog on TCP port `5555`.

The first Graylog input for Wazuh alerts was a Raw/Plaintext TCP input:

| Setting | Value |
|---|---|
| Title | `WAZUH EVENTS FLUENT BIT - TCP` |
| Bind address | `0.0.0.0` |
| Port | `5555` |

This is fine for a lab, but the Part 4 notes call out that TLS should be enabled for production so Wazuh alert data is not sent in cleartext.

Part 4 also hardened agent enrollment by enabling registration password support through `authd.pass`. Leaving registration open means any reachable agent can attempt to enroll. A password does not make the design perfect, but it makes onboarding more controlled.

### 2.5 Wazuh Agents

Part 5 added two monitored endpoints:

| Endpoint | Address | Role |
|---|---|---|
| `debian12-1` | `192.168.71.201` | Debian 12 endpoint with Wazuh agent and Packetbeat |
| `windows-server-2025-1` | `192.168.71.202` | Windows Server 2025 endpoint with Wazuh agent and Sysmon |

Agent enrollment and event transport use different manager-side ports:

| Port | Process | Purpose |
|---|---|---|
| `1515/TCP` | `wazuh-authd` | Agent registration and key exchange |
| `1514/TCP` | `wazuh-remoted` | Operational agent event traffic |

This split is easy to forget. If `1515` is open but `1514` is blocked, an agent may register but not send usable events. If `1515` is blocked, onboarding fails.

Agent status can also briefly show `never connected` in the UI even after registration. That does not automatically mean deployment failed. It can mean the agent service has not started, the heartbeat has not arrived, or the UI has not refreshed.

### 2.6 Sysmon and Packetbeat

The basic Wazuh agent is useful, but Part 5 extended telemetry with endpoint-specific tools:

- Sysmon on Windows for richer Windows activity telemetry.
- Packetbeat on Debian for network flow visibility.

Sysmon writes to:

```text
Applications and Services Logs > Microsoft > Windows > Sysmon > Operational
```

The Windows Wazuh group collects it with:

```xml
<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
</localfile>
```

Packetbeat writes JSON output to:

```text
/tmp/packetbeat/packetbeat
```

The Linux Wazuh group collects it with:

```xml
<localfile>
    <log_format>json</log_format>
    <location>/tmp/packetbeat/packetbeat</location>
</localfile>
```

The broader pattern is useful: a separate tool can generate telemetry locally, and the Wazuh agent can collect the output if the path and format are configured correctly.

---

## 3. Troubleshooting Lessons That Actually Mattered

### 3.1 Certificate SANs Beat Hostname Assumptions

The Graylog-to-Indexer issue in Part 3 is the cleanest example. `/etc/hosts` can make a hostname resolve, but it cannot make a TLS certificate valid for that hostname. The certificate Subject Alternative Name decides what names or IPs are valid.

When the cert only listed `IP Address:192.168.71.100`, Graylog had to use the IP address in `elasticsearch_hosts`.

### 3.2 Regenerate Certificates When the Node List Changes

Part 2 required certificate regeneration because the Dashboard was added after the original certificate set was created. The mistake was not catastrophic, but it created extra work across already-running systems.

The better workflow is to populate `config.yml` with all planned Wazuh nodes before running `wazuh-certs-tool.sh`.

### 3.3 Copying Commands Can Introduce Bad Characters

Part 4 preserved a failed Wazuh GPG import command where the dash before `import` was an en dash instead of two ASCII hyphens.

Bad:

```bash
–import
```

Good:

```bash
--import
```

This is a small detail, but it is exactly the kind of copy/paste issue that wastes time during infrastructure builds.

### 3.4 File Ownership Can Break Services Even When XML Looks Right

Part 5 captured a Wazuh Manager failure that looked like an `ossec.conf` XML read problem. The root cause was not really the content of the file. The file had been recreated with the wrong ownership: `root:root` instead of `root:wazuh`.

The fix was:

```bash
sudo chmod 640 /var/ossec/etc/ossec.conf
sudo chown root:wazuh /var/ossec/etc/ossec.conf
```

The lesson is to check ownership and permissions before assuming the service is complaining about syntax.

### 3.5 Sysmon Download Is Not Sysmon Installation

Part 5 showed that downloading Sysinternals or Sysmon files does not install the Sysmon service. The service must exist and run as `Sysmon64`.

The failed state was confusing because the Sysmon Event Log channel still appeared even though the service did not exist. That stale Event Log registration caused manifest installation failures until the lab cleaned up stale services, drivers, registry keys, leftover files, and the old EVTX file, then rebooted before reinstalling Sysmon.

The working sequence was:

- Download Sysmon.
- Download the Sysmon XML configuration.
- Clean stale Sysmon registration when installation fails.
- Reboot.
- Install with `Sysmon64.exe -accepteula -i`.
- Apply the XML configuration with `Sysmon64.exe -c`.
- Verify the service and event channel.

### 3.6 Dashboard Visibility Is Not the Same as Ingestion

In Part 5, Windows endpoint events were visible in the manager-side archive file even though the Dashboard was not showing them yet. That was a data view or index visibility issue, not proof that ingestion failed.

The command-line validation mattered:

```bash
sudo tail -f /var/ossec/logs/archives/archives.json | grep -Ei "002|windows-server-2025-1|Sysmon|Microsoft-Windows-Sysmon"
```

When debugging SIEM pipelines, check the lowest layer that can prove receipt before assuming the UI is authoritative.

---

## 4. Operational Guardrails

### 4.1 Keep Production Assumptions Separate From Lab Shortcuts

This lab intentionally uses shortcuts that are acceptable for learning:

- Single-node Wazuh Indexer.
- Single-node Graylog and MongoDB.
- Raw/Plaintext TCP input on port `5555`.
- Lab-only credentials in preserved command history.
- Short retention for Wazuh alert data.

The notes repeatedly mark where production would need more rigor: TLS-enabled Graylog inputs, multiple nodes for availability, restricted firewall rules, careful credential handling, and tested backup or recovery procedures.

### 4.2 Validate Each Layer Before Moving Up

The most reliable validation pattern across Parts 1-6 was layer-by-layer:

- Is the service running?
- Is the port listening?
- Can the next component connect?
- Is the event visible in a local file or service log?
- Is the event parsed into fields?
- Is the event routed into the right stream or index?

Skipping directly to the web UI can hide where the real failure is.

### 4.3 Keep Third-Party Rules and Configs Controlled

The SOCFortress rules and Packetbeat configuration are useful, but Part 4 warns not to run third-party Wazuh rule scripts blindly. Duplicate rule IDs or conflicting rules can stop the Wazuh Manager from starting.

The safe pattern is to inspect third-party scripts before running them, apply them intentionally, and check Wazuh Manager logs immediately afterward.

### 4.4 Static Routing Fields Are Simple and Reliable

Part 6 used `log_type=wazuh` as a static field on the Fluent Bit input. This is not complex, but it is dependable. It gives Graylog a routing marker that does not depend on a payload field that might vary between alert types.

For future sources, the same idea can be reused:

| Source | Possible Static Field |
|---|---|
| Wazuh alerts | `log_type=wazuh` |
| Firewall logs | `log_type=firewall` |
| Office 365 logs | `log_type=o365` |

That keeps stream rules readable and makes it easier to route each source to the right index set.

---

## 5. Current State After Part 6

By the end of Part 6, the lab has:

- A Wazuh Indexer at `192.168.71.100`.
- A Wazuh Dashboard at `192.168.71.103`.
- A Graylog Server at `192.168.71.101`.
- A Wazuh Manager at `192.168.71.102`.
- A Debian endpoint at `192.168.71.201`.
- A Windows Server 2025 endpoint at `192.168.71.202`.
- Wazuh agents enrolled into Linux and Windows groups.
- Sysmon collected through the Windows group.
- Packetbeat JSON collected through the Linux group.
- Wazuh alerts forwarded to Graylog through Fluent Bit.
- Wazuh alert JSON parsed into fields.
- Wazuh alerts routed by `log_type=wazuh`.
- Routed Wazuh alerts stored in the `WAZUH ALERTS` index set with the `wazuh-alerts-nimish` prefix.

The next natural work is dashboarding, visualization, alert review, and threat investigation workflows on top of the parsed and routed data.
