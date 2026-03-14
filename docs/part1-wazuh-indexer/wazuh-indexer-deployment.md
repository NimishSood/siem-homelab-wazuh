# SIEM HomeLab — Part 1: Wazuh Indexer Deployment

| Field | Details |
|---|---|
| **Lab Type** | Cybersecurity / SIEM Homelab |
| **Platform** | VMware Workstation • Ubuntu Server 24.04.4 LTS |
| **Date** | March 2026 |
| **Component** | Wazuh Indexer (OpenSearch) — Single-Node Deployment |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Lab Environment](#2-lab-environment)
3. [Key Concepts — Data Organisation](#3-key-concepts--data-organisation)
4. [Installation & Setup](#4-installation--setup)
5. [Cluster Initialisation](#5-cluster-initialisation)
6. [Verification](#6-verification)
7. [Observations & Notes](#7-observations--notes)
8. [Conclusion](#8-conclusion)

---

## 1. Introduction

### 1.1 Lab Overview

This document covers Part 1 of a multi-part SIEM (Security Information and Event Management) HomeLab series. The objective is to build a production-like security monitoring platform using open-source tooling running on VMware Workstation. Part 1 focuses exclusively on deploying the **Wazuh Indexer** — the data storage and search engine at the heart of the Wazuh stack.

The lab is designed to reflect real-world security operations, including network segmentation recommendations, TLS certificate management, and production-grade tuning (memory locking and JVM heap configuration).

### 1.2 What Is the Wazuh Indexer?

The Wazuh Indexer is a highly scalable, full-text search and analytics engine based on **OpenSearch**. It is the data persistence layer of the Wazuh SIEM, responsible for receiving, storing, and making security event data queryable in near real-time.

In a production cluster, nodes can be assigned one or more of the following roles:

- **Master Node** — Manages cluster-wide settings and topology. At least one node holds this role; in a 3-node cluster all may be eligible, but only one is active master at runtime. If the active master goes offline, a remaining eligible node is automatically promoted.
- **Data Node** — Stores shards and handles indexing, searching, and aggregation. I/O and storage intensive — benefits significantly from SSD storage.
- **Ingest Node** — Pre-processes documents through transformation pipelines before they are written to an index.

> This HomeLab uses a **single-node configuration** in which one VM performs all three roles simultaneously. This is appropriate for learning and development purposes.

### 1.3 Role in the SIEM Architecture

The Wazuh Indexer is the central data store for the full SIEM stack planned for this HomeLab series:

| Component | Role | Status |
|---|---|---|
| **Wazuh Indexer (OpenSearch)** | Data storage and search layer | *This lab* |
| **Wazuh Dashboard** | Web-based visualisation and monitoring UI | Part 2 |
| **Graylog** | Log collection and processing (replaces the Wazuh Filebeat pipeline) | Part 3 |
| **Grafana** | Additional dashboarding and alerting | Future |
| **Cassandra / MySQL** | Supporting databases for Graylog | Future |

> **Note:** Because Graylog is used for log ingestion instead of the default Wazuh Filebeat agent, the **Wazuh Server component is not deployed** in this architecture.

---

## 2. Lab Environment

### 2.1 Host Platform

| Setting | Value |
|---|---|
| **Hypervisor** | VMware Workstation |
| **Operating System** | Ubuntu Server 24.04.4 LTS (Noble Numbat) |
| **Hostname** | `wazuh-indexer` |
| **ISO Source** | https://ubuntu.com/download/server |

### 2.2 Virtual Machine Specifications

| Resource | Specification |
|---|---|
| **CPU** | 1 Processor, 4 Cores |
| **RAM** | 8 GB |
| **Disk** | 100 GB SSD (single virtual disk file) |
| **Network** | VMware NAT — `192.168.71.0/24` |
| **Static IP** | `192.168.71.100` |
| **Default Gateway** | `192.168.71.2` *(192.168.71.1 is the VMware virtual interface)* |

### 2.3 Network Port Requirements

The following TCP ports must be accessible between Wazuh components. In production, all SIEM services should be isolated in a dedicated VLAN with only these ports whitelisted.

| Component | Port / Protocol | Purpose |
|---|---|---|
| Wazuh Indexer | `9200` TCP | RESTful API |
| Wazuh Indexer | `9300–9400` TCP | Cluster Communication |
| Wazuh Dashboard | `443` TCP | Web User Interface |

---

## 3. Key Concepts — Data Organisation

### 3.1 Indexes

- Each index is a uniquely named data store. In a Wazuh deployment, a new index is created every day to hold that day's events.
- Storing data for extended periods in a single index leads to severely degraded retrieval performance.
- Wazuh (via Graylog) allows configuring how many shards compose each index.

### 3.2 Shards and Replicas

- Each index is divided into one or more **shards** distributed across data nodes.
- Without replicas, data stored on a node that goes offline is inaccessible until that node recovers.
- **Replica shards** are copies of primary shards maintained on different nodes, providing fault tolerance at the cost of approximately double the disk space.

### 3.3 Storage Sizing

- Storage requirements depend on endpoint count, log verbosity, and data retention policy.
- Wazuh provides an official storage estimation table as a starting point, but real-world usage will differ. Always over-provision.
- Use **SSD storage** in all production environments.
- Isolate indexer storage in a dedicated VLAN and whitelist only the required inter-component connections.

---

## 4. Installation & Setup

### Step 1 — Create the Virtual Machine

Create a new VM in VMware Workstation using the **Ubuntu Server 24.04.4 LTS** ISO with the following specifications:

- CPU: 1 Processor, 4 Cores
- RAM: 8 GB
- Disk: 100 GB SSD — *Store virtual disk as a single file*

![VMware virtual machine configuration settings](./screenshots/figure1-vmware-vm-config.png)

---

### Step 2 — Install Ubuntu Server

Power on the VM and proceed through the Ubuntu Server installer. Accept defaults unless otherwise noted. After installation completes, reboot into the new OS.

![Ubuntu Server 24.04.4 LTS installation screen](./screenshots/figure2-ubuntu-install.png)

---

### Step 3 — Configure Static IP and Hostname

Assign a static IP to the Wazuh Indexer VM, then register its hostname in `/etc/hosts`:

```bash
# /etc/hosts entry
192.168.71.100 wazuh-indexer
```

> **Note:** Also pre-register `192.168.71.101` for Graylog and the Grafana Dashboard IP in `/etc/hosts`. These entries are referenced during certificate generation in the next step.

---

### Step 4 — Generate SSL/TLS Certificates

Wazuh encrypts all inter-component communications using TLS. The `wazuh-certs-tool.sh` script generates certificates for all components in one pass.

#### 4a — Download the certificate tool and configuration template

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```

#### 4b — Edit config.yml with node hostnames and IP addresses

Replace placeholder node names and IPs with actual values for all Wazuh components. The Graylog node (`192.168.71.101`) is entered in the `server` field, as it replaces the Wazuh Server in this architecture.

![config.yml edited with node names and IP addresses](./screenshots/figure3-config-yml.png)

#### 4c — Generate the certificates

```bash
bash ./wazuh-certs-tool.sh -A
```

![wazuh-certs-tool.sh certificate generation output](./screenshots/figure4-cert-generation.png)

---

### Step 5 — Update the System

Before installing packages, bring the system fully up to date:

```bash
sudo apt-get update
sudo apt upgrade
```

---

### Step 6 — Install Dependencies and Add the Wazuh Repository

#### 6a — Install required system packages

```bash
apt-get install gnupg apt-transport-https
```

#### 6b — Import the Wazuh GPG signing key

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | gpg --no-default-keyring \
        --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
        --import \
  && chmod 644 /usr/share/keyrings/wazuh.gpg
```

#### 6c — Add the Wazuh apt repository

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee -a /etc/apt/sources.list.d/wazuh.list
```

#### 6d — Refresh package metadata

```bash
apt-get update
```

---

### Step 7 — Install the Wazuh Indexer Package

```bash
apt-get -y install wazuh-indexer
```

![Wazuh Indexer apt installation command](./screenshots/figure5-apt-install.png)

> **Access note:** The `/etc/wazuh-indexer/` directory is root-restricted. Use `sudo`-prefixed commands to access it without opening a permanent root session:

```bash
sudo ls /etc/wazuh-indexer
sudo nano /etc/wazuh-indexer/opensearch.yml
sudo cat /etc/wazuh-indexer/opensearch.yml
```

---

### Step 8 — Configure opensearch.yml

Open the primary Wazuh Indexer configuration file:

```bash
sudo nano /etc/wazuh-indexer/opensearch.yml
```

Apply the following changes:

| Setting | Value |
|---|---|
| `network.host` | `192.168.71.100` |
| `node.name` | `wazuh-indexer` |
| `cluster.initial_master_nodes` | `["wazuh-indexer"]` |
| `cluster.name` | *(Update as desired — optional)* |
| `discovery.seed_hosts` | Add only the current node (single-node cluster) |
| `node.max_local_storage_nodes` | Change from `3` to `1` |

![opensearch.yml fully configured for single-node deployment](./screenshots/figure6-opensearch-yml.png)

---

### Step 9 — Package Certificates into a Tar Archive

```bash
sudo tar -cvf ./wazuh-certificates.tar -C ./wazuh-certificates/ .
```

![Compressing generated certificates into tar archive](./screenshots/figure7-tar-certificates.png)

---

### Step 10 — Deploy Certificates to the Indexer

#### 10a — Set the node name environment variable

```bash
NODE_NAME=wazuh-indexer
```

#### 10b — Create the certs directory and extract required certificate files

```bash
sudo mkdir /etc/wazuh-indexer/certs

sudo tar -xf ./wazuh-certificates.tar \
  -C /etc/wazuh-indexer/certs/ \
  ./$NODE_NAME.pem \
  ./$NODE_NAME-key.pem \
  ./admin.pem \
  ./admin-key.pem \
  ./root-ca.pem
```

![Certificate files extracted into /etc/wazuh-indexer/certs/](./screenshots/figure8-certs-extracted.png)

#### 10c — Update certificate file paths in opensearch.yml

Re-open `opensearch.yml` and update the certificate filename references in all 4 locations to match the extracted files. Refer to [Figure 6](#step-8--configure-opensearchyml) for the final configuration.

#### 10d — Set correct file permissions and ownership

```bash
sudo bash -c 'chmod 400 /etc/wazuh-indexer/certs/*'
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

> **Important:** The shell expands the wildcard (`*`) before `sudo` gains root privileges. Wrapping the command in `bash -c` ensures the elevated context applies to the entire glob expression.

---

### Step 11 — Configure Memory Locking

Preventing the JVM heap from being swapped to disk is critical for search engine performance. This requires changes in **three files**.

#### 11a — Enable memory locking in opensearch.yml

Add or uncomment the following line in `/etc/wazuh-indexer/opensearch.yml`:

```yaml
bootstrap.memory_lock: true
```

![bootstrap.memory_lock enabled in opensearch.yml](./screenshots/figure9-memory-lock.png)

#### 11b — Set the systemd memory lock limit

```bash
sudo nano /usr/lib/systemd/system/wazuh-indexer.service
```

Add the following under the `[Service]` section:

```ini
[Service]
LimitMEMLOCK=infinity
```

![LimitMEMLOCK=infinity configured in the systemd service unit](./screenshots/figure10-systemd-memlimit.png)

#### 11c — Configure JVM heap size in jvm.options

```bash
sudo nano /etc/wazuh-indexer/jvm.options
```

Change the heap allocation from the default 1 GB to **4 GB** (50% of available RAM):

```
-Xms4g
-Xmx4g
```

![JVM heap size set to 4 GB in jvm.options](./screenshots/figure11-jvm-heap.png)

> **Sizing rule:** Allocate **50% of system RAM** to the JVM heap. On an 8 GB machine, this is 4 GB. Do not exceed **31 GB** due to JVM compressed ordinary object pointer behaviour.

---

### Step 12 — Enable and Start the Wazuh Indexer Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
sudo systemctl start wazuh-indexer
```

![Wazuh Indexer systemd service enabled and started](./screenshots/figure12-systemctl-start.png)

---

## 5. Cluster Initialisation

### Step 13 — Run the Security Admin Initialisation Script

The final step loads the TLS certificate configuration into the OpenSearch security plugin and initialises the cluster. This only needs to be run **once — on any single node** — regardless of cluster size.

```bash
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

![indexer-security-init.sh running and completing successfully](./screenshots/figure13-security-init.png)

---

## 6. Verification

### 6.1 Verify Cluster Initialisation via Log

Read the Wazuh cluster log to confirm successful initialisation. The expected output includes the message: **`Node has been initialized`**.

```bash
sudo cat /var/log/wazuh-indexer/wazuh-cluster.log
```

![Cluster log confirming "Node has been initialized"](./screenshots/figure14-cluster-log.png)

### 6.2 Verify Service Status

Confirm the Wazuh Indexer systemd service is active and running without errors:

```bash
sudo systemctl status wazuh-indexer
```

![Wazuh Indexer service active and healthy](./screenshots/figure15-service-status.png)

---

## 7. Observations & Notes

### 7.1 Certificate Permissions — Wildcard sudo Gotcha

Standard usage of `sudo chmod 400 /etc/wazuh-indexer/certs/*` fails because the shell expands the wildcard **before** `sudo` elevates privileges. The correct workaround wraps the command in `bash -c`:

```bash
sudo bash -c 'chmod 400 /etc/wazuh-indexer/certs/*'
```

### 7.2 Root-Restricted Configuration Directory

The `/etc/wazuh-indexer/` directory is intentionally root-protected to prevent accidental modification. Always use `sudo`-prefixed individual commands rather than switching to a persistent root shell when editing files in this directory.

### 7.3 Single-Node vs Production Cluster

In this HomeLab, one VM acts as master, data, and ingest node simultaneously. In production, a minimum of **3 nodes** is recommended for high availability. With multiple eligible master nodes, if the active master goes offline the cluster automatically elects a replacement. A single-node cluster has no such resilience.

### 7.4 Graylog Replaces the Wazuh Server Component

This architecture uses **Graylog** for log collection and forwarding instead of the default Wazuh Filebeat pipeline. As a result the Wazuh Server is not deployed. The `server` field in `config.yml` is populated with the Graylog node address (`192.168.71.101`) as a placeholder to generate the correct certificates for future use.

### 7.5 Cluster Initialisation Is a One-Time Operation

The `indexer-security-init.sh` script only needs to be executed **once per cluster deployment**. It does not need to be re-run after reboots, configuration changes, or when adding new nodes to the cluster at a later stage.

---

## 8. Conclusion

Part 1 of the SIEM HomeLab series successfully deployed a single-node Wazuh Indexer on Ubuntu Server 24.04.4 LTS running in VMware Workstation, completing the full workflow from VM creation through to verified cluster operation.

**Achievements in this lab:**

- Provisioned an Ubuntu Server VM with specifications appropriate for a single-node OpenSearch deployment
- Generated TLS certificates for all planned Wazuh stack components using the official `wazuh-certs-tool.sh` script
- Installed and configured the Wazuh Indexer (OpenSearch) with correct network binding, cluster identity, and certificate references
- Applied production-grade tuning: memory locking (`bootstrap.memory_lock`), systemd `LimitMEMLOCK`, and JVM heap allocation (4 GB)
- Enabled and started the Wazuh Indexer systemd service
- Completed cluster initialisation via the security admin script and verified successful operation through log inspection

---

> **Next Steps:** Part 2 will deploy the **Wazuh Dashboard** as the web-based visualisation and monitoring interface. The following step, Part 3, will deploy the **Graylog server** as the log collection layer and configure it to forward security events into the Wazuh Indexer.
