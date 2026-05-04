# SIEM HomeLab - Wazuh Stack

This repository documents a hands-on SIEM homelab built around the Wazuh stack on VMware Workstation VMs and monitored endpoint hosts. The project is being developed as a staged build series, with each major component documented in its own deployment guide.

## Current Completion

Five parts of the lab are currently documented and complete:

| Part | Component | Status | Notes |
|---|---|---|---|
| 1 | Wazuh Indexer | Complete | Single-node Indexer deployed, secured with TLS, tuned, and verified |
| 2 | Wazuh Dashboard | Complete | Dashboard deployed on a separate VM and reachable over HTTPS |
| 3 | Graylog Server | Complete | Graylog deployed with MongoDB 8.0, Java 21, TLS truststore configuration, and verified web access |
| 4 | Wazuh Manager | Complete | Manager deployed, connected to Graylog through Fluent Bit, and connected to the Wazuh Dashboard |
| 5 | Wazuh Agents | Complete | Debian 12 and Windows Server 2025 agents deployed, with Sysmon and Packetbeat telemetry configured |
| Future | Grafana / parsing / supporting services | Planned | Graylog parsing pipelines, additional dashboards, alerting, and supporting data services |

At the current stage, the repository documents a working Wazuh Indexer, Wazuh Dashboard, Graylog Server, Wazuh Manager, and initial endpoint agent deployment. Graylog is reachable at `http://192.168.71.101:9000`, and the Wazuh Manager forwards raw Wazuh alert output to Graylog through Fluent Bit on TCP port `5555`. Debian 12 and Windows Server 2025 endpoints are onboarded to the manager, with Packetbeat extending Linux network visibility and Sysmon extending Windows endpoint telemetry. Later parts are still needed to parse raw messages into structured Graylog fields, complete downstream routing, add Grafana, and replace any remaining placeholder assets or configuration snapshots.

## What Is In This Repo

- [Part 1 Indexer guide](docs/part1-wazuh-indexer/wazuh-indexer-deployment.md) covers VM creation, TLS certificate generation, Wazuh Indexer installation, OpenSearch tuning, cluster initialization, and verification.
- [Part 2 Dashboard guide](docs/part2-wazuh-dashboard/wazuh-dashboard-deployment.md) covers the dashboard VM, certificate regeneration, dashboard installation, configuration, service startup, and browser verification.
- [Part 3 Graylog Server guide](docs/part3-graylog-server/graylog-server-deployment.md) covers the Graylog VM, MongoDB 8.0, Graylog 7.0, Java 21, Wazuh Indexer TLS truststore setup, backend user creation, troubleshooting, and browser verification.
- [Part 4 Wazuh Manager guide](docs/part4-wazuh-manager/wazuh-manager-deployment.md) covers Wazuh Manager installation, Graylog Raw/Plaintext TCP input setup, Fluent Bit forwarding, dashboard manager connection, hardening, vulnerability detection, agent groups, and optional SOCFortress rules.
- [Part 5 Wazuh Agents guide](part5-wazuh-agents/wazuh-agents-deployment.md) covers Debian 12 and Windows Server 2025 agent deployment, Sysmon installation and troubleshooting, Windows group Sysmon collection, Packetbeat installation, and Linux group Packetbeat collection.
- `architecture/` contains supporting diagrams for the homelab design.
- `configs/` contains configuration snapshots for the stack, including `opensearch.yml`, `jvm.options`, and `graylog.conf`.
- `resources/` contains learning notes and references for the lab.
- `scripts/` contains placeholder directories for future setup and attack-simulation automation.

> **Placeholder note:** Some referenced repository items are intentionally incomplete while the series is still being built. Diagrams such as `architecture/siem-architecture-diagram.png`, configuration snapshots under `configs/`, future pipeline files, and automation scripts may be placeholders until the full SIEM HomeLab series is finished.

## Lab Environment

The documented environment currently consists of:

| Component | Hostname | IP | OS | VM Sizing |
|---|---|---|---|---|
| Wazuh Indexer | `wazuh-indexer` | `192.168.71.100` | Ubuntu Server 24.04.4 LTS | 4 vCPU, 8 GB RAM, 100 GB disk |
| Wazuh Dashboard | `wazuh-dashboard` | `192.168.71.103` | Ubuntu Server 24.04.4 LTS | 2 vCPU, 2 GB RAM, 40 GB disk |
| Graylog Server | `graylog-server` | `192.168.71.101` | Ubuntu Server 24.04.4 LTS | 4 vCPU, 6 GB RAM, 80 GB disk |
| Wazuh Manager | `wazuh-manager` | `192.168.71.102` | Ubuntu Server 24.04 LTS | 4 vCPU, 6 GB RAM, 80 GB disk |
| Linux Endpoint | `debian12-1` | `192.168.71.201` | Debian 12 | Endpoint VM with Wazuh agent and Packetbeat |
| Windows Endpoint | `windows-server-2025-1` | `192.168.71.202` | Windows Server 2025 Standard | 4 vCPU, 8 GB RAM, 60 GB disk |

Shared lab characteristics:

- Hypervisor: VMware Workstation
- Network: VMware NAT on `192.168.71.0/24`
- Deployment model: single-node Wazuh Indexer with separate Dashboard, Graylog Server, Wazuh Manager, and monitored endpoint VMs

## Repository Structure

```text
siem-homelab-wazuh/
├── README.md
├── architecture/
│   ├── component-flow.png
│   ├── network-topology.png
│   └── siem-architecture-diagram.png
├── configs/
│   ├── graylog.conf
│   ├── jvm.options
│   ├── opensearch.yml
│   └── pipelines/
├── docs/
│   ├── part1-wazuh-indexer/
│   │   ├── screenshots/
│   │   └── wazuh-indexer-deployment.md
│   ├── part2-wazuh-dashboard/
│   │   ├── screenshots/
│   │   └── wazuh-dashboard-deployment.md
│   ├── part3-graylog-server/
│   │   ├── screenshots/
│   │   └── graylog-server-deployment.md
│   ├── part4-wazuh-manager/
│   │   ├── screenshots/
│   │   └── wazuh-manager-deployment.md
│   └── part5-wazuh-agents/
│       └── screenshots/
├── part5-wazuh-agents/
│   └── wazuh-agents-deployment.md
├── resources/
│   ├── learning-notes.md
│   └── references.md
├── screenshots/
└── scripts/
    ├── attack-simulation/
    └── setup/
```

## Next Steps

The next documented phase should focus on:

1. Parsing raw Wazuh, Sysmon, and Packetbeat messages in Graylog into searchable fields.
2. Creating Graylog streams, pipelines, and routing behavior for processed security events.
3. Validating downstream forwarding paths for processed endpoint telemetry.
4. Adding Grafana and any supporting services needed for visualization and alerting.
5. Replacing placeholder diagrams, configuration snapshots, pipeline files, and automation with finalized artifacts when the entire series is complete.
6. Expanding the repo with operational notes, references, and automation as the lab matures.

## Purpose

This project is a learning-focused homelab for understanding SIEM architecture, component integration, TLS-secured service deployment, and the operational tradeoffs involved in building a monitoring stack from open-source tooling.

## Note

This is a lab environment for education and experimentation, not a production deployment.
