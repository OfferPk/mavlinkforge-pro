# MAVLinkForge Pro Technical Architecture & Specification

## 1\. MAVLink 2.0 Protocol Architecture

MAVLink (Micro Air Vehicle Link) is a lightweight point-to-point and broadcast messaging protocol designed for unmanned aerial systems, ground control stations, and companion computers.

### Core Architectural Features:

1. **10-Byte Packet Header**:  
   - `0xFD`: Start-of-frame packet delimiter.  
   - `Payload Length` (1 byte): Length of payload data.  
   - `Incompat Flags` (1 byte): Flags that require immediate drop if unrecognized by recipient (e.g. signing).  
   - `Compat Flags` (1 byte): Flags that do not prevent processing if unhandled.  
   - `Sequence Number` (1 byte): Packet counter to detect telemetry loss over lossy radio links.  
   - `System ID` (1 byte): Unique identifier for vehicle / GCS.  
   - `Component ID` (1 byte): Subsystem identifier (autopilot, gimbal, camera, companion computer).  
   - `Message ID` (3 bytes): 24-bit identifier allowing up to 16 million unique telemetry message definitions.  
2. **CRC-16/MCRF4XX Integrity**:  
   - Accumulates bytes starting from payload length through payload end using polynomial \$X^{16} \+ X^{12} \+ X^5 \+ 1\$.  
   - Appends a message-specific `CRC-EXTRA` byte derived from XML definition files to prevent silent desynchronization between differing software builds.  
3. **Payload Zero-Truncation**:  
   - Automatically truncates trailing zeroes in payload fields before transmission, reducing telemetry airtime over 433MHz/915MHz telemetry radios.  
   - The recipient zero-fills the truncated bytes up to the expected message schema length.