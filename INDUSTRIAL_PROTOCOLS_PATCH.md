# Patch Set Summary: Industrial Protocol Support (Modbus, CIP, Ethernet/IP)

## Overview

This patch set ports the changes from [google/gopacket PR #758](https://github.com/google/gopacket/pull/758) to add support for industrial automation protocols: **Modbus**, **CIP** (Common Industrial Protocol), and **ENIP** (Ethernet/IP).

## New Protocol Implementations

### 1. CIP (Common Industrial Protocol)
**File**: `layers/cip.go`

- **Purpose**: Application-layer protocol used in industrial automation
- **Key Features**:
  - Service code definitions (GetAttributesAll, SetAttributesAll, GetAttributeSingle, SetAttributeSingle, MultipleServicePacket)
  - Status code handling (Success, ConnectionFailure, ResourceUnavailable, etc.)
  - Path segment parsing for Class ID, Instance ID, and Attribute ID
  - Request/Response differentiation
  - Full decoder implementation following gopacket patterns

### 2. ENIP (Ethernet/IP)
**File**: `layers/enip.go`

- **Purpose**: Ethernet encapsulation protocol for CIP
- **Key Features**:
  - 24-byte header parsing
  - Command codes (NOP, ListServices, RegisterSession, UnregisterSession, SendRRData, SendUnitData, etc.)
  - Status code handling
  - Session management fields (SessionHandle, SenderContext)
  - Automatic CIP payload decoding for SendRRData and SendUnitData commands
  - Serialization support
  - Little-endian encoding (as per ENIP specification)

### 3. ModbusTCP
**File**: `layers/modbustcp.go` (already existed)

- **Purpose**: Industrial SCADA protocol over TCP
- **Key Features**:
  - MBAP (Modbus Application Protocol) header decoding
  - Transaction identifier, protocol identifier, length, and unit identifier fields
  - PDU (Protocol Data Unit) payload extraction

## Modified Files

### Core Layer Registration

#### `layers/layertypes.go`
- Added `LayerTypeENIP` (ID: 151)
- Added `LayerTypeCIP` (ID: 152)
- Existing `LayerTypeModbusTCP` (ID: 141)

```go
LayerTypeENIP = gopacket.RegisterLayerType(151, gopacket.LayerTypeMetadata{
    Name: "ENIP", 
    Decoder: gopacket.DecodeFunc(decodeENIP)
})
LayerTypeCIP = gopacket.RegisterLayerType(152, gopacket.LayerTypeMetadata{
    Name: "CIP", 
    Decoder: gopacket.DecodeFunc(decodeCIP)
})
```

#### `layers/ports.go`
Added port mappings for automatic protocol detection:

**TCP Ports**:
- Port 502 → `LayerTypeModbusTCP` (Modbus over TCP)
- Port 2222 → `LayerTypeENIP` (EtherNet/IP-1)
- Port 44818 → `LayerTypeENIP` (EtherNet/IP-2)

**UDP Ports**:
- Port 2222 → `LayerTypeENIP` (EtherNet/IP-1)
- Port 44818 → `LayerTypeENIP` (EtherNet/IP-2)

#### `layers/enums.go`
- Added missing `errors` import
- Added `EthernetTypeERSPAN` constant (0x88be)
- Added `EthernetTypeRaw` constant (0xFFFF)

## Protocol Specifications

### Port Numbers
| Protocol | TCP Port | UDP Port | Description |
|----------|----------|----------|-------------|
| ModbusTCP | 502 | - | Modbus over TCP/IP |
| ENIP | 2222 | 2222 | EtherNet/IP-1 (standard) |
| ENIP | 44818 | 44818 | EtherNet/IP-2 (alternate) |

### Layer Hierarchy
```
Ethernet
  └── IPv4/IPv6
      └── TCP/UDP
          ├── ModbusTCP (port 502)
          │   └── Payload (Modbus PDU)
          └── ENIP (ports 2222, 44818)
              └── CIP (for SendRRData/SendUnitData commands)
                  └── Payload
```

## Credits

- Original implementation by @traetox in [PR #408](https://github.com/google/gopacket/pull/408)
- Reimplemented by @dreadl0ck in [PR #758](https://github.com/google/gopacket/pull/758)
- Ported to gopacket-community repository

## References

- **CIP**: ODVA Common Industrial Protocol specification
- **ENIP**: ODVA Ethernet/IP specification
- **ModbusTCP**: Modbus TCP/IP specification (port 502)
- **IANA Ports**: Port 2222 (EtherNet-IP-1), Port 44818 (EtherNet-IP-2)

