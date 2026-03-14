# SIEM HomeLab - Wazuh Stack

This repository documents a hands-on SIEM homelab built around the Wazuh stack on Ubuntu Server VMs in VMware Workstation. The project is being developed as a staged build series, with each major component documented in its own deployment guide.

## Current Completion

Two parts of the lab are currently documented and complete:

| Part | Component | Status | Notes |
|---|---|---|---|
| 1 | Wazuh Indexer | Complete | Single-node Indexer deployed, secured with TLS, tuned, and verified |
| 2 | Wazuh Dashboard | Complete | Dashboard deployed on a separate VM and reachable over HTTPS |
| 3 | Graylog | Planned | Will handle log collection and forwarding into the Indexer |
| Future | Grafana / supporting services | Planned | Additional dashboards, alerting, and supporting data services |

At the current stage, the repository documents a working Indexer plus Dashboard deployment. The Dashboard is reachable, but later parts are still needed to complete the end-to-end event pipeline and replace the temporary backend/API placeholders described in the Part 2 notes.

## What Is In This Repo

- [Part 1 Indexer guide](C:\Users\Nimish\Desktop\siem-homelab-wazuh\siem-homelab-wazuh\docs\part1-wazuh-indexer\wazuh-indexer-deployment.md) covers VM creation, TLS certificate generation, Wazuh Indexer installation, OpenSearch tuning, cluster initialization, and verification.
- [Part 2 Dashboard guide](C:\Users\Nimish\Desktop\siem-homelab-wazuh\siem-homelab-wazuh\docs\part2-wazuh-dashboard\wazuh-dashboard-deployment.md) covers the dashboard VM, certificate regeneration, dashboard installation, configuration, service startup, and browser verification.
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

Shared lab characteristics:

- Hypervisor: VMware Workstation
- Network: VMware NAT on `192.168.71.0/24`
- Deployment model: single-node Wazuh Indexer with a separate Dashboard VM

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
│   └── part2-wazuh-dashboard/
│       ├── screenshots/
│       └── wazuh-dashboard-deployment.md
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

1. Deploying Graylog as the collection and processing layer.
2. Connecting Graylog to the Wazuh Indexer to begin building the event ingestion pipeline.
3. Replacing the temporary Dashboard backend/API placeholders so the expected "no server connected" state is resolved.
4. Expanding the repo with operational notes, references, and automation as the lab matures.

## Purpose

This project is a learning-focused homelab for understanding SIEM architecture, component integration, TLS-secured service deployment, and the operational tradeoffs involved in building a monitoring stack from open-source tooling.

## Note

This is a lab environment for education and experimentation, not a production deployment.
