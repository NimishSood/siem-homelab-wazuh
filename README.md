# SIEM Homelab --- Wazuh Stack

This repository documents my **SIEM homelab project**, where I am
learning how security monitoring systems are deployed and configured
using open‑source tools.

The lab is being built step‑by‑step and documented as a series to better
understand how **security event pipelines, indexing systems, and
monitoring infrastructure** operate.

------------------------------------------------------------------------

## Current Status

**Part 1 Completed**

Part 1 focuses on deploying the **Wazuh Indexer (OpenSearch)**, which
acts as the storage and search engine for security events within the
SIEM stack.

Topics covered in Part 1 include:

-   Creating an Ubuntu Server virtual machine
-   Installing the Wazuh Indexer
-   Generating TLS certificates for SIEM components
-   Configuring `opensearch.yml`
-   Applying JVM heap and memory locking configuration
-   Initializing the security cluster
-   Verifying service health and cluster initialization

------------------------------------------------------------------------

## Technologies Used

-   **Wazuh Indexer (OpenSearch)** --- Security event storage and search
-   **Ubuntu Server 24.04 LTS**
-   **VMware Workstation**

------------------------------------------------------------------------

## Lab Environment

  Component    Specification
  ------------ -------------------------
  Hypervisor   VMware Workstation
  OS           Ubuntu Server 24.04 LTS
  CPU          4 cores
  RAM          8 GB
  Storage      100 GB
  Network      VMware NAT

------------------------------------------------------------------------

## Repository Structure

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
    │   └── part1-wazuh-indexer/
    │       ├── screenshots/
    │       └── wazuh-indexer-deployment.md
    ├── resources/
    │   ├── learning-notes.md
    │   └── references.md
    ├── screenshots/
    └── scripts/
        ├── attack-simulation/
        └── setup/

Additional documentation will be added as the homelab progresses.

------------------------------------------------------------------------

## Purpose of This Project

This homelab is intended as a learning exercise to:

-   Understand SIEM architecture
-   Gain hands‑on experience with security infrastructure
-   Practice documenting technical deployments

------------------------------------------------------------------------

## Notes

This project is **not a production deployment** and is intended for
**educational purposes only**.
