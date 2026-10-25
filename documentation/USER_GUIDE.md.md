# MAVLinkForge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your browser.  
2. In the "UAV Packet Generator" panel, select your vehicle System ID (e.g. `1`) and Component ID (e.g. `1` for Autopilot).  
3. Select target message (e.g., `HEARTBEAT`, `ATTITUDE`, or `GLOBAL_POSITION_INT`).  
4. Choose the simulated flight state (e.g., `ArduPilot Guided / Armed`).  
5. Toggle "Enable MAVLink 2.0 Zero-Truncation" to simulate wire bandwidth optimization.  
6. Click **Generate Frame** to serialize the MAVLink packet, calculate CRC-16 with CRC-EXTRA, and inspect the hex dump.  
7. Click **Inject CRC Error** to test receiver error handling boundaries.  
8. Click **Run Normative Audit** to evaluate all 10 protocol rules.

## 2\. Standalone Headless Python CLI

Run automated regression tests and generate compliance reports:

python3 mavlink-cli.py \--audit \--msg HEARTBEAT \--sysid 1