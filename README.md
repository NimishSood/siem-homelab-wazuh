# SIEM HomeLab - Wazuh Stack

This repository documents a hands-on SIEM homelab built around the Wazuh stack on Ubuntu Server VMs in VMware Workstation. The project is being developed as a staged build series, with each major component documented in its own deployment guide.

## Current Completion

Three parts of the lab are currently documented and complete:

| Part | Component | Status | Notes |
|---|---|---|---|
| 1 | Wazuh Indexer | Complete | Single-node Indexer deployed, secured with TLS, tuned, and verified |
| 2 | Wazuh Dashboard | Complete | Dashboard deployed on a separate VM and reachable over HTTPS |
| 3 | Graylog Server | Complete | Graylog deployed with MongoDB 8.0, Java 21, TLS truststore configuration, and verified web access |
| 4 | Wazuh Manager | Planned | Will receive agent events and provide the `alerts.json` source for Graylog ingestion |
| Future | Grafana / supporting services | Planned | Additional dashboards, alerting, and supporting data services |

At the current stage, the repository documents a working Wazuh Indexer, Wazuh Dashboard, and Graylog Server deployment. Graylog is reachable at `http://192.168.71.101:9000` and has been prepared to connect to the Wazuh Indexer, but later parts are still needed to complete the end-to-end event pipeline, deploy the Wazuh Manager, configure forwarding into Graylog, and replace the temporary backend/API placeholders described in the Part 2 notes.

## What Is In This Repo

- [Part 1 Indexer guide](docs/part1-wazuh-indexer/wazuh-indexer-deployment.md) covers VM creation, TLS certificate generation, Wazuh Indexer installation, OpenSearch tuning, cluster initialization, and verification.
- [Part 2 Dashboard guide](docs/part2-wazuh-dashboard/wazuh-dashboard-deployment.md) covers the dashboard VM, certificate regeneration, dashboard installation, configuration, service startup, and browser verification.
- [Part 3 Graylog Server guide](docs/part3-graylog-server/graylog-server-deployment.md) covers the Graylog VM, MongoDB 8.0, Graylog 7.0, Java 21, Wazuh Indexer TLS truststore setup, backend user creation, troubleshooting, and browser verification.
- `architecture/` contains supporting diagrams for the homelab design.
- `configs/` contains configuration snapshots for the stack, including `opensearch.yml`, `jvm.options`, and `graylog.conf`.
- `resources/` exists for notes and references, but both markdown files are currently empty placeholders.
- `scripts/` contains placeholder directories for future setup and attack-simulation automation.

## Lab Environment

The documented environment currently consists of:

| Component | Hostname | IP | OS | VM Sizing |
|---|---|---|---|---|
| Wazuh Indexer | `wazuh-indexer` | `192.168.71.100` | Ubuntu Server 24.04.4 LTS | 4 vCPU, 8 GB RAM, 100 GB disk |
| Wazuh Dashboard | `wazuh-dashboard` | `192.168.71.103` | Ubuntu Server 24.04.4 LTS | 2 vCPU, 2 GB RAM, 40 GB disk |
| Graylog Server | `graylog-server` | `192.168.71.101` | Ubuntu Server 24.04.4 LTS | 4 vCPU, 6 GB RAM, 80 GB disk |

Shared lab characteristics:

- Hypervisor: VMware Workstation
- Network: VMware NAT on `192.168.71.0/24`
- Deployment model: single-node Wazuh Indexer with separate Dashboard and Graylog Server VMs

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
│   └── part3-graylog-server/
│       ├── screenshots/
│       └── graylog-server-deployment.md
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

1. Deploying the Wazuh Manager as the event collection and analysis component.
2. Configuring Wazuh agents and the `alerts.json` output that will feed Graylog.
3. Creating Graylog inputs, streams, pipelines, and forwarding behavior for processed security events.
4. Replacing the temporary Dashboard backend/API placeholders so the expected "no server connected" state is resolved.
5. Expanding the repo with operational notes, references, and automation as the lab matures.

## Purpose

This project is a learning-focused homelab for understanding SIEM architecture, component integration, TLS-secured service deployment, and the operational tradeoffs involved in building a monitoring stack from open-source tooling.

## Note

This is a lab environment for education and experimentation, not a production deployment.
