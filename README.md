# OAI 5G SA Network Slicing Emulation (RF Simulator)

This repository contains the configuration and setup instructions for emulating a 5G Standalone (SA) network with Network Slicing using OpenAirInterface (OAI).

The setup utilizes Docker for the 5G Core Network (CN5G) and runs the gNB and nr-UE (User Equipment) directly on the host machine using the RF Simulator (no SDR hardware required).

## Table of Contents

1.  [Architecture](#architecture)
2.  [Prerequisites](#prerequisites)
3.  [Configuration & Fixes](#configuration-fixes)
    * [Core Network](#core-network)
    * [Database Schema](#database-schema)
    * [UE Configuration](#ue-configuration)
4.  [How to Run](#how-to-run)
    * [Start the Core Network](#start-the-core-network)
    * [Start the gNB](#start-the-gnb)
    * [Start the UE](#start-the-ue)
    * [Verify Traffic](#verify-traffic)
5.  [File Structure](#file-structure)
6.  [Verification](#verification)
7.  [Logs](#logs)
8.  [Troubleshooting Tips](#troubleshooting-tips)

---

## 1. Architecture

This setup provides a basic yet complete 5G SA (Standalone) network emulation on a single Linux machine. Key features include:

* **Core Network (Dockerized)**: OAI CN5G (v2.1.0 images).
    Slice 1: SST=1, SD=1 (eMBB/Default)
    Slice 2: SST=2, SD=2 (Custom/OAI)
    Functions: AMF, SMF (x2), UPF (x2), NRF (x2), UDM, UDR, AUSF, NSSF.
    Database: MySQL 8.0.
* **RAN (Host Machine)**: OAI gNB (Monolithic).
* **UE (Host Machine)**: OAI nr-uesoftmodem (RF Simulator).

## 2. Prerequisites

* **Operating System**: Ubuntu 20.04 / 22.04 (Low-latency kernel recommended but not required for RFSim).
* **Sudo Privileges**: Required for installation and managing system services.
* **Git**: For cloning this repository and managing versions.
    $ sudo apt update
    $ sudo apt install git

* **Software**: Docker & Docker Compose, OpenAirInterface repository (built for gNB and UE), and oai-cn5g-fed repository (for Core Network scripts).

## 3. Configuration & Fixes

This setup addresses several common integration issues between OAI components.

### Core Network

Main file: docker-compose-slicing-basic-nrf.yaml

MySQL 8.0 Compatibility: The MySQL service command is forced to use legacy authentication:
    command: --default-authentication-plugin=mysql_native_password

NRF Routing: Common functions (UDM, AUSF) are explicitly configured to register with oai-nrf-slice1 via slicing_base_config.yaml to prevent discovery failures.

Latency Tuning: AMF http_request_timeout increased to 5000 (ms) to prevent authentication timeouts under high CPU load.

1.  **Add Open5GS repository and install:**
    $ sudo add-apt-repository ppa:open5gs/latest
    $ sudo apt update
    $ sudo apt install open5gs

2.  [cite_start]**Install Open5GS WebUI:** The WebUI facilitates UE subscription management and monitoring. [cite: 1]
    $ sudo apt install open5gs-webui

### Database Schema

Main file: oai_db2.sql

The default SQL dump has been modified to prevent ERROR 1062 (Duplicate entry) during initialization.
    Fix: The PRIMARY KEY on SessionManagementSubscriptionData was replaced with a standard INDEX to allow multiple slice subscriptions for the same UE.

### UE Configuration

Main file: ue.conf

S-NSSAI Matching: The Slice Differentiator (SD) is standardized to Hex format (0x1) in both the snssaiList (Registration) and pduSessions (Establishment) blocks to match the Core Network's expectations.

PLMN: Configured for MCC=208, MNC=95 (Length=2).

## 4. How to Run

### Start the Core Network

Navigate to your docker-compose directory:
    # Clean up any stale state (Crucial for DB initialization)
    $ docker compose -f docker-compose-slicing-basic-nrf.yaml down -v
    # Start the Core
    $ docker compose -f docker-compose-slicing-basic-nrf.yaml up -d
    
Verification: Wait 30-60 seconds. Check that MySQL is healthy and SMFs are registered:
    $ docker ps | grep mysql  # Must be (healthy)
    $ docker logs oai-smf-slice1 | grep "NF Update"

### Start the gNB

Navigate to your openairinterface5g directory:
    $ sudo nice -n -10 ./cmake_targets/ran_build/build/nr-softmodem \ -O $PWD/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.pci0.rfsim.conf \ --rfsim

### Start the UE

In a separate terminal:
    $ sudo nice -n -10 ./cmake_targets/ran_build/build/nr-uesoftmodem \ -O $PWD/targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \ -C 3619200000 --band 78 --rfsim

### Verify Traffic

Once the UE log shows PDU Session Establishment Accept and an IP (e.g., 12.2.1.2), test the connection:
    # Ping Google DNS through the UE tunnel interface
    $ ping -I oaitun_ue1 8.8.8.8

### 5. File Structure

├── README.md                          # This file

├── Core_configs/

    ├── docker-compose-slicing-basic-nrf.yaml

    ├── nssf_slice_config.yaml

    ├── slicing_base_config.yaml

    ├── slicing_slice1_config.yaml

    ├── slicing_slice2_config.yaml

    └── oai_db2.sql

├── RAN_UE_configs/

    ├── gnb.sa.band78.fr1.106PRB.pci0.rfsim.conf
    
    ├── neighbour-config-rfsim.conf

    └── ue.conf

└── logs/

    ├── docker_logs_oai_amf.log          # Example AMF logs for UE registration
    
    ├── docker_logs_oai_smf-slice1.log   # Example SMF slice1 logs for PDU session setup
    
    ├── docker_logs_oai_smf-slice2.log   # Example SMF slice2 logs for PDU session setup
    
    ├── docker_logs_oai_upf-slice1.log   # Example UPF slice1 logs for UPF session creation
    
    ├── docker_logs_oai_upf-slice2.log   # Example UPF slice2 logs for UPF session creation
    
    ├── docker_logs_oai_nrf-slice1.log   # Example NRF slice1 logs for Network Functions catalog
    
    ├── docker_logs_oai_nrf-slice2.log   # Example NRF slice2 logs for Network Functions catalog
    
    ├── docker_logs_gnb_sa_band78_fr1_106PRB_pci0_rfsim.log          # Example gNB logs
    
    ├── docker_logs_ue.log               # Example UE logs
    
    ├── docker_logs_oai_udr.log          # Example UDR logs for unified UE DataBase
    
    ├── docker_logs_mysql.log            # Example MySQL DataBase logs
    
    └── ip_a.log                         # Example ip a output

### 6. Verification

After starting the network, verify its operation using the following methods:

Monitor Core Network Logs: Observe real-time logs for UE registration, PDU session establishment, and internal NF communication. 
    $ docker logs oai-amf -f          # For AMF logs
    $ docker logs oai-smf-slice1 -f   # For SMF slice1 logs
    $ docker logs oai-smf-slice2 -f   # For SMF slice2 logs
    $ docker logs oai-upf-slice1 -f   # For UPF slice1 logs
    $ docker logs oai-upf-slice2 -f   # For UPF slice2 logs
    $ docker logs oai-udr -f          # For UPF slice2 logs
    $ docker logs mysql -f            # For MySQL DataBase logs
    $ # It is necessary to verify that the MySQL DataBase is healthy and then confirm that the UDR is connected to the MySQL DataBase.
    $ # You can check other services like nrf, nssf, udm, ausf

Check gNB and UE Logs: Verify gNB's NG Setup success and UE's successful registration and PDU session establishment available on the starting commands: 
    $ sudo nice -n -10 ./cmake_targets/ran_build/build/nr-softmodem -O $PWD/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.pci0.rfsim.conf --rfsim
    $ sudo nice -n -10 ./cmake_targets/ran_build/build/nr-uesoftmodem -O $PWD/targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf -C 3619200000 --band 78 --rfsim

### 7. Logs

Example log outputs demonstrating successful operation are included in the logs/ directory of this repository. These logs capture critical messages during network startup, UE registration, and PDU session establishment from various components.

### 8. Troubleshooting Tips

Error: MySQL Exited (1)
    Cause: Schema conflict or Auth plugin mismatch.
    Fix: Check oai_db2.sql for duplicate keys. Ensure mysql_native_password is set. Run down -v.

Error: UDR: Can't connect to MySQL (111)
    Cause: MySQL is not healthy yet.
    Fix: Wait 60s. Restart UDR: docker restart oai-udr.

Error: AMF: Authentication failed (Timeout)
    Cause: CPU load causing DB latency >1s.
    Fix: Increase http_request_timeout in AMF config or use nice for RAN processes.

Error: UE: Registration Reject (Message too short)
    Cause: User not found in DB.
    Fix: Verify DB content. Re-import SQL by running down -v and restarting.

Error: UE: NSSAI parameters not match
    Cause: Config mismatch or UDR query fail.
    Fix: Ensure ue.conf uses sd=0x1 (Hex) in both blocks. Check if UDR is connected to DB.

