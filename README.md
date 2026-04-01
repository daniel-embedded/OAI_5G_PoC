# OAI 5G Network Slicing Emulation

This repository contains a Proof of Concept (PoC) for emulating a 5G Service-Based Architecture (SBA) with strict network slicing. It is designed to gather quantitative control-plane metrics and evaluate dynamic PDU session transitions, providing a reproducible testbed for comparing OpenAirInterface (OAI) implementation overhead against Open5GS architectures for IEEE SMC 2026 research.

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
7.  [Troubleshooting Tips](#troubleshooting-tips)

## 1. Architecture Overview

This testbed decouples the 5G Core from the Radio Access Network (RAN) and User Equipment (UE) to allow for precise control-plane stress testing. 

* **Core Network:** Deployed via `docker-compose-slicing-basic-nrf.yaml`. Features isolated Network Repository Functions (NRFs), Session Management Functions (SMFs), and User Plane Functions (UPFs) to enforce strict traffic separation.
* **RAN & UEs:** Emulated using **UERANSIM**. To simplify the radio layer and isolate core metrics, this setup bypasses the `oai-gnb`, `oai-nr-ue`, and `gnbsim` containers. We exclusively run the `ueransim` service defined in `docker-compose-slicing-ransim.yaml`.
* **Topology:** 1 simulated gNB serving 8 UEs (IMSIs `208950000000031` through `208950000000038`).
* **Active Slices:**
  * **Slice 1:** `SST: 1, SD: 000001` (DNN: `default`)
  * **Slice 2:** `SST: 2, SD: 000002` (DNN: `oai`)
  
## 2. Prerequisites

* **Operating System**: Ubuntu 20.04 / 22.04 (Low-latency kernel recommended but not required for RFSim).
* **Sudo Privileges**: Required for installation and managing system services.
* **Git**: For cloning this repository and managing versions.
    $ sudo apt update
    $ sudo apt install git

* **Software**: Docker & Docker Compose, OpenAirInterface repository (built for gNB and UE), and oai-cn5g-fed repository (for Core Network scripts).

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

### 3. NSSF Discovery (`nssf_slice_config.yaml`)
To prevent the Access and Mobility Management Function (AMF) from timing out during SMF discovery, the `nrfId` and `nrfNfMgtUri` fields for Slices 1, 2, and 3 were updated to point directly to the active Docker IPv4 addresses (e.g., `192.168.70.136`, `192.168.70.146`) instead of unresolved FQDNs.

### 4. UERANSIM APN Parameter (`docker-compose-slicing-ransim.yaml`)
Ensure the UERANSIM environment variables use the legacy `- APN=default` flag instead of `DNN`. The `ueransim:latest` container entrypoint script will fatally crash if `APN` is missing from the compose file.

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

By default, UERANSIM boots all 8 UEs and requests PDU sessions strictly on Slice 1. To investigate control-plane overhead during inter-slice mobility, you can manually trigger secondary PDU sessions using the nr-cli tool.

Establish a connection to Slice 2 (DNN: oai)
    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-establish IPv4 --sst 2 --sd 000002 --dnn oai"

(Note: Watch the core logs for the T3580 timer to measure session establishment latency).

Verify active sessions and multiple virtual TUN interfaces:
    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-list"
    $ docker exec -it ueransim ifconfig

Release the original Slice 1 connection:
    $ docker exec -it ueransim ./nr-cli imsi-208950000000031 -e "ps-release 1"

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

    └── oai_db2.sql

├── RAN_UE_configs/

    └── docker-compose-slicing-ransim.yaml

└── logs/

    ├── docker_logs_amf.log          # Example AMF logs for UE registration
    
    ├── docker_logs_smf-slice1.log   # Example SMF slice1 logs for PDU session setup
    
    ├── docker_logs_smf-slice2.log   # Example SMF slice2 logs for PDU session setup
    
    ├── docker_logs_upf-slice1.log   # Example UPF slice1 logs for UPF session creation
    
    ├── docker_logs_upf-slice2.log   # Example UPF slice2 logs for UPF session creation
    
    ├── docker_logs_nrf-slice1.log   # Example NRF slice1 logs for Network Functions catalog
    
    ├── docker_logs_nrf-slice2.log   # Example NRF slice2 logs for Network Functions catalog
    
    ├── docker_logs_gnb_ue.log          # Example gNB and UE logs
    
    ├── docker_logs_udr.log          # Example UDR logs for unified UE DataBase
    
    ├── docker_logs_mysql.log            # Example MySQL DataBase logs
    
    └── ip_a.log                         # Example ip a output

## 6. Verification

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
