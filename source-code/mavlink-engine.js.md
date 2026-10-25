/\*\*  
 \* MAVLink 2.0 Core Wire Framing & CRC-16/MCRF4XX Checksum Engine  
 \* Implements 10-byte header, CRC-EXTRA message seeding, zero-truncation,  
 \* and standard message serialization (HEARTBEAT, ATTITUDE, GLOBAL\_POSITION\_INT).  
 \*/

const MAVLINK\_V2\_MAGIC \= 0xFD;  
const MAVLINK\_V1\_MAGIC \= 0xFE;

// Standard MAVLink Message Definitions with CRC-EXTRA bytes  
const MAVLINK\_MSG\_SPECS \= {  
  HEARTBEAT: {  
    msgId: 0,  
    crcExtra: 50,  
    fields: \[  
      { name: 'custom\_mode', type: 'uint32' },  
      { name: 'type', type: 'uint8' },  
      { name: 'autopilot', type: 'uint8' },  
      { name: 'base\_mode', type: 'uint8' },  
      { name: 'system\_status', type: 'uint8' },  
      { name: 'mavlink\_version', type: 'uint8' }  
    \]  
  },  
  ATTITUDE: {  
    msgId: 30,  
    crcExtra: 39,  
    fields: \[  
      { name: 'time\_boot\_ms', type: 'uint32' },  
      { name: 'roll', type: 'float' },  
      { name: 'pitch', type: 'float' },  
      { name: 'yaw', type: 'float' },  
      { name: 'rollspeed', type: 'float' },  
      { name: 'pitchspeed', type: 'float' },  
      { name: 'yawspeed', type: 'float' }  
    \]  
  },  
  GLOBAL\_POSITION\_INT: {  
    msgId: 33,  
    crcExtra: 104,  
    fields: \[  
      { name: 'time\_boot\_ms', type: 'uint32' },  
      { name: 'lat', type: 'int32' },  
      { name: 'lon', type: 'int32' },  
      { name: 'alt', type: 'int32' },  
      { name: 'relative\_alt', type: 'int32' },  
      { name: 'vx', type: 'int16' },  
      { name: 'vy', type: 'int16' },  
      { name: 'vz', type: 'int16' },  
      { name: 'hdg', type: 'uint16' }  
    \]  
  }  
};

class MAVLinkEngine {  
  constructor(sysId \= 1, compId \= 1\) {  
    this.sysId \= sysId;  
    this.compId \= compId;  
    this.sequence \= 0;  
  }

  getNextSequence() {  
    const seq \= this.sequence;  
    this.sequence \= (this.sequence \+ 1\) & 0xFF;  
    return seq;  
  }

  /\*\*  
   \* MAVLink CRC-16/MCRF4XX polynomial accumulator  
   \* Polynomial: X^16 \+ X^12 \+ X^5 \+ 1 (0x1021 reversed: 0x8408)  
   \*/  
  static accumulateCRC(byte, crc) {  
    let tmp \= byte ^ (crc & 0xFF);  
    tmp \= (tmp ^ (tmp \<\< 4)) & 0xFF;  
    return ((crc \>\> 8\) ^ (tmp \<\< 8\) ^ (tmp \<\< 3\) ^ (tmp \>\> 4)) & 0xFFFF;  
  }

  static calculateChecksum(bytes, crcExtra) {  
    let crc \= 0xFFFF;  
    for (let i \= 0; i \< bytes.length; i++) {  
      crc \= MAVLinkEngine.accumulateCRC(bytes\[i\], crc);  
    }  
    crc \= MAVLinkEngine.accumulateCRC(crcExtra, crc);  
    return crc;  
  }

  /\*\*  
   \* Truncate trailing zero bytes per MAVLink 2.0 specification  
   \*/  
  static truncatePayload(payloadBytes) {  
    let lastNonZero \= payloadBytes.length;  
    while (lastNonZero \> 0 && payloadBytes\[lastNonZero \- 1\] \=== 0\) {  
      lastNonZero--;  
    }  
    return payloadBytes.slice(0, lastNonZero);  
  }

