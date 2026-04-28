# SIEM HomeLab — Part 3: Graylog Server Deployment

| Field | Details |
|---|---|
| **Lab Type** | Cybersecurity / SIEM Homelab |
| **Project** | SIEM HomeLab Project |
| **Platform** | VMware Workstation • Ubuntu Server 24.04.4 LTS |
| **Date** | April 2026 |
| **Author** | Nimish Sood |
| **Series** | SIEM HomeLab — Wazuh + Graylog + Grafana |
| **Component** | Graylog Server — Single-Node Deployment |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Lab Environment](#2-lab-environment)
3. [Key Concepts](#3-key-concepts)
4. [Installation & Configuration](#4-installation--configuration)
5. [Troubleshooting and Corrections](#5-troubleshooting-and-corrections)
6. [Verification](#6-verification)
7. [Observations & Notes](#7-observations--notes)
8. [Conclusion](#8-conclusion)

---

## 1. Introduction

### 1.1 Lab Overview

This document covers Part 3 of the SIEM HomeLab series. Part 1 deployed the Wazuh Indexer, and Part 2 deployed the Wazuh Dashboard. Part 3 deploys the Graylog Server on a dedicated Ubuntu Server virtual machine and prepares it to act as the log ingestion, processing, buffering, and forwarding layer for the lab architecture.

> **Note:** Every screenshot is retained in this document. Incorrect attempts and later fixes are not removed; they are marked clearly so the troubleshooting path remains visible.

### 1.2 What Is Graylog?

Graylog is used here as the universal log collection and processing layer. It can receive logs from endpoints, firewalls, antivirus platforms, and other services that send events in syslog or compatible formats. Graylog also provides a web interface for managing inputs, streams, pipelines, enrichment, routing, and index-related behavior without relying entirely on manual YAML configuration.

In this HomeLab, Graylog sits between the Wazuh Manager side of the architecture and the Wazuh Indexer. The intended flow is: Wazuh Agents send events to the Wazuh Manager; Fluent on the Wazuh Manager collects the `alerts.json` output; Fluent forwards those alerts to a Graylog input; and Graylog then forwards the processed events to the Wazuh Indexer.

> **Important:** Graylog can buffer events to disk if the backend storage is temporarily unavailable. This provides operational leeway, but it is not the same as true high availability.

### 1.3 Role in the SIEM Architecture

1. Wazuh Agents — endpoint-side collection from monitored systems.
2. Wazuh Manager — receives agent events and produces `alerts.json`.
3. Fluent / log forwarder — reads `alerts.json` and forwards events to Graylog.
4. Graylog Server — receives, parses, enriches, buffers, and forwards events.
5. Wazuh Indexer — stores searchable event data.
6. Wazuh Dashboard / Grafana — provide visualization and investigation interfaces.

Although Wazuh can send data directly to the Wazuh Indexer using Filebeat or Logstash, this lab intentionally places Graylog in the middle because it provides easier log handling, enrichment, routing, and web-based management capabilities.

---

## 2. Lab Environment

### 2.1 Virtual Machine Specifications

| Setting | Value |
|---|---|
| **Role** | Graylog Server — Single Node |
| **Operating System** | Ubuntu Server 24.04.4 LTS |
| **Hypervisor** | VMware Workstation |
| **CPU** | 1 Processor, 4 Cores |
| **RAM** | 6 GB |
| **Disk** | 80 GB SSD / single virtual disk file |
| **Hostname** | `graylog-server` |
| **Static IP Address** | `192.168.71.101` |
| **Wazuh Indexer IP** | `192.168.71.100` |
| **Wazuh Dashboard IP** | `192.168.71.103` |

### 2.2 Locked Software Versions Used in This Lab

| Component | Version / Selection |
|---|---|
| **Graylog Server** | `7.0.5` |
| **Wazuh Manager / Server** | `4.14.4` |
| **Wazuh Indexer** | `4.14.4` |
| **Java Runtime** | OpenJDK 21 JRE Headless |
| **MongoDB** | `8.0` |

> **Version correction:** This lab locks to Graylog 7.0.5 with Java 21 and MongoDB 8.0.

### 2.3 Port and Service Reference

| Port / Service | Purpose |
|---|---|
| `9000/TCP` | Graylog web interface and REST API in this lab |
| `9200/TCP` | Graylog to Wazuh Indexer HTTPS connection |
| `27017/TCP` | MongoDB local metadata database for Graylog |
| Syslog input ports | Future Graylog inputs for endpoints, firewalls, Wazuh/Fluent, and third-party log sources |

> **Production note:** In production, expose only the required ports and place SIEM components in a dedicated network segment or VLAN. Graylog web access should not be left broadly exposed.

---

## 3. Key Concepts

### 3.1 Graylog as a Log Processing Layer

Graylog is not only a log receiver. It can normalize fields, enrich events, route messages into streams, apply pipelines, and forward processed data to backend storage. This is useful when ingesting logs from multiple technologies with inconsistent field names or formats.

The raw notes also identified a future enrichment use case: Graylog can call external APIs such as VirusTotal, submit fields such as a source IP address, and add the returned intelligence to the event before storage or alerting.

### 3.2 MongoDB Role

MongoDB stores Graylog configuration and metadata. In a production deployment, Graylog is normally deployed as a multi-node cluster. A common resilient design uses three Graylog nodes, each with its own MongoDB component, to improve availability. This HomeLab uses a single Graylog + MongoDB node to keep the environment manageable.

### 3.3 Backend Compatibility and Index Management

Graylog historically supported Elasticsearch up to 7.10.2 because of licensing changes. For this lab, the intended backend is Wazuh Indexer, which is based on OpenSearch. Graylog 7 supports OpenSearch 2.x, so the backend must identify itself correctly as OpenSearch rather than pretending to be Elasticsearch 7.10.2.

Index policy matters because log data is written to disk continuously. Without index rotation and retention policies, disk usage can grow until the backend is full and new events cannot be stored. Graylog provides built-in index management features that will be refined later in the SIEM pipeline.

---

## 4. Installation & Configuration

### Step 1 — Create the Graylog Server Virtual Machine

A new VM was created in VMware Workstation using the same Ubuntu Server 24.04.4 LTS installation baseline used in the earlier parts of the SIEM HomeLab series.

1. Create a new Ubuntu 64-bit virtual machine in VMware Workstation.
2. Assign 1 processor with 4 cores.
3. Assign 6 GB RAM.
4. Create an 80 GB SSD-backed virtual disk as a single file.
5. Install Ubuntu Server 24.04.4 LTS and complete the initial operating system setup.

> **Correction:** During VM creation, the VM name was initially left as Ubuntu 64-bit. It was later corrected to `graylog-server`. The screenshot is retained because it reflects the actual lab process.

![VMware Workstation virtual machine wizard for the Graylog server VM](./screenshots/figure1-vmware-vm-wizard.png)

After Ubuntu Server installation completed, the first login to the `graylog-server` VM confirmed the system was operational.

![First login to the graylog-server Ubuntu Server VM](./screenshots/figure2-first-login.png)

---

### Step 2 — Configure Static IP Address

The Graylog server was assigned a static IP address of `192.168.71.101` so that other SIEM components can reliably reach it.

| Node | IP Address |
|---|---|
| **Graylog Server** | `192.168.71.101` |
| **Wazuh Indexer** | `192.168.71.100` |
| **Wazuh Dashboard** | `192.168.71.103` |

![Static IP configuration for graylog-server at 192.168.71.101](./screenshots/figure3-static-ip-config.png)

---

### Step 3 — Update the System and Install Java 21 Prerequisites

Before installing Graylog, the OS package metadata was updated and available upgrades were applied.

```bash
sudo apt update && sudo apt upgrade
```

When prompted, press `y` to continue with the upgrade.

Graylog depends on Java. The older guide used JDK 11, but this lab uses Java 21 for the locked Graylog 7.0.5 deployment.

```bash
sudo apt install apt-transport-https openjdk-21-jre-headless uuid-runtime pwgen dirmngr gnupg wget
```

When prompted, press `Y` to confirm the installation.

![Java 21 and supporting package installation output](./screenshots/figure4-java21-packages-installation.png)

---

### Step 4 — Install MongoDB 8.0

MongoDB stores Graylog configuration and metadata. The older guide installed MongoDB 4.4, but this lab uses MongoDB 8.0 to match the selected Graylog 7 deployment.

```bash
sudo apt-get install gnupg curl
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt-get update
sudo apt-get install -y mongodb-org
```

![MongoDB 8.0 repository configuration and installation output](./screenshots/figure5-mongodb-installation.png)

---

### Step 5 — Enable and Verify MongoDB

After installation, the `mongod` service was enabled, restarted, and verified as active.

```bash
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
sudo systemctl --type=service --state=active | grep mongod
```

The service check confirmed that MongoDB was running.

![mongod service enabled and running](./screenshots/figure6-mongod-service-active.png)

---

### Step 6 — Install the Graylog Repository and Graylog Server

The Graylog 7.0 repository package was installed, the package index was refreshed, and the `graylog-server` package was installed.

> **Correction:** The source guide referenced Graylog 4.0.5. This deployment uses Graylog 7.0.5, so the Graylog 7.0 repository package is used instead.

```bash
wget https://packages.graylog2.org/repo/packages/graylog-7.0-repository_latest.deb
sudo dpkg -i graylog-7.0-repository_latest.deb
sudo apt-get update
sudo apt-get install graylog-server
```

![Graylog 7.0 repository configuration and graylog-server installation output](./screenshots/figure7-graylog-repository-installation.png)

---

### Step 7 — Prepare Java Truststore for Wazuh Indexer TLS

Graylog will connect to the Wazuh Indexer over HTTPS. Because the Indexer certificate is signed by the Wazuh self-signed root CA, Graylog Java must trust that root CA. The Java `cacerts` file was copied to a Graylog-specific certificate directory and the Wazuh root CA was imported into it.

```bash
sudo mkdir /etc/graylog/server/certs
sudo cp -a /usr/lib/jvm/java-21-openjdk-amd64/lib/security/cacerts /etc/graylog/server/certs/cacerts
```

The `root-ca.pem` file was copied from the Wazuh Indexer VM at `192.168.71.100` to the Graylog server at `192.168.71.101` using WinSCP.

```bash
sudo cp root-ca.pem /etc/graylog/server/certs/root-ca.pem
sudo keytool -importcert -keystore /etc/graylog/server/certs/cacerts -storepass changeit -alias root_ca -file /etc/graylog/server/certs/root-ca.pem
```

When prompted by `keytool`, enter `yes` to trust and import the certificate.

![Wazuh root CA imported into the Graylog Java truststore](./screenshots/figure8-root-ca-java-truststore.png)

---

### Step 8 — Configure Graylog Java Options

The Graylog Java options were updated so Graylog uses the truststore created in the previous step when connecting to HTTPS backends. This edit was made in `/etc/default/graylog-server`.

```bash
nano /etc/default/graylog-server
```

```bash
GRAYLOG_SERVER_JAVA_OPTS="$GRAYLOG_SERVER_JAVA_OPTS -Dlog4j2.formatMsgNoLookups=true -Djavax.net.ssl.trustStore=/etc/graylog/server/certs/cacerts -Djavax.net.ssl.trustStorePassword=changeit"
```

> **Production note:** For production, tune Graylog heap and garbage collection settings instead of relying only on defaults. A common starting point is reserving roughly half of the available system memory for Graylog runtime usage, while still leaving enough RAM for MongoDB and the operating system.

![`/etc/default/graylog-server` updated with Java TLS truststore options](./screenshots/figure9-graylog-java-options-truststore.png)

---

### Step 9 — Generate the Graylog Password Secret

Graylog requires a `password_secret` value for internal cryptographic operations. A 96-character secret was generated and redirected into `password.txt`.

```bash
sudo pwgen -N 1 -s 96 > password.txt
```

![password_secret generated and saved to password.txt](./screenshots/figure10-password-secret-generated.png)

The generated secret was copied into the `password_secret` field in `/etc/graylog/server/server.conf`.

![password_secret pasted into /etc/graylog/server/server.conf](./screenshots/figure11-password-secret-server-conf.png)

---

### Step 10 — Generate the Graylog Root Password Hash

The Graylog web UI root password is not stored directly in `server.conf`. Instead, a SHA-256 hash is generated and pasted into the `root_password_sha2` field.

```bash
sudo echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
```

Copy the generated SHA-256 hash and paste it into the `root_password_sha2` field in `/etc/graylog/server/server.conf`. The default root username is `admin` unless changed in the same configuration file.

![root_password_sha2 value generated for the Graylog admin password](./screenshots/figure12-root-password-sha2-generated.png)

---

### Step 11 — Create the Graylog Backend User in Wazuh Dashboard

Graylog needs credentials to connect to the Wazuh Indexer. The internal user was created from the Wazuh Dashboard at `https://192.168.71.103` using the lab credentials from Part 2.

1. Open `https://192.168.71.103` in a browser.
2. Log in with the Wazuh Dashboard credentials used in Part 2, `admin/admin` in this lab.
3. Navigate to Security.
4. Create a new internal user for Graylog.
5. Set the username to `graylog`.
6. Assign the `admin` backend role for this lab deployment.

> **Credential handling note:** Special characters were avoided for the Graylog backend password. In practice, special characters can break the `elasticsearch_hosts` URL unless they are URL-encoded, so using a simple alphanumeric lab password avoids URL parsing issues.

![Wazuh Dashboard security area used to create an internal user](./screenshots/figure13-wazuh-dashboard-security-user-area.png)

![Internal Wazuh user graylog created with admin backend role](./screenshots/figure14-graylog-internal-user-created.png)

---

### Step 12 — Add Local Hostname Mapping for Wazuh Indexer

A local host mapping was added on `graylog-server` so the name `wazuh-indexer` resolves to the Indexer IP address.

```bash
sudo nano /etc/hosts

192.168.71.100   wazuh-indexer
```

![`/etc/hosts` edited on graylog-server to include wazuh-indexer](./screenshots/figure15-etc-hosts-wazuh-indexer.png)

---

### Step 13 — Configure Graylog Backend Connection to Wazuh Indexer

The Graylog `server.conf` file was updated under the `elasticsearch_hosts` setting to connect to the Wazuh Indexer. The initial attempt used the `wazuh-indexer` hostname.

```text
https://username:password@wazuh-indexer:9200
```

> **Important:** This hostname-based value was later corrected because the Wazuh Indexer certificate contained the IP address in the Subject Alternative Name field, but not the hostname `wazuh-indexer`. The troubleshooting and corrected value are documented in Section 5.

---

### Step 14 — Enable Graylog Server and Review Service Status

After the initial configuration, the Graylog service was enabled and its active state was checked.

```bash
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.service
sudo systemctl --type=service --state=active | grep graylog
```

> **Operational note:** If the service is not already running after installation, start it with `sudo systemctl start graylog-server.service`. The raw notes captured enable and verification commands, followed by log review.

![Graylog service enabled and active](./screenshots/figure16-graylog-service-active.png)

---

### Step 15 — Review Graylog Server Logs

The Graylog server log was tailed to confirm whether the service could connect to the backend Indexer.

```bash
tail -f /var/log/graylog-server/server.log
```

![Graylog server log showing the initial backend connection error](./screenshots/figure17-initial-backend-connection-error.png)

---

## 5. Troubleshooting and Corrections

### 5.1 Issue 1 — TLS Hostname Verification Failure

The first backend connection problem was caused by a certificate hostname mismatch. Graylog attempted to connect to `https://wazuh-indexer:9200`, but the Wazuh Indexer certificate was valid for the IP address `192.168.71.100`, not for the hostname `wazuh-indexer`.

```bash
openssl s_client -connect 192.168.71.100:9200 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

The certificate inspection returned the following relevant subject and Subject Alternative Name details:

```text
subject=C = US, L = California, O = Wazuh, OU = Wazuh, CN = wazuh-indexer
X509v3 Subject Alternative Name:
    IP Address:192.168.71.100
```

> **Correction:** The certificate was valid for `IP Address:192.168.71.100`, not for the hostname `wazuh-indexer`. Therefore, the `elasticsearch_hosts` setting was changed to use the IP address instead of the hostname.

```text
https://username:password@192.168.71.100:9200
```

![Graylog log after fixing the hostname mismatch and exposing the next backend compatibility error](./screenshots/figure18-hostname-fix-backend-compatibility-error.png)

### 5.2 Issue 2 — Backend Reported as Elasticsearch 7.10.2

After the hostname mismatch was corrected, the previous error disappeared. Graylog then connected successfully but detected the backend as Elasticsearch 7.10.2 instead of OpenSearch 2.x. Graylog 7 expects an OpenSearch 2.x-compatible backend in this architecture.

The cause was the Wazuh Indexer setting below, which forces OpenSearch to return a legacy Elasticsearch 7.10.2-compatible version response for older ingestion tools:

```yaml
compatibility.override_main_response_version: true
```

The correction was applied on the Wazuh Indexer VM, not on the Graylog VM.

```bash
sudo cp /etc/wazuh-indexer/opensearch.yml /etc/wazuh-indexer/opensearch.yml.bak
sudo nano /etc/wazuh-indexer/opensearch.yml
```

Change this:

```yaml
compatibility.override_main_response_version: true
```

To this:

```yaml
# compatibility.override_main_response_version: true
```

Then restart the Wazuh Indexer and verify its status:

```bash
sudo systemctl restart wazuh-indexer
sudo systemctl status wazuh-indexer --no-pager
```

Finally, restart Graylog and re-check the server log:

```bash
sudo systemctl restart graylog-server
tail -f /var/log/graylog-server/server.log
```

> **Important:** This was a compatibility correction on the backend. Leaving `compatibility.override_main_response_version` enabled caused Graylog to treat the backend as Elasticsearch 7.10.2 instead of OpenSearch 2.x.

### 5.3 Expose the Graylog Web Interface

After backend connectivity was corrected, Graylog still needed to be exposed on the network for browser access. The HTTP bind address was updated in `/etc/graylog/server/server.conf`.

```bash
sudo nano /etc/graylog/server/server.conf
```

```ini
http_bind_address = 0.0.0.0:9000
```

> **Security note:** Binding to `0.0.0.0` exposes the Graylog web interface on all interfaces. This is acceptable for a controlled HomeLab, but production deployments should restrict access with firewall rules, reverse proxy policy, VPN access, or a management VLAN.

![server.conf edited to bind the Graylog HTTP interface to 0.0.0.0:9000](./screenshots/figure19-http-bind-address-server-conf.png)

Restart Graylog after the web interface bind address change:

```bash
sudo systemctl restart graylog-server
```

At this point, the Graylog web UI should be reachable from a browser.

---

## 6. Verification

### 6.1 Access the Graylog Web Interface

The Graylog web interface was reached successfully in the browser. The default username is `admin` unless changed in `server.conf`. The password is the cleartext password that was hashed earlier and stored as `root_password_sha2`.

```text
http://192.168.71.101:9000
```

![Graylog web login screen reached successfully](./screenshots/figure20-graylog-login-screen.png)

After authenticating, the Graylog web UI loaded successfully.

![Graylog web interface loaded after successful login](./screenshots/figure21-graylog-web-interface.png)

### 6.2 Verification Checklist

| Validation Item | Status |
|---|---|
| Ubuntu Server VM created | Completed |
| Static IP set to `192.168.71.101` | Completed |
| Java 21 prerequisites installed | Completed |
| MongoDB 8.0 installed and `mongod` active | Completed |
| Graylog 7.0 repository and server package installed | Completed |
| Wazuh root CA imported into Graylog Java truststore | Completed |
| `password_secret` and `root_password_sha2` configured | Completed |
| Graylog backend user created in Wazuh Dashboard | Completed |
| TLS hostname mismatch identified and corrected | Completed |
| Wazuh Indexer compatibility override disabled | Completed |
| Graylog HTTP interface exposed on `0.0.0.0:9000` | Completed |
| Graylog web UI reached and login successful | Completed |

---

## 7. Observations & Notes

### 7.1 Graylog Is a Buffer, Not Full High Availability

Graylog can write events to disk temporarily when the backend is unavailable. This helps during short interruptions, but it does not replace a proper high-availability design. A production deployment should use multiple Graylog nodes, resilient MongoDB design, redundant backend storage, and tested failure recovery procedures.

### 7.2 Certificate SANs Matter More Than Hostnames in /etc/hosts

Adding `wazuh-indexer` to `/etc/hosts` made the hostname resolvable, but it did not make the TLS certificate valid for that hostname. The certificate Subject Alternative Name contained only `IP Address:192.168.71.100`. Graylog therefore had to connect to the IP address unless the certificate set is regenerated with `wazuh-indexer` included as a DNS SAN.

### 7.3 Wazuh Indexer Compatibility Override Can Break Graylog Detection

The setting `compatibility.override_main_response_version: true` is useful for older tools that expect an Elasticsearch 7.10.2 response. In this lab, it caused Graylog 7 to classify the Wazuh Indexer incorrectly. Commenting the setting allowed Graylog to detect the backend correctly as OpenSearch-compatible.

### 7.4 Password Secret and Root Password Serve Different Purposes

The `password_secret` value is used internally by Graylog and must be long, random, and stable after initial configuration. The `root_password_sha2` value is the SHA-256 hash of the password used to log in to the Graylog web interface as `admin`. These two values are not interchangeable.

### 7.5 Index and Retention Planning Is Required

Because SIEM data grows continuously, index rotation and retention policies must be planned before real ingestion begins. Graylog can manage index sets, but the retention policy must match available disk capacity and investigative requirements.

---

## 8. Conclusion

Part 3 successfully deploys the Graylog Server on a dedicated Ubuntu Server 24.04.4 LTS virtual machine and prepares it as the log ingestion and forwarding layer for the SIEM HomeLab. The deployment included VM provisioning, static networking, Java 21 prerequisites, MongoDB 8.0, Graylog 7.0, Java truststore configuration for Wazuh Indexer TLS, password and secret configuration, backend user creation, troubleshooting of TLS hostname verification, troubleshooting of the OpenSearch compatibility response, and final web UI access.

The completed Graylog server is reachable at `http://192.168.71.101:9000` and is ready for the next stage of the SIEM pipeline: configuring inputs, routing Wazuh alerts into Graylog, and forwarding processed security events to the Wazuh Indexer.

---

> **Next:** SIEM HomeLab — Part 4: Wazuh Manager
