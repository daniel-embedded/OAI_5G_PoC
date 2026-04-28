# OAI 5G Network Slicing Emulation

This repository contains a Proof of Concept (PoC) for emulating a 5G Service-Based Architecture (SBA) with strict network slicing. It is designed to gather quantitative control-plane metrics and evaluate dynamic PDU session transitions, providing a reproducible testbed for comparing OpenAirInterface (OAI) implementation overhead against other architectures.

The baseline deployment and core configuration files in this project originated from the official OAI tutorial: [DEPLOY_SA5G_SLICING.md](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/develop/docs/DEPLOY_SA5G_SLICING.md).

## Table of Contents

1.  [Architecture Overview](#architecture-overview)
2.  [Prerequisites](#prerequisites)
3.  [Critical Configuration Changes](#critical-configuration-changes)
    * [Database Schema Override](#database-schema-override)
    * [UPF Hostname Resolution](#upf-hostname-resolution)
    * [NSSF Discovery](#nssf-discovery)
    * [UERANSIM APN Parameter](#ueransim-apn-parameter)
4.  [Deployment Instructions](#deployment-instructions)
5.  [File Structure](#file-structure)
6.  [Verification](#verification)
    * [Verifying basic operation](#verifying-basic-operation)
    * [Diagnostic commands](#diagnostic-commands)
    * [User Plane triggering](#user-plane-triggering)
    * [Routing and connectivity](#routing-and-connectivity)
    * [Deep packet inspection](#deep-packet-inspection)
7.  [Troubleshooting Tips](#troubleshooting-tips)

## 1. Architecture Overview

This testbed decouples the 5G Core from the Radio Access Network (RAN) and User Equipment (UE) to allow for precise control-plane stress testing. 

* **Core Network:** Deployed via `docker-compose-slicing-basic-nrf.yaml`. Features isolated Network Repository Functions (NRFs), Session Management Functions (SMFs), and User Plane Functions (UPFs) to enforce strict traffic separation.
* **RAN & UEs:** Emulated using **UERANSIM**. To simplify the radio layer and isolate core metrics, this setup bypasses the `oai-gnb`, `oai-nr-ue`, and `gnbsim` containers. We exclusively run the `ueransim` service defined in `docker-compose-slicing-ransim.yaml`.
* **Topology:** 1 simulated gNB serving 8 UEs (IMSIs `208950000000031` through `208950000000038`).
* **Active Slices:**
  * **Slice 1:** `SST: 1, SD: 000001` (DNN: `default`)
  * **Slice 2:** `SST: 2, SD: 000002` (DNN: `oai`)
  * **Slice 222:** `SST: 222, SD: 000001` (DNN: `gatekeeper`)

                      +-------------------------------------------------------------------------------------------------------+
                      |                              +------------------------------------------------------------------+     |
                      |                              |                       SHARED CONTROL PLANE                       |     |
                      |                              |                                                                  |     |
                      |                              |  +-------------+    +----------+    +----------+    +----------+ |     |
                      |                              |  |   oai-nssf  |    | oai-ausf |    | oai-udm  |    | oai-udr  | |     |
                      |                              |  |.70.132 (N22)|    | .70.135  |    | .70.134  |    | .70.133  | |     |
                      |                              |  +------+------+    +----+-----+    +----+-----+    +----+-----+ |     |
                      |                              |         |                |               |               |       |     |
                      |                              |         |                |               |               |       |     |
                      |                              |  +------+------+         |          +----+-----+         |       |     |
                      |                              |  |   oai-amf   +---------+          | oai-nrf  |         |       |     |
                      |                              |  |   .70.138   +--------------------+ .70.136  +---------+       |     |
                      |                              |  +------+------+                    +----+-----+                 |     |
                      |                              |         |                                |                       |     |
                      |                              +---------|--------------------------------|-----------------------+     |
                      |                                        |                                |                             |
                      |+-----------------------+               |                                |                             |
                      ||       RAN/UEs         |               |             +------------------+-----------------+           |
                      ||     (ueransim)        |      +--------|--------+    | +--------+-------|-+      +--------|-------+   |
                      ||                       |      |    SLICE 222    |    | |      SLICE 1   | |      |      SLICE 2   |   |
                      ||  +-----------------+  |      |  (SST:222/SD:1) |    | |   (SST:1/SD:1) | |      |   (SST:2/SD:2) |   |
                      ||  |    8 x UEs      |  |      |                 |    | |                | |      |                |   |
                      ||  | (IMSI ...031-   +----(N1)-->                |    | |                | |      |                |   |
                      ||  |  IMSI ...038)   |  |      |  +-----------+  |    | |  +-----------+ | |      |  +-----------+ |   |
                      ||  +--------+--------+  |      |  | oai-smf-  |<-(N11)+ +->| oai-smf-  |<+ +-(N11)-> | oai-smf-  | |   |
                      ||           |           |      |  | slice222  |  |      |  | slice1    |        |    | slice2    | |   |
                      ||  +--------+--------+  |      |  | .70.141   |  |      |  | .70.139   |        |    | .70.140   | |   |
                      ||  |      gNB        |  |      |  +-----+-----+  |      |  +-----+-----+        |    +-----+-----+ |   |
                      ||  | 192.168.70.152  +----(N2)-->       | (N4)   |      |        | (N4)         |          | (N4)  |   |
                      ||  +----+-------+----+  |      |  +-----+-----+  |      |  +-----+-----+        |    +-----+----+  |   |
                      ||       |   |   |       |      |  | oai-upf-  |  |      |  | oai-upf-  |        |    | oai-upf- |  |   |
                      |+-------|---|---|-------+      |  | slice222  |  |      |  | slice1    |        |    | slice2   |  |   |
                      |        |   |   +--------(N3)---->| .70.144   +--(N6)---+  | .70.142   +--(N6)---+   | .70.143  +--+   |
                      |        |   +------------(N3)-------------------------->|  +-----------+        |    +----------+  |   |
                      |        +----------------(N3)-------------------------------------------------->|               |  |   |
                      |                               +-----------------+      +-----------------------+               |  |   |
                      |                                                                                                |  |   |
                      |                                                                                                |  |   |
                      |                                                                              +-----------------+--+--+|
                      |                                                                              |      oai-ext-dn       ||
                      |                                                                              |       .70.145         ||
                      |                                                                              |     (Data Network)    ||
                      |                                                                              +-----------------------+|
                      |                                                                              |     mysql (.70.131)   ||
                      |                                                                              +-----------------------+|
                      +-------------------------------------------------------------------------------------------------------+

## 2. Prerequisites

**Operating System**: Ubuntu 20.04 / 22.04 (Low-latency kernel recommended but not required for RFSim).
**Sudo Privileges**: Required for installation and managing system services.
**Git**: For cloning this repository and managing versions.

    $ sudo apt update

    $ sudo apt install git

**Software**: Docker & Docker Compose, OpenAirInterface repository (built for gNB and UE), and oai-cn5g-fed repository (for Core Network scripts).

## 3. Critical Configuration Changes

To achieve multi-slice mobility for a single UE, several modifications were made to the default OAI tutorial files.

### 1. Database Schema Override (`oai_db2.sql`)
By default, the OAI MySQL initialization script restricts UEs to a single data plane profile. To allow UEs to simultaneously connect to Slice 1 and Slice 2, the `PRIMARY KEY` constraint on the Session Management table must be removed from the blueprint prior to container initialization.
* Commented out the `ALTER TABLE SessionManagementSubscriptionData ADD PRIMARY KEY` block at the bottom of the SQL dump.
* Provisioned 8 UEs with multiple JSON profiles across both `AccessAndMobilitySubscriptionData` and `SessionManagementSubscriptionData`.

### 2. UPF Hostname Resolution (`slicing_sliceX_config.yaml`)
When duplicating the SMF/UPF YAML configurations for isolated slices, the generic `host: oai-upf` variable causes NXDOMAIN DNS failures on the N4 interface. 
* Updated the NFs block in `slicing_slice1_config.yaml` to specifically target `host: oai-upf-slice1`.
* Updated the NFs block in `slicing_slice2_config.yaml` to specifically target `host: oai-upf-slice2`.
* Updated the NFs block in `slicing_slice222_config.yaml` to specifically target `host: oai-upf-slice222`.

### 3. NSSF Discovery (`nssf_slice_config.yaml`)
To prevent the Access and Mobility Management Function (AMF) from timing out during SMF discovery, the `nrfId` and `nrfNfMgtUri` fields for Slices 1, 2, and 222 were updated to point directly to the active Docker IPv4 addresses (e.g., `192.168.70.136`, `192.168.70.146`) instead of unresolved FQDNs.

### 4. UERANSIM APN Parameter (`docker-compose-slicing-ransim.yaml`)
Ensure the UERANSIM environment variables use the legacy `- APN=gatekeeper` flag instead of `DNN`. The `ueransim:latest` container entrypoint script will fatally crash if `APN` is missing from the compose file.

## 4. Deployment Instructions
Navigate to your docker-compose directory.

**1. Clean previous environments**
To ensure the database ingests the modified `oai_db2.sql` without primary key collisions, wipe any existing Docker volumes:

    $ docker compose -f docker-compose-slicing-basic-nrf.yaml down -v

**2. Initialize the 5G Core**

    $ docker compose -f docker-compose-slicing-basic-nrf.yaml up -d

Wait approximately 15-20 seconds for the MySQL database to populate and the AMF to register with the isolated NRFs. It is important to ensure that the UDR function connected to the MySQL database:

    $ docker logs oai-udr | grep "MySQL"

If not, restart both containers

    $ docker restart oai-udr mysql

**3. Launch the RAN and UEs**
Bring up strictly the UERANSIM container to generate the gNB and 8 UEs.

    $ docker compose -f docker-compose-slicing-ransim.yaml up -d ueransim

Dynamic Slice Transitions & Testing

By default, UERANSIM boots all 8 UEs and requests PDU sessions strictly on Slice 222. To investigate control-plane overhead during inter-slice mobility, you can manually trigger secondary PDU sessions using the nr-cli tool.

Establish a connection to Slice 2 (DNN: oai)

    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-establish IPv4 --sst 2 --sd 000002 --dnn oai"

Establish a connection to Slice 1 (DNN: default)
    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-establish IPv4 --sst 1 --sd 000001 --dnn default"

(Note: Watch the core logs for the T3580 timer to measure session establishment latency).

Verify active sessions and multiple virtual TUN interfaces:

    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-list"
    $ docker exec -it ueransim ifconfig

Release the original Slice 222 connection:

    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-release 222"

Batch Testing:
To sequentially transition all 8 UEs to the secondary slice for load testing:

    $ for i in {32..38}; do docker exec -it ueransim ./nr-cli imsi-2089500000000$i -e "ps-establish IPv4 --sst 2 --sd 000002 --dnn oai"; done

## 5. File Structure

├── README.md                          # This file

├── Core_configs/

    ├── docker-compose-slicing-basic-nrf.yaml

    ├── nssf_slice_config.yaml

    ├── slicing_base_config.yaml

    ├── slicing_slice1_config.yaml

    ├── slicing_slice2_config.yaml

    ├── slicing_slice222_config.yaml

    └── oai_db2.sql

├── RAN_UE_configs/

    ├── gnb.yaml

    └── docker-compose-slicing-ransim.yaml

└── logs/

    ├── docker_logs_amf.log          # Example AMF logs for UE registration
    
    ├── docker_logs_smf-slice1.log   # Example SMF slice1 logs for PDU session setup
    
    ├── docker_logs_smf-slice2.log   # Example SMF slice2 logs for PDU session setup
    
    ├── docker_logs_smf-slice222.log # Example SMF slice222 logs for PDU session setup
    
    ├── docker_logs_upf-slice1.log   # Example UPF slice1 logs for UPF session creation
    
    ├── docker_logs_upf-slice2.log   # Example UPF slice2 logs for UPF session creation
    
    ├── docker_logs_upf-slice222.log # Example UPF slice222 logs for UPF session creation
    
    ├── docker_logs_nrf.log          # Example NRF logs for Network Functions catalog
    
    ├── docker_logs_gnb_ue.log       # Example gNB and UE logs
    
    ├── docker_logs_udm.log          # Example UDM logs for User Data catalog
    
    ├── docker_logs_udr.log          # Example UDR logs for unified UE DataBase
    
    ├── docker_logs_mysql.log        # Example MySQL DataBase logs
    
    └── active_routing_table.log     # Example active routing table output

## 6. Verification

**1. Verifying basic operation**

After starting the network, verify its operation using the following methods:

Monitor Core Network Logs: Observe real-time logs for UE registration, PDU session establishment, and internal NF communication. 

    $ docker logs oai-amf -f          # For AMF logs
    $ docker logs oai-smf-slice1 -f   # For SMF slice1 logs
    $ docker logs oai-smf-slice2 -f   # For SMF slice2 logs
    $ docker logs oai-upf-slice1 -f   # For UPF slice1 logs
    $ docker logs oai-upf-slice2 -f   # For UPF slice2 logs
    $ docker logs oai-udr -f          # For UPF slice2 logs
    $ docker logs mysql -f            # For MySQL DataBase logs

It is necessary to verify that the MySQL DataBase is healthy and then confirm that the UDR is connected to the MySQL DataBase.

**2. Diagnostic commands**

For Control Plane Verification (Logging), diagnostic commands, and testing methodologies, validate this architecture checking the NF logs is the fastest way to verify SBI registrations and control plane signaling.

Check NSSF to NRF Registration:

    $ docker logs oai-nssf | grep -i "nrf"

Monitor AMF Slice Selection Queries (NSSF Handshake):

    $ docker logs oai-amf | grep -E "NSSF|nsselection"

Verify SMF Connections (NRF and UDM):

    $ docker logs oai-smf-slice222 | grep -E "success|nrf|Nrf|NRF|UDM|response"

Monitor UE Attachment Status (gNB/UE perspective):

    $ docker logs ueransim | tail -n 100

**3. User Plane triggering**

When you need to force specific UEs to request distinct slices (S-NSSAI) without relying on default startup parameters.

Trigger a specific PDU Session manually via CLI:

    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-establish IPv4 --sst 1 --sd 000001 --dnn default"
    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-establish IPv4 --sst 2 --sd 000002 --dnn oai"

Trigger multiple sessions in bulk (Bash Loop):

    $ for i in {31..38}; do docker exec -it ueransim ./nr-cli imsi-2089500000000$i -e "ps-establish IPv4 --sst 2 --sd 000002 --dnn oai"; done

**4. Routing and connectivity**

Verifying that the external data network (ext-dn) knows how to route the return traffic to the correct UPF container based on the UE's subnet.

View active routing table in the Ext-DN gateway:

    $ docker exec -it oai-ext-dn ip route

Inject static return routes manually (if missed by Docker Compose):

    $ docker exec -it oai-ext-dn ip route add 12.1.1.0/24 via 192.168.70.142
    $ docker exec -it oai-ext-dn ip route add 12.2.1.0/24 via 192.168.70.143
    $ docker exec -it oai-ext-dn ip route add 12.3.1.0/24 via 192.168.70.144

Test external connectivity from a specific UE interface(Must be run from inside the UERANSIM container):

    $ docker exec -it ueransim ping -I uesimtun0 8.8.8.8

**5. Deep packet inspection**

Inspecting the actual packets guarantees that routing isolation and JSON signaling are behaving exactly as expected.

Install tcpdump dynamically on a stripped OAI container (e.g., UPF):

    $ docker exec -u root -it oai-upf-slice222 apt-get update
    $ docker exec -u root -it oai-upf-slice222 apt-get install -y tcpdump

Verify User Plane isolation (listen for ICMP traffic on a specific UPF):

    $ docker exec -it oai-upf-slice222 tcpdump -i eth0 icmp

Verify Control Plane signaling (read the JSON HTTP/2 payload between AMF and NSSF):

    $ docker exec -it oai-amf tcpdump -i eth0 host oai-nssf -A

This toolkit covers the entire lifecycle of troubleshooting a containerized 5G core—from initial NF discovery down to the ICMP packets on the user plane. It is a highly robust methodology that will serve you well when you eventually move on to benchmarking network performance.

You can check other services like nrf, nssf, udm, ausf. Example log outputs demonstrating successful operation are included in the logs/ directory of this repository. These logs capture critical messages during network startup, UE registration, and PDU session establishment from various components.

## 7. Troubleshooting Tips

Error: MySQL Exited (1)
    Cause: Schema conflict or Auth plugin mismatch.
    Fix: Check oai_db2.sql for duplicate keys. Ensure mysql_native_password is set. Run down -v.

Error: UDR: Can't connect to MySQL (111)
    Cause: MySQL is not healthy yet.
    Fix: Wait 60s. Restart UDR and MySQL: docker restart oai-udr mysql.

Error: UE: NSSAI parameters not match
    Cause: Config mismatch or UDR query fail.
    Fix: Ensure ue.conf uses sd=0x1 (Hex) in all blocks. Check if UDR is connected to DB.