  /\*\*  
   \* Serializes a standard MAVLink 2.0 frame  
   \*/  
  serializeMessage(msgName, values \= {}, options \= {}) {  
    const spec \= MAVLINK\_MSG\_SPECS\[msgName\];  
    if (\!spec) throw new Error(\`Unknown MAVLink message: \${msgName}\`);

    const seq \= options.seq \!== undefined ? options.seq : this.getNextSequence();  
    const sysId \= options.sysId \!== undefined ? options.sysId : this.sysId;  
    const compId \= options.compId \!== undefined ? options.compId : this.compId;  
    const incompatFlags \= options.incompatFlags \|\| 0x00;  
    const compatFlags \= options.compatFlags \|\| 0x00;

    // Pack raw payload  
    let rawPayload \= \[\];  
    if (msgName \=== 'HEARTBEAT') {  
      const customMode \= values.custom\_mode \|\| 0;  
      const type \= values.type \|\| 2; // Quadrotor  
      const autopilot \= values.autopilot \|\| 3; // ArduPilot  
      const baseMode \= values.base\_mode \|\| 0x81; // Armed / Guided  
      const systemStatus \= values.system\_status \|\| 4; // Active  
      const mavVersion \= values.mavlink\_version \|\| 3;  
        
      rawPayload.push(  
        customMode & 0xFF, (customMode \>\> 8\) & 0xFF, (customMode \>\> 16\) & 0xFF, (customMode \>\> 24\) & 0xFF,  
        type, autopilot, baseMode, systemStatus, mavVersion  
      );  
    } else if (msgName \=== 'ATTITUDE') {  
      const tBoot \= values.time\_boot\_ms \|\| 1000;  
      rawPayload.push(tBoot & 0xFF, (tBoot \>\> 8\) & 0xFF, (tBoot \>\> 16\) & 0xFF, (tBoot \>\> 24\) & 0xFF);  
      const floats \= \[values.roll \|\| 0.0, values.pitch \|\| 0.0, values.yaw \|\| 0.0,   
                      values.rollspeed \|\| 0.0, values.pitchspeed \|\| 0.0, values.yawspeed \|\| 0.0\];  
      for (let f of floats) {  
        const buf \= new ArrayBuffer(4);  
        new Float32Array(buf)\[0\] \= f;  
        const u8 \= new Uint8Array(buf);  
        rawPayload.push(...u8);  
      }  
    } else {  
      rawPayload \= \[0x00, 0x00, 0x00, 0x00\];  
    }

    // Zero-truncation for MAVLink 2.0  
    const payload \= options.noTruncation ? rawPayload : MAVLinkEngine.truncatePayload(rawPayload);  
    const len \= payload.length;

    // 10-byte MAVLink 2.0 Header  
    const header \= \[  
      MAVLINK\_V2\_MAGIC,  
      len,  
      incompatFlags,  
      compatFlags,  
      seq,  
      sysId,  
      compId,  
      spec.msgId & 0xFF,  
      (spec.msgId \>\> 8\) & 0xFF,  
      (spec.msgId \>\> 16\) & 0xFF  
    \];

    // Compute Checksum over header (excluding magic) \+ payload \+ CRC-EXTRA  
    const checksumBytes \= header.slice(1).concat(payload);  
    const crc \= MAVLinkEngine.calculateChecksum(checksumBytes, spec.crcExtra);

    const frame \= \[  
      ...header,  
      ...payload,  
      crc & 0xFF,  
      (crc \>\> 8\) & 0xFF  
    \];

    return {  
      msgName: msgName,  
      msgId: spec.msgId,  
      seq: seq,  
      sysId: sysId,  
      compId: compId,  
      payloadLength: len,  
      truncatedBytes: rawPayload.length \- len,  
      crc: crc,  
      wireBytes: frame,  
      hexDump: frame.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }

  /\*\*  
   \* Deserializes and validates a MAVLink 2.0 frame  
   \*/  
  deserializeMessage(bytes) {  
    if (bytes.length \< 12\) throw new Error('Frame too short (\< 12 bytes)');  
    if (bytes\[0\] \!== MAVLINK\_V2\_MAGIC) throw new Error(\`Invalid magic byte: 0x\${bytes\[0\].toString(16)}\`);

    const len \= bytes\[1\];  
    const incompat \= bytes\[2\];  
    const compat \= bytes\[3\];  
    const seq \= bytes\[4\];  
    const sysId \= bytes\[5\];  
    const compId \= bytes\[6\];  
    const msgId \= bytes\[7\] \| (bytes\[8\] \<\< 8\) \| (bytes\[9\] \<\< 16);

    const payload \= bytes.slice(10, 10 \+ len);  
    const receivedCrc \= bytes\[10 \+ len\] \| (bytes\[10 \+ len \+ 1\] \<\< 8);

    // Find spec by msgId  
    let spec \= null;  
    let name \= 'UNKNOWN';  
    for (const \[k, v\] of Object.entries(MAVLINK\_MSG\_SPECS)) {  
      if (v.msgId \=== msgId) {  
        spec \= v;  
        name \= k;  
        break;  
      }  
    }

    let isValid \= false;  
    if (spec) {  
      const checksumBytes \= bytes.slice(1, 10 \+ len);  
      const computedCrc \= MAVLinkEngine.calculateChecksum(checksumBytes, spec.crcExtra);  
      isValid \= (computedCrc \=== receivedCrc);  
    }

    return {  
      isValid: isValid,  
      msgName: name,  
      msgId: msgId,  
      seq: seq,  
      sysId: sysId,  
      compId: compId,  
      payloadLength: len,  
      payload: payload  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= { MAVLinkEngine, MAVLINK\_MSG\_SPECS, MAVLINK\_V2\_MAGIC, MAVLINK\_V1\_MAGIC };  
}  
