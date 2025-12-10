# ZeroTierOne P2P Networking Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Protocol Versions and Constants](#protocol-versions-and-constants)
3. [Packet Format and Structure](#packet-format-and-structure)
4. [Protocol Verbs (Message Types)](#protocol-verbs-message-types)
5. [P2P Connection Establishment](#p2p-connection-establishment)
6. [NAT Traversal and Hole Punching](#nat-traversal-and-hole-punching)
7. [Relay Mechanism Through Root Servers](#relay-mechanism-through-root-servers)
8. [Complete Packet Handling Flow](#complete-packet-handling-flow)
9. [Encryption and Security](#encryption-and-security)
10. [Path Selection and Management](#path-selection-and-management)
11. [Multipath Bonding](#multipath-bonding)
12. [Threading and Concurrency](#threading-and-concurrency)
13. [Network Layers (VL1 and VL2)](#network-layers-vl1-and-vl2)
14. [End-to-End Communication Flow](#end-to-end-communication-flow)

---

## Architecture Overview

ZeroTier implements a peer-to-peer virtual networking system with server-assisted NAT traversal. The architecture consists of:

- **VL1 (Virtual Layer 1)**: The encrypted P2P transport layer
- **VL2 (Virtual Layer 2)**: The virtual Ethernet network layer
- **Root Servers**: Public infrastructure nodes for relaying and NAT traversal
- **Peers**: Network nodes that can communicate directly or via relay
- **Switch**: Core packet routing and handling component
- **Topology**: Maintains knowledge of peers and network structure

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Layer                       │
│                    (Virtual TAP Device)                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                           │  VL2: Virtual Layer 2           │
│                           ▼                                  │
│          ┌─────────────────────────────────┐                │
│          │    Network (Virtual Ethernet)   │                │
│          │  - MAC addressing                │                │
│          │  - Multicast                     │                │
│          │  - Bridge/Router logic           │                │
│          └─────────────────────────────────┘                │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┼───────────────────────────────────┐
│                          │   VL1: Virtual Layer 1            │
│                          ▼                                    │
│          ┌────────────────────────────────────┐             │
│          │        Switch Component            │             │
│          │  - Packet routing                  │             │
│          │  - QoS/AQM                         │             │
│          │  - Fragmentation                   │             │
│          └──────┬──────────────┬──────────────┘             │
│                 │              │                             │
│      ┌──────────▼────┐   ┌────▼─────────┐                  │
│      │  Peer Manager │   │  Topology    │                  │
│      │  - Path sel.  │   │  - Root info │                  │
│      │  - Bonding    │   │  - Peer DB   │                  │
│      └───────────────┘   └──────────────┘                  │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────┼──────────────────────────────────┐
│                           │  Physical Network                 │
│                           ▼                                   │
│   ┌────────────────────────────────────────────────┐         │
│   │         UDP/TCP Transport                      │         │
│   │   - Direct P2P paths                           │         │
│   │   - Relayed paths (via root servers)           │         │
│   │   - Multiple simultaneous paths (bonding)      │         │
│   └────────────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```


---

## Protocol Versions and Constants

### Current Protocol Version
- **Protocol Version**: 13 (ZT_PROTO_VERSION)
- **Minimum Supported**: Version 4 (ZT_PROTO_VERSION_MIN)
- **Maximum Hops**: 7 (protocol allows), 3 (pragmatic limit for ZT_RELAY_MAX_HOPS)

### Protocol Evolution
```
Version 1-2:  Initial versions (0.2.x - 0.4.5)
Version 3:    New crypto, multicast redesign (0.5.0 - 0.6.0)
Version 4:    New identity format (0.6.0 - 1.0.6)
Version 5:    Echo, in-band updates, clustering (1.1.0 - 1.1.5)
Version 6:    Config format revisions (1.1.5 - 1.1.10)
Version 7:    Trusted paths (1.1.10 - 1.1.17)
Version 8:    Multipart configs, tags, capabilities (1.1.17 - 1.2.0)
Version 9:    (1.2.0 - 1.2.14)
Version 10:   (1.4.0 - 1.4.6)
Version 11:   Multipath load balancing (1.4.7 - 1.4.8)
Version 12:   AES-GMAC-SIV (1.4.8 - 1.16.0)
Version 13:   Ephemeral keying, encrypted HELLO (1.16.0+)
```

### Key Constants
```
ZT_ADDRESS_LENGTH = 5 bytes (40-bit addresses)
ZT_DEFAULT_MTU = 2800 bytes
ZT_MAX_PACKET_FRAGMENTS = 7
ZT_DEFAULT_PHYSMTU = varies by network
ZT_SYMMETRIC_KEY_SIZE = 48 bytes
```

---

## Packet Format and Structure

### Wire Packet Format
```
Bytes 0-27: Fixed Header (minimum viable packet size)
===================================================

┌────────────────────────────────────────────────┐
│  0-7:   Packet ID / IV / Counter (64-bit)     │  8 bytes
│         - Random IV for crypto                 │
│         - Lower 3 bits: packet counter         │
├────────────────────────────────────────────────┤
│  8-12:  Destination Address (40-bit)          │  5 bytes
├────────────────────────────────────────────────┤
│  13-17: Source Address (40-bit)               │  5 bytes
├────────────────────────────────────────────────┤
│  18:    Flags/Cipher/Hops (8-bit)             │  1 byte
│         Bits: FFCCCHHH                         │
│         - FF: Flags (2 bits)                   │
│           0x80: Extended armor                 │
│           0x40: Fragmented                     │
│         - CCC: Cipher suite (3 bits)           │
│         - HHH: Hop count (3 bits, 0-7)         │
├────────────────────────────────────────────────┤
│  19-26: MAC or Trusted Path ID (64-bit)       │  8 bytes
│         - Authentication tag for encrypted     │
│         - Trusted path ID for trusted mode     │
└────────────────────────────────────────────────┘
         ↓ Begin encrypted/authenticated envelope
┌────────────────────────────────────────────────┐
│  27:    Verb and Encrypted Flags (8-bit)      │  1 byte
│         Bits: FFFVVVVV                         │
│         - FFF: Encrypted flags (3 bits)        │
│           0x80: Compressed (LZ4)               │
│         - VVVVV: Verb/opcode (5 bits, 0-31)    │
├────────────────────────────────────────────────┤
│  28+:   Verb-specific Payload                  │  Variable
│         (different for each verb type)         │
└────────────────────────────────────────────────┘

Total minimum: 28 bytes
Total maximum: ZT_PROTO_MAX_PACKET_LENGTH
```

### Fragment Format
When packets exceed MTU, they're fragmented:
```
┌────────────────────────────────────────────────┐
│  0-7:   Packet ID (same as original)          │  8 bytes
├────────────────────────────────────────────────┤
│  8-12:  Destination Address                    │  5 bytes
├────────────────────────────────────────────────┤
│  13:    Fragment Indicator (0xff)              │  1 byte
├────────────────────────────────────────────────┤
│  14:    Fragment Number                        │  1 byte
├────────────────────────────────────────────────┤
│  15:    Hop Count                              │  1 byte
├────────────────────────────────────────────────┤
│  16+:   Fragment Payload                       │  Variable
└────────────────────────────────────────────────┘
```


---

## Protocol Verbs (Message Types)

ZeroTier uses 32 possible verbs (5 bits) for different packet types:

### Core Protocol Verbs

| Verb Code | Name | Purpose |
|-----------|------|---------|
| 0x00 | NOP | No operation (ignored) |
| 0x01 | HELLO | Identity announcement and key exchange |
| 0x02 | ERROR | Error response |
| 0x03 | OK | Success response to request |
| 0x04 | WHOIS | Request peer identity |
| 0x05 | RENDEZVOUS | NAT traversal assistance |
| 0x06 | FRAME | Layer 2 frame transport |
| 0x07 | EXT_FRAME | Extended frame with additional fields |
| 0x08 | ECHO | Ping/latency measurement |
| 0x09 | MULTICAST_LIKE | Subscribe to multicast group |
| 0x0a | NETWORK_CREDENTIALS | Push network credentials |
| 0x0b | NETWORK_CONFIG_REQUEST | Request network configuration |
| 0x0c | NETWORK_CONFIG | Network configuration delivery |
| 0x0d | MULTICAST_GATHER | Gather multicast subscribers |
| 0x0e | MULTICAST_FRAME | Multicast frame delivery |
| 0x10 | PUSH_DIRECT_PATHS | Share direct path information |
| 0x12 | ACK | Acknowledgment |
| 0x13 | QOS_MEASUREMENT | QoS metrics exchange |
| 0x14 | USER_MESSAGE | User-defined message |
| 0x15 | REMOTE_TRACE | Remote debugging/trace |
| 0x16 | PATH_NEGOTIATION_REQUEST | Negotiate path parameters |

### Verb Flow Patterns

#### Request-Response Pattern
```
Peer A                           Peer B
  │                                 │
  ├──────── WHOIS ─────────────────>│
  │        (request identity)       │
  │                                 │
  │<────────── OK ──────────────────┤
  │        (identity data)          │
```

#### Fire-and-Forget Pattern
```
Peer A                           Peer B
  │                                 │
  ├──────── FRAME ─────────────────>│
  │       (Ethernet frame)          │
  │                                 │
  │        (no response)            │
```

#### Multicast Announcement
```
Node                             Network
  │                                 │
  ├──── MULTICAST_LIKE ────────────>│
  │    (subscribe to group)         │
  │                                 │
  ├─── MULTICAST_GATHER ───────────>│
  │   (request subscribers)         │
  │                                 │
  │<──────── OK ────────────────────┤
  │   (subscriber list)             │
```


---

## P2P Connection Establishment

### Initial Connection Flow

```
Node A (10.0.1.5:9993)                                Root Server                              Node B (192.168.1.10:9993)
behind NAT A                                      (public IP)                                    behind NAT B
      │                                                 │                                              │
      │                                                 │                                              │
      ├────── 1. HELLO (unencrypted) ──────────────────>│                                              │
      │       Contains:                                 │                                              │
      │       - Protocol version                        │                                              │
      │       - Node identity (public key)              │                                              │
      │       - Timestamp                                │                                              │
      │                                                 │                                              │
      │<─────────── 2. OK (HELLO) ──────────────────────┤                                              │
      │       Contains:                                 │                                              │
      │       - Server's protocol version               │                                              │
      │       - Timestamp echo                          │                                              │
      │       - World definition (root server list)     │                                              │
      │                                                 │                                              │
      │  (Path to root server established)              │                                              │
      │  (Node A now appears at NAT A's public IP)      │                                              │
      │                                                 │                                              │
      │                                                 │<─────── 3. HELLO (from Node B) ──────────────┤
      │                                                 │       (Similar exchange)                     │
      │                                                 │                                              │
      │                                                 │────────── 4. OK (HELLO) ────────────────────>│
      │                                                 │                                              │
      │                                                 │  (Path to root established for Node B)       │
      │                                                 │  (Node B now appears at NAT B's public IP)   │
      │                                                 │                                              │
      │────── 5. WHOIS (Node B's addr) ─────────────────>│                                              │
      │       "How do I reach Node B?"                  │                                              │
      │                                                 │                                              │
      │<────────── 6. OK (WHOIS) ────────────────────────┤                                              │
      │       Contains Node B's identity                │                                              │
      │       and last known paths                      │                                              │
      │                                                 │                                              │
      │────── 7. RENDEZVOUS request ─────────────────────>│                                              │
      │       "Help me connect to Node B"               │                                              │
      │                                                 │                                              │
      │                                                 │────── 8. RENDEZVOUS ────────────────────────>│
      │                                                 │       "Node A wants to reach you"            │
      │                                                 │       Contains Node A's public endpoint       │
      │                                                 │                                              │
      │<────────────────── 9. RENDEZVOUS ───────────────┤                                              │
      │       "Node B endpoint info"                    │                                              │
      │       Contains Node B's public endpoint         │                                              │
      │                                                 │                                              │
```

### Direct Path Establishment (Hole Punching)

```
Node A (NAT A: 1.2.3.4:12345)                                                         Node B (NAT B: 5.6.7.8:54321)
      │                                                                                          │
      │  (Both nodes have each other's public endpoints from RENDEZVOUS)                        │
      │                                                                                          │
      ├────── 10. HELLO (direct attempt) ──────────────────────X (blocked by NAT B) ────────────┤
      │        Sent to 5.6.7.8:54321                                                            │
      │        Creates outbound NAT mapping:                                                    │
      │        NAT A: internal 10.0.1.5:9993 → external 1.2.3.4:12345                          │
      │                                                                                          │
      │                                                                                          │
      │<─ ─ ─ ─ 11. HELLO (direct attempt) ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ─┤
      │        Sent to 1.2.3.4:12345                                                            │
      │        NAT A now has hole punched!                                                       │
      │        Creates outbound NAT mapping:                                                     │
      │        NAT B: internal 192.168.1.10:9993 → external 5.6.7.8:54321                       │
      │                                                                                          │
      ├────────────── 12. OK (HELLO) ──────────────────────────────────────────────────────────>│
      │        Direct path SUCCESS!                                                             │
      │        Node A → NAT A → Internet → NAT B → Node B                                       │
      │                                                                                          │
      │<───────────────── 13. OK (HELLO) ────────────────────────────────────────────────────────┤
      │        Direct path CONFIRMED!                                                           │
      │        Node B → NAT B → Internet → NAT A → Node A                                       │
      │                                                                                          │
      │  ═══════════════════  P2P CONNECTION ESTABLISHED  ═══════════════════                   │
      │                                                                                          │
      ├──────────────────── 14. FRAME (data) ──────────────────────────────────────────────────>│
      │                                                                                          │
      │<─────────────────────── 15. FRAME (data) ───────────────────────────────────────────────┤
      │                                                                                          │
```

### Path Lifecycle

```
Path State Machine:
                    
    [Created]
        │
        ├──> Send packets
        │
        ▼
   [Attempting]
        │
        ├──> Receive OK ──────> [Alive]
        │                          │
        │                          ├──> Regular traffic
        │                          │    
        ├──> Timeout ──────────> [Dead]
        │                          │
        │                          ├──> Retry after delay
        │                          │
        └──────────────────────────┘
```


---

## NAT Traversal and Hole Punching

### NAT Types and Traversal

ZeroTier handles multiple NAT types:

```
NAT Type Matrix:

                    │  Full Cone  │  Restricted  │  Port Restricted  │  Symmetric
────────────────────┼─────────────┼──────────────┼───────────────────┼─────────────
Full Cone           │    Direct   │    Direct    │      Direct       │   Direct
Restricted          │    Direct   │    Direct    │      Direct       │   RENDEZVOUS
Port Restricted     │    Direct   │    Direct    │      RENDEZVOUS   │   Relay
Symmetric           │    Direct   │  RENDEZVOUS  │      Relay        │   Relay

Direct = Hole punching succeeds easily
RENDEZVOUS = Requires coordinated timing
Relay = Must use relay server
```

### Hole Punching Technique

#### Birthday Paradox Attack on NATs

ZeroTier uses simultaneous packet sending from both sides:

```
Step 1: Both nodes learn each other's public endpoints via RENDEZVOUS

Node A (Behind NAT A)                NAT A                    Internet                    NAT B                Node B (Behind NAT B)
      │                                │                           │                           │                        │
      │  Internal: 10.0.1.5:9993       │  External: 1.2.3.4:X     │                           │  External: 5.6.7.8:Y  │  Internal: 192.168.1.10:9993
      │                                │                           │                           │                        │

Step 2: Simultaneous packet sending creates temporary holes

      ├─── Packet to 5.6.7.8:Y ──────>│                           │                           │                        │
      │                                ├─── Creates mapping ──────>│                           │                        │
      │                                │   (1.2.3.4:12345)         │                           │                        │
      │                                │           ↓               │                           │                        │
      │                                │      [NAT Hole]           │                           │                        │
      │                                │                           │    Packet to 1.2.3.4:? ───┤← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
      │                                │                           │<─── Creates mapping ──────┤                        │
      │                                │                           │     (5.6.7.8:54321)       │                        │
      │                                │                           │              ↓            │                        │
      │                                │                           │         [NAT Hole]        │                        │

Step 3: Packets can now traverse both NATs

      │                                │                           │                           │                        │
      ├────────── Data Packet ─────────>│                           │                           │                        │
      │                                ├────────────────────────────>│                           │                        │
      │                                │         (using hole)        ├───────────────────────────>│                        │
      │                                │                           │                           ├────────────────────────>│
      │                                │                           │                           │      Data arrives!      │
```

### RENDEZVOUS Protocol

The RENDEZVOUS packet facilitates NAT traversal:

```
RENDEZVOUS Packet Structure:
┌────────────────────────────────────┐
│ Flags (1 byte)                     │
├────────────────────────────────────┤
│ ZeroTier Address (5 bytes)         │  ← Target peer address
├────────────────────────────────────┤
│ Port (2 bytes)                     │  ← Target peer port
├────────────────────────────────────┤
│ Address Length (1 byte)            │  
├────────────────────────────────────┤
│ IP Address (4 or 16 bytes)         │  ← Target peer IP
└────────────────────────────────────┘

Root Server Role:
┌─────────────────────────────────────────────────────────┐
│ 1. Receive connection request from Node A               │
│ 2. Query own peer database for Node B's endpoints      │
│ 3. Send RENDEZVOUS to Node A with Node B's info        │
│ 4. Send RENDEZVOUS to Node B with Node A's info        │
│ 5. Both nodes attempt simultaneous connection           │
└─────────────────────────────────────────────────────────┘
```

### Retry and Backoff Strategy

```
Attempt Timeline:

T=0ms     ├─ First HELLO attempt (both directions)
          │
T=100ms   ├─ Retry if no response
          │
T=300ms   ├─ Retry with exponential backoff
          │
T=700ms   ├─ Another retry
          │
T=1500ms  ├─ Final direct attempt
          │
          ▼
   If all direct attempts fail:
          │
          ├─ Fall back to relay via root server
          │
          ▼
   Continue attempting direct path in background
```


---

## Relay Mechanism Through Root Servers

### Relay Architecture

When direct P2P connection fails, ZeroTier uses relay through root servers:

```
Node A                           Root Server                        Node B
(Behind Symmetric NAT)         (Public Internet)              (Behind Symmetric NAT)
      │                                │                                 │
      │                                │                                 │
      ├──── 1. Packet to Node B ──────>│                                 │
      │     (wrapped in relay frame)   │                                 │
      │                                │                                 │
      │                                ├──── 2. Forward packet ──────────>│
      │                                │    (unwrap and rewrite src)     │
      │                                │                                 │
      │                                │<──── 3. Response packet ─────────┤
      │                                │                                 │
      │<──── 4. Forward response ──────┤                                 │
      │                                │                                 │
      │                                │                                 │
      │  ═════════════════════════════════════════════════════════════  │
      │        All traffic flows through root server                    │
      │  ═════════════════════════════════════════════════════════════  │
```

### Relay Packet Flow

```
Original Packet from Node A:
┌──────────────────────────────────────┐
│ ZT Header (destination: Node B)      │
│ Encrypted payload                    │
└──────────────────────────────────────┘
            │
            ▼
UDP encapsulation to Root Server:
┌──────────────────────────────────────┐
│ UDP Header (to Root Server)          │
│   Source: Node A's public endpoint   │
│   Dest: Root Server's endpoint       │
├──────────────────────────────────────┤
│ ZT Header (destination: Node B)      │
│ Encrypted payload                    │
└──────────────────────────────────────┘
            │
            ▼ (Root server receives)
            │
            ▼ (Root server checks destination)
            │
            ▼ (Looks up Node B's path)
            │
            ▼
UDP encapsulation to Node B:
┌──────────────────────────────────────┐
│ UDP Header (to Node B)               │
│   Source: Root Server's endpoint     │
│   Dest: Node B's public endpoint     │
├──────────────────────────────────────┤
│ ZT Header (destination: Node B)      │
│ Encrypted payload (unchanged)        │
└──────────────────────────────────────┘
```

### Hop Count and TTL

```
Hop Count Tracking:

Original packet from Node A:
  Flags/Cipher/Hops = 0bFF000001  (Hops=1)
                         ↑↑↑
                         HHH = hop count (0-7)

After relay through Root Server:
  Flags/Cipher/Hops = 0bFF000010  (Hops=2)
                                   ↑
                         Incremented by root server

Maximum hops = 7 (protocol limit)
Practical limit = 3 (ZT_RELAY_MAX_HOPS)

Packets with hops > limit are dropped to prevent loops.
```

### Root Server Selection

```
Root Server Priority:

1. Lowest Latency
   ├─ Measure RTT with ECHO packets
   └─ Update every 60 seconds

2. Alive Status  
   ├─ Must respond to HELLO
   └─ Must have recent activity

3. Geographic Proximity
   ├─ Prefer same region
   └─ Use IP geolocation hints

Selection Algorithm:
┌────────────────────────────────────┐
│ FOR each root server:              │
│   IF alive AND latency known:      │
│     score = 1000 / (latency_ms+1)  │
│   ELSE:                            │
│     score = 0                      │
│ END FOR                            │
│ SELECT root with highest score     │
└────────────────────────────────────┘
```

### Relay Path Upgrade

ZeroTier continuously attempts to establish direct paths even when using relay:

```
Timeline with Active Relay:

T=0s      Using relay path
          │
          ├──── Background: Send HELLO to direct endpoint
          │
T=1s      │
          ├──── Background: Retry direct HELLO
          │
T=2s      │
          ├──── Direct path SUCCESS!
          │
          ├──── Switch to direct path
          │     (relay path kept as backup)
          │
T=3s+     Using direct path
          │
          ├──── If direct path fails:
          └──── Fall back to relay immediately
```


---

## Complete Packet Handling Flow

### Incoming Packet Processing Pipeline

```
Physical Network (UDP)
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│  1. PacketMultiplexer / PHY Layer                           │
│     - Receive raw UDP packet                                │
│     - Extract socket and remote address                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Switch::onRemotePacket()                                │
│     - Initial packet validation                             │
│     - Check minimum packet size (28 bytes)                  │
│     - Extract source/dest addresses                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Fragment Reassembly (if fragmented)                     │
│     - Check fragmented flag (0x40)                          │
│     - Collect all fragments by packet ID                    │
│     - Reassemble when complete                              │
│     - Timeout incomplete fragments after 5 seconds          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  4. IncomingPacket::tryDecode()                             │
│     - Check cipher suite                                    │
│     - Handle trusted paths (NO_CRYPTO mode)                 │
│     - Handle unencrypted HELLO                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Peer Lookup                                             │
│     - Get peer from Topology                                │
│     - Create new peer if needed (on HELLO)                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Packet::dearmor() - Authentication & Decryption         │
│     - Verify MAC (Poly1305 or AES-GMAC-SIV)                │
│     - Decrypt payload (Salsa20/12 or AES-CTR)              │
│     - Handle ephemeral keys (extended armor)                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Decompression (if compressed flag set)                  │
│     - LZ4 decompression                                     │
│     - Verify decompressed size                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  8. Verb Dispatch                                           │
│     - Extract verb (lower 5 bits of byte 27)               │
│     - Route to appropriate handler                          │
│       • HELLO → _doHELLO()                                  │
│       • OK → _doOK()                                        │
│       • FRAME → _doFRAME()                                  │
│       • RENDEZVOUS → _doRENDEZVOUS()                        │
│       • etc.                                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  9. Verb-Specific Processing                                │
│     Example for FRAME:                                      │
│     - Extract network ID                                    │
│     - Verify network membership                             │
│     - Check credentials (tags, capabilities)                │
│     - Extract Ethernet frame                                │
│     - Pass to Network layer                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  10. Network Layer Processing                               │
│      - MAC address learning                                 │
│      - Multicast handling                                   │
│      - Apply network rules                                  │
│      - QoS classification                                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  11. TAP Device Injection                                   │
│      - Write Ethernet frame to virtual interface            │
│      - Frame appears on OS network stack                    │
└─────────────────────────────────────────────────────────────┘
                       │
                       ▼
              Application Layer
```

### Outgoing Packet Processing Pipeline

```
Application Layer
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│  1. TAP Device Read                                         │
│     - Read Ethernet frame from virtual interface            │
│     - Extract dest MAC, source MAC, EtherType              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Network::processIncomingPacket()                        │
│     - Match network ID                                      │
│     - Verify source MAC authorization                       │
│     - Apply egress rules                                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Destination Resolution                                  │
│     - Unicast: Lookup dest MAC in ARP/ND table             │
│     - Multicast: Lookup group subscribers                   │
│     - Broadcast: Send to all network members                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Switch::onLocalEthernet()                               │
│     - Create ZT packet with FRAME or EXT_FRAME verb        │
│     - Add network ID and frame payload                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  5. QoS Classification                                      │
│     - Match against network rules                           │
│     - Assign to QoS bucket (0-3)                           │
│     - Extract flow ID (5-tuple hash)                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Compression (optional)                                  │
│     - LZ4 compression for large frames                      │
│     - Set compressed flag if used                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Packet::armor() - Encryption & Authentication           │
│     - Generate random packet ID / IV                        │
│     - Encrypt with peer's shared key                        │
│     - Generate MAC (authentication tag)                     │
│     - Apply ephemeral key (if configured)                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  8. Path Selection                                          │
│     - Query Peer for best path                              │
│     - Consider: latency, packet loss, QoS                  │
│     - Apply bonding policy if multipath                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  9. AQM (Active Queue Management)                           │
│     - Enqueue in per-network, per-flow queue               │
│     - Apply CoDel algorithm                                 │
│     - Drop packets if queue too deep                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  10. Fragmentation (if needed)                              │
│      - Check if packet > path MTU                           │
│      - Split into fragments                                 │
│      - Mark with fragment flag and numbers                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  11. UDP Transmission                                       │
│      - Send via PHY layer                                   │
│      - Track for ACK if needed                              │
│      - Update path statistics                               │
└─────────────────────────────────────────────────────────────┘
                       │
                       ▼
          Physical Network (UDP)
```


---

## Encryption and Security

### Cipher Suites

ZeroTier supports multiple cipher suites for different security requirements:

| Suite ID | Name | Description | Use Case |
|----------|------|-------------|----------|
| 0 | C25519_POLY1305_NONE | Auth only, no encryption | HELLO packets |
| 1 | C25519_POLY1305_SALSA2012 | Legacy encryption | Older clients |
| 2 | NO_CRYPTO_TRUSTED_PATH | No crypto | Trusted LANs |
| 3 | AES_GMAC_SIV | Modern AES encryption | Current default |

### Cryptographic Primitives

```
Key Agreement:
  Algorithm: Curve25519 ECDH
  Key Size: 256 bits
  Purpose: Establish shared secret between peers

Symmetric Encryption (Suite 1):
  Cipher: Salsa20/12
  Key: 256 bits (derived from ECDH)
  IV: 64-bit packet ID
  
Symmetric Encryption (Suite 3):
  Cipher: AES-256 in GMAC-SIV mode
  Keys: Two 256-bit keys (K0, K1)
  Derivation: KBKDF-HMAC-SHA384
  
Authentication:
  MAC: Poly1305 (Suite 1) or AES-GMAC (Suite 3)
  Tag Size: 64 bits
  Purpose: Packet integrity and authentication
```

### Key Derivation Flow

```
Node A Identity                               Node B Identity
    │                                             │
    ├─ Private Key A (secret)                     ├─ Private Key B (secret)
    └─ Public Key A ──────────────────────────────┘─ Public Key B
              │                                             │
              └─────────────┬──────────────────────────────┘
                            │
                    Curve25519 ECDH
                            │
                            ▼
                ┌───────────────────────┐
                │  Shared Secret (256b) │
                └───────────┬───────────┘
                            │
                            ├─────────────────────┐
                            │                     │
                            ▼                     ▼
             ┌──────────────────────┐  ┌──────────────────────┐
             │ KBKDF-HMAC-SHA384    │  │ KBKDF-HMAC-SHA384    │
             │ Label: '0'           │  │ Label: '1'           │
             └──────────┬───────────┘  └──────────┬───────────┘
                        │                         │
                        ▼                         ▼
            ┌─────────────────────┐   ┌─────────────────────┐
            │   AES Key 0 (K0)    │   │   AES Key 1 (K1)    │
            │   256 bits          │   │   256 bits          │
            └─────────────────────┘   └─────────────────────┘
                        │                         │
                        └────────────┬────────────┘
                                     │
                        Used for AES-GMAC-SIV encryption
```

### Packet Encryption Process (AES-GMAC-SIV)

```
Plaintext Packet:
┌────────────────────────────────────────┐
│ Verb (1 byte)                          │
│ Payload (variable)                     │
└────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────┐
│ Step 1: Generate Packet ID/IV          │
│   - Random 64-bit value                │
│   - Used as crypto IV                  │
│   - Lower 3 bits = counter             │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ Step 2: Build AAD (Additional Auth Data)│
│   - Packet ID                          │
│   - Source Address                     │
│   - Destination Address                │
│   - Flags/Cipher/Hops                  │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ Step 3: AES-GMAC-SIV Encrypt           │
│   Input: Plaintext + AAD               │
│   Keys: K0, K1                         │
│   Output: Ciphertext + MAC             │
└────────────────┬───────────────────────┘
                 │
                 ▼
Encrypted Packet:
┌────────────────────────────────────────┐
│ Packet ID / IV (8 bytes)               │
│ Destination Address (5 bytes)          │
│ Source Address (5 bytes)               │
│ Flags/Cipher/Hops (1 byte)             │
│ MAC (8 bytes)                          │
│ Encrypted Verb + Payload               │
└────────────────────────────────────────┘
```

### Extended Armor (Ephemeral Keying)

For enhanced security, protocol version 13+ uses ephemeral keys:

```
Standard Encryption:
  Packet encrypted with long-term shared key

Extended Armor (Flag 0x80):
  Step 1: Encrypt with long-term shared key
  Step 2: Generate ephemeral key (random)
  Step 3: Encrypt again with ephemeral key using header as IV
  Step 4: Append ephemeral key to packet
  
  This hides HELLO packets and adds forward secrecy.

Format with Extended Armor:
┌────────────────────────────────────────┐
│ Standard encrypted packet              │
├────────────────────────────────────────┤
│ Ephemeral Key (32 bytes)               │  ← Additional encryption layer
└────────────────────────────────────────┘
```

### Trusted Path Mode

For high-performance local networks:

```
Trusted Path Configuration:
  - No encryption
  - No authentication via crypto
  - Path identified by 64-bit Trusted Path ID
  - TPID stored in MAC field
  - Must match configured trusted path list
  
Security Model:
  ┌─────────────────────────────────────┐
  │ Physical security assumed            │
  │ (e.g., isolated data center LAN)    │
  └─────────────────────────────────────┘
              │
              ▼
  ┌─────────────────────────────────────┐
  │ Switch checks TPID against whitelist│
  └─────────────────────────────────────┘
              │
              ├─ Match: Accept packet
              └─ No match: Drop packet

Performance: ~50% lower CPU usage vs. encrypted
```


---

## Path Selection and Management

### Path Quality Metrics

Each path maintains multiple quality metrics:

```
Path Metrics:
┌─────────────────────────────────────────────┐
│ Metric              │ Unit    │ Purpose     │
├─────────────────────┼─────────┼─────────────┤
│ Latency Mean        │ ms      │ Avg RTT     │
│ Latency Variance    │ ms²     │ Jitter      │
│ Packet Loss Ratio   │ 0.0-1.0 │ Reliability │
│ Packet Error Ratio  │ 0.0-1.0 │ Corruption  │
│ Last In             │ ms      │ RX activity │
│ Last Out            │ ms      │ TX activity │
│ MTU                 │ bytes   │ Max packet  │
└─────────────────────────────────────────────┘

Quality Score Calculation:
  score = 1000 / (1 + latency_mean + 10*latency_variance + 
                   1000*packet_loss + 1000*packet_error)
```

### Path Lifecycle Management

```
Path State Transitions:

    [New]
      │
      ├─ Send initial HELLO
      │
      ▼
  [Attempting] ──────────────┐
      │                      │
      ├─ Receive OK          │ Timeout (10s)
      │                      │
      ▼                      ▼
   [Alive] ────────────> [Dead]
      │                      │
      │ No activity          │ Retry after
      │ (60s)                │ cool-down
      │                      │
      └──────────────────────┘

Path Expiration:
  - Dead paths removed after 120 seconds
  - Alive paths maintained indefinitely
  - Periodic ECHO keeps paths alive
```

### Multi-Path Scenario

```
Peer-to-Peer with Multiple Paths:

Node A                                    Node B
  │                                          │
  ├─ Path 1: WiFi (192.168.1.5)            ─┤
  │    Latency: 5ms                          │
  │    Quality: High                         │
  │                                          │
  ├─ Path 2: LTE (10.20.30.40)             ─┤
  │    Latency: 50ms                         │
  │    Quality: Medium                       │
  │                                          │
  ├─ Path 3: Relay (via root)              ─┤
  │    Latency: 80ms                         │
  │    Quality: Low (fallback)               │
  │                                          │
  
Selection Algorithm (without bonding):
  1. Filter: Only alive paths
  2. Sort: By quality score (descending)
  3. Select: Highest quality path
  4. Fallback: Next best if current fails

With bonding enabled, see next section.
```

### Direct Path Push

Nodes can advertise their endpoints to peers:

```
PUSH_DIRECT_PATHS Verb:
┌──────────────────────────────────────────┐
│ Flags (1 byte)                           │
│   0x01: Forget old paths                 │
│   0x02: Cluster redirect                 │
├──────────────────────────────────────────┤
│ Count (1 byte)                           │  ← Number of paths
├──────────────────────────────────────────┤
│ Path 1:                                  │
│   - Family (1 byte): IPv4/IPv6           │
│   - IP Address (4 or 16 bytes)           │
│   - Port (2 bytes)                       │
├──────────────────────────────────────────┤
│ Path 2: ...                              │
│ Path N: ...                              │
└──────────────────────────────────────────┘

Use Cases:
  - Dynamic IP changes
  - Multi-homed hosts
  - Load balancer behind node
  - Cluster member redirection
```

### Path Expiration and Cleanup

```
Housekeeping Cycle (every 30 seconds):

FOR each peer:
  FOR each path:
    age = now - path.lastIn
    
    IF age > 120s:
      Remove path (expired)
    ELSE IF age > 60s AND NOT path.alive:
      Mark as dead
    ELSE IF path.alive:
      IF age > 30s:
        Send ECHO (keepalive)
      END IF
    END IF
  END FOR
END FOR

Rate Limiting:
  - Max 1 ECHO per path per 10 seconds
  - Max 1 PUSH_DIRECT_PATHS per peer per 30 seconds
```


---

## Multipath Bonding

### Bonding Policies

ZeroTier supports multiple bonding policies for aggregating bandwidth and improving reliability:

```
┌────────────────────────────────────────────────────────────────┐
│ Policy              │ Behavior                                  │
├─────────────────────┼───────────────────────────────────────────┤
│ NONE (0)            │ Single path, no load balancing           │
├─────────────────────┼───────────────────────────────────────────┤
│ ACTIVE_BACKUP (1)   │ One active path, fail-over on failure    │
│                     │ - Fast switchover                          │
│                     │ - No bandwidth aggregation                 │
├─────────────────────┼───────────────────────────────────────────┤
│ BROADCAST (2)       │ Send on ALL paths simultaneously          │
│                     │ - Maximum reliability                      │
│                     │ - High bandwidth cost                      │
├─────────────────────┼───────────────────────────────────────────┤
│ BALANCE_RR (3)      │ Round-robin across all paths              │
│                     │ - Equal distribution                       │
│                     │ - May cause reordering                     │
├─────────────────────┼───────────────────────────────────────────┤
│ BALANCE_XOR (4)     │ Hash-based path selection                 │
│                     │ - Flow affinity (same flow → same path)   │
│                     │ - No reordering within flows               │
├─────────────────────┼───────────────────────────────────────────┤
│ BALANCE_AWARE (5)   │ Dynamic load balancing                    │
│                     │ - Quality-based distribution               │
│                     │ - Flow-aware                               │
│                     │ - Optimal performance                      │
└────────────────────────────────────────────────────────────────┘
```

### Bonding Architecture

```
                     Application Data
                            │
                            ▼
                    ┌───────────────┐
                    │  Switch/QoS   │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  Bond Logic   │
                    │  (Per-Peer)   │
                    └───────┬───────┘
                            │
                ┌───────────┼───────────┐
                │           │           │
                ▼           ▼           ▼
           ┌────────┐  ┌────────┐  ┌────────┐
           │ Path 1 │  │ Path 2 │  │ Path 3 │
           │ WiFi   │  │ LTE    │  │ Cable  │
           └────┬───┘  └────┬───┘  └────┬───┘
                │           │           │
                └───────────┴───────────┘
                            │
                     Physical Network
```

### Policy-Specific Behaviors

#### ACTIVE_BACKUP Flow
```
State: Primary path is Path 1 (WiFi)

Normal operation:
  ALL packets ──────> Path 1 (WiFi)

Path 1 fails:
  Detection time: <100ms (missed ECHO)
  Switch to Path 2 (LTE) ──────> Path 2

Path 1 recovers:
  Depends on reselection policy:
    - ALWAYS: Switch back immediately
    - BETTER: Switch if metrics improved
    - FAILURE: Stay on Path 2
    - OPTIMIZE: Switch if significantly better
```

#### BALANCE_RR Flow
```
Packet Distribution:

Packet 1 ─────────> Path 1
Packet 2 ─────────> Path 2
Packet 3 ─────────> Path 3
Packet 4 ─────────> Path 1  (wrap around)
Packet 5 ─────────> Path 2
...

Issue: Packet reordering possible if paths have different latencies
Solution: Used for protocols tolerant to reordering (or with reassembly)
```

#### BALANCE_XOR Flow
```
Flow-based hashing:

Flow ID = hash(src_ip, dst_ip, src_port, dst_port, protocol)

Flow 1 (hash % 3 = 0) ──────> Path 1
Flow 2 (hash % 3 = 1) ──────> Path 2
Flow 3 (hash % 3 = 2) ──────> Path 3
Flow 4 (hash % 3 = 0) ──────> Path 1

Benefits:
  + Packets in same flow always use same path
  + No reordering within flows
  + Load distributed across paths
```

#### BALANCE_AWARE Flow
```
Quality-Aware Distribution:

Path Quality Scores:
  Path 1 (WiFi):  score = 900  (low latency)
  Path 2 (LTE):   score = 600  (medium latency)
  Path 3 (Cable): score = 800  (good latency)

Total Score: 2300

Allocation ratios:
  Path 1: 900/2300 = 39% of flows
  Path 2: 600/2300 = 26% of flows
  Path 3: 800/2300 = 35% of flows

Dynamic Adjustment:
  - Monitor per-path performance
  - Reallocate flows every 10 seconds
  - Move flows from congested paths
  - Maintain flow affinity when possible
```

### Link Quality Monitoring

```
Per-Path Metrics Collection:

Every packet exchange updates:
┌──────────────────────────────────────┐
│ QoS Metrics:                         │
│   - Latency (RTT from ECHO)          │
│   - Packet Delivery Ratio            │
│   - Jitter (latency variance)        │
│   - Error rate                       │
│   - Available since last check       │
└──────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│ Quality Score Computation:           │
│                                      │
│ IF latency > MAX_LATENCY:            │
│   score = 0                          │
│ ELSE IF packetloss > MAX_LOSS:       │
│   score = 0                          │
│ ELSE:                                │
│   score = f(latency, jitter, loss)   │
│   Normalize to 0-1000 range          │
│ END IF                               │
└──────────────────────────────────────┘
             │
             ▼
   Used for path selection
```

### Bonding Configuration Example

```
{
  "settings": {
    "bond": {
      "policy": "balance-aware",
      "links": {
        "eth0": {
          "enabled": true,
          "speed": 1000000,  // 1 Gbps
          "mode": "primary"
        },
        "wlan0": {
          "enabled": true,
          "speed": 100000,   // 100 Mbps
          "mode": "spare",
          "failoverTo": "eth0"
        }
      }
    }
  }
}
```


---

## Threading and Concurrency

### Threading Model

ZeroTier uses a hybrid threading model with single-threaded core logic and multi-threaded I/O:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Application Threads                          │
│                  (External to ZeroTier)                         │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Thread Pointer (tPtr)                        │
│   - Passed through all API calls                               │
│   - Identifies calling thread context                          │
│   - Used for callbacks to application                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
        ┌───────────────────┴──────────────────┐
        │                                      │
        ▼                                      ▼
┌──────────────────────┐           ┌──────────────────────┐
│   Core Node Thread   │           │   I/O Threads        │
│   (Single-threaded)  │           │   (Multi-threaded)   │
├──────────────────────┤           ├──────────────────────┤
│ - Packet processing  │           │ - Socket I/O         │
│ - State management   │           │ - TAP device I/O     │
│ - Peer management    │           │ - HTTP API server    │
│ - Topology updates   │           │ - Background tasks   │
└──────────┬───────────┘           └──────────┬───────────┘
           │                                  │
           │                                  │
           └──────────────┬───────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Mutex Protection    │
              │   (Critical Sections) │
              └───────────────────────┘
```

### Synchronization Primitives

#### Mutex Implementation

```cpp
// Platform-Specific Mutex:

#ifdef __UNIX_LIKE__
// POSIX threads (pthread)
class Mutex {
  pthread_mutex_t _mh;
  
  void lock() {
    pthread_mutex_lock(&_mh);
  }
  
  void unlock() {
    pthread_mutex_unlock(&_mh);
  }
};
#endif

#ifdef __WINDOWS__
// Windows Critical Sections
class Mutex {
  CRITICAL_SECTION _cs;
  
  void lock() {
    EnterCriticalSection(&_cs);
  }
  
  void unlock() {
    LeaveCriticalSection(&_cs);
  }
};
#endif

// RAII Lock Guard
class Mutex::Lock {
  Mutex* _m;
  Lock(Mutex& m) : _m(&m) { _m->lock(); }
  ~Lock() { _m->unlock(); }
};
```

#### Atomic Operations

```cpp
// AtomicCounter for lock-free operations:

class AtomicCounter {
#ifdef __GNUC__
  // GCC/Clang built-ins
  int _v;
  
  int operator++() {
    return __sync_add_and_fetch(&_v, 1);
  }
  
  int operator--() {
    return __sync_sub_and_fetch(&_v, 1);
  }
  
  int load() const {
    return __sync_or_and_fetch(&_v, 0);
  }
#else
  // C++11 std::atomic
  std::atomic_int _v;
  
  int operator++() { return ++_v; }
  int operator--() { return --_v; }
  int load() const { return _v.load(); }
#endif
};

Used for:
  - Packet counters
  - Reference counting (SharedPtr)
  - Statistics
```

### Critical Sections and Locking

```
Major Protected Data Structures:

1. Peer Table (Topology)
   ┌────────────────────────────────────┐
   │ Mutex: _peers_m                    │
   │ Protects: Peer list and lookup    │
   │ Operations:                        │
   │   - getPeer()                      │
   │   - addPeer()                      │
   │   - removePeer()                   │
   └────────────────────────────────────┘

2. Path List (per Peer)
   ┌────────────────────────────────────┐
   │ Mutex: _paths_m                    │
   │ Protects: Path array per peer     │
   │ Operations:                        │
   │   - Path selection                 │
   │   - Path addition/removal          │
   │   - Path quality updates           │
   └────────────────────────────────────┘

3. Network List (Node)
   ┌────────────────────────────────────┐
   │ Mutex: _networks_m                 │
   │ Protects: Network configurations   │
   │ Operations:                        │
   │   - Join/leave network             │
   │   - Network config updates         │
   │   - Multicast subscriptions        │
   └────────────────────────────────────┘

4. TX Queue (Switch)
   ┌────────────────────────────────────┐
   │ Mutex: Per-queue lock              │
   │ Protects: Outbound packet queues   │
   │ Operations:                        │
   │   - Enqueue packets                │
   │   - Dequeue for transmission       │
   │   - AQM/CoDel operations           │
   └────────────────────────────────────┘
```

### Lock Ordering and Deadlock Prevention

```
Lock Hierarchy (must acquire in this order):

Level 1: _networks_m (Node)
  │
  └─> Level 2: _peers_m (Topology)
        │
        └─> Level 3: _paths_m (Peer)
              │
              └─> Level 4: Queue locks (Switch)

Rules:
  ✓ Can acquire lower-level lock while holding higher-level
  ✗ NEVER acquire higher-level lock while holding lower-level
  ✓ Always use RAII (Mutex::Lock) for automatic unlock
  ✓ Keep critical sections short
  ✗ NEVER call external callbacks while holding locks
```

### Thread-Safe Packet Processing

```
Packet RX Path Threading:

┌─────────────────────────────────────────────────────────────┐
│ I/O Thread (per socket)                                     │
│   │                                                          │
│   ├─ 1. Receive UDP packet from OS                         │
│   │                                                          │
│   ├─ 2. Create IncomingPacket object (stack)               │
│   │                                                          │
│   └─ 3. Call Switch::onRemotePacket(tPtr, ...)             │
│         │                                                    │
│         └─> LOCK-FREE initial validation                    │
│             (size check, address extraction)                │
│                                                             │
│         ┌─> 4. Fragment reassembly (if needed)             │
│         │     - Lock per packet ID                          │
│         │                                                    │
│         └─> 5. Peer lookup                                  │
│               - Short lock on _peers_m                      │
│               - Clone SharedPtr<Peer>                       │
│                                                             │
│             6. Decrypt & process (lock-free with peer obj)  │
│                                                             │
│             7. Verb dispatch (mostly lock-free)            │
│                                                             │
│             8. Update peer stats                            │
│                - Short lock on _paths_m                     │
└─────────────────────────────────────────────────────────────┘
```

### Background Task Thread

```
Background Thread Responsibilities:

Main Loop (triggered by processBackgroundTasks):
┌────────────────────────────────────────────────────────────┐
│ WHILE running:                                             │
│                                                            │
│   1. Housekeeping (every 30 seconds)                      │
│      ├─ Clean expired peers                               │
│      ├─ Clean expired paths                               │
│      ├─ Clean expired multicast subscriptions             │
│      └─ Update statistics                                 │
│                                                            │
│   2. Peer Maintenance (periodic)                          │
│      ├─ Send ECHO for keepalive                           │
│      ├─ Retry failed paths                                │
│      └─ Check peer aliveness                              │
│                                                            │
│   3. Network Maintenance                                  │
│      ├─ Request network configs                           │
│      ├─ Renew credentials                                 │
│      └─ Announce multicast subscriptions                  │
│                                                            │
│   4. Topology Updates                                     │
│      ├─ Query root servers                                │
│      └─ Update world definition                           │
│                                                            │
│   5. AQM Dequeue Cycle                                    │
│      └─ Process TX queues (CoDel algorithm)               │
│                                                            │
│   SLEEP until next deadline                               │
└────────────────────────────────────────────────────────────┘
```

### Lock-Free Techniques

ZeroTier uses several lock-free patterns:

```
1. SharedPtr with Atomic Reference Counting:
   - Lock-free increment/decrement
   - Safe object lifecycle
   - Copy-on-write for some structures

2. Immutable Objects:
   - Identity, World, NetworkConfig
   - Read without locks
   - Replace entire object on update

3. Thread-Local Storage:
   - tPtr identifies calling thread
   - Thread-specific buffers
   - No contention

4. Compare-and-Swap Operations:
   - Packet ID generation
   - Statistics counters
   - Flag updates
```

### Service Thread Architecture

```
OneService Main Thread:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Main Event Loop:                                           │
│    │                                                         │
│    ├─> PHY (Physical Layer) Poll                           │
│    │   └─ epoll/kqueue/select on all sockets              │
│    │                                                         │
│    ├─> Process incoming packets                            │
│    │   └─ Call node->processWirePacket()                   │
│    │                                                         │
│    ├─> Process TAP device reads                            │
│    │   └─ Call node->processVirtualNetworkFrame()          │
│    │                                                         │
│    ├─> Process HTTP API requests                           │
│    │   └─ Handle control plane operations                  │
│    │                                                         │
│    ├─> Check background task deadline                      │
│    │   └─ Call node->processBackgroundTasks() if due       │
│    │                                                         │
│    └─> LOOP                                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```


---

## Network Layers (VL1 and VL2)

### Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         VL2: Virtual Layer 2                    │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Virtual Ethernet Network                                 │ │
│  │  - 64-bit Network ID                                      │ │
│  │  - MAC addressing (48-bit)                                │ │
│  │  - Broadcast/Multicast support                            │ │
│  │  - MTU: typically 2800 bytes                              │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Network-Level Features:                                        │
│  ├─ Certificate of Membership (COM)                            │
│  ├─ Tags and Capabilities                                      │
│  ├─ Network rules (firewall, QoS)                              │
│  └─ Multicast group management                                 │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                         VL1: Virtual Layer 1                    │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Encrypted P2P Transport                                  │ │
│  │  - 40-bit ZeroTier addresses                              │ │
│  │  - End-to-end encryption                                  │ │
│  │  - NAT traversal                                           │ │
│  │  - Multipath support                                       │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Transport Features:                                            │
│  ├─ Peer discovery (WHOIS)                                     │
│  ├─ Path management                                            │
│  ├─ Relay via root servers                                     │
│  └─ Quality-of-Service                                         │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                   Physical Network (UDP)
```

### VL2: Ethernet Frame Transport

```
Ethernet Frame on TAP Device:
┌────────────────────────────────────────────────┐
│ Dest MAC (6 bytes)                             │
│ Source MAC (6 bytes)                           │
│ EtherType (2 bytes)                            │
│ Payload (up to 2788 bytes)                     │
│ [CRC - handled by OS]                          │
└────────────────────────────────────────────────┘
            │
            ▼ Encapsulation
            │
ZeroTier FRAME Packet:
┌────────────────────────────────────────────────┐
│ ZT Header (28 bytes)                           │
│   - Packet ID, Addresses, MAC, etc.           │
├────────────────────────────────────────────────┤
│ Verb: FRAME (0x06)                             │
├────────────────────────────────────────────────┤
│ Network ID (8 bytes)                           │
├────────────────────────────────────────────────┤
│ EtherType (2 bytes)                            │
├────────────────────────────────────────────────┤
│ Ethernet Payload                               │
└────────────────────────────────────────────────┘

Note: Source/Dest MAC derived from ZT addresses
      for FRAME (simple unicast)
      
For multicast/broadcast, uses EXT_FRAME (0x07):
┌────────────────────────────────────────────────┐
│ ZT Header (28 bytes)                           │
├────────────────────────────────────────────────┤
│ Verb: EXT_FRAME (0x07)                         │
├────────────────────────────────────────────────┤
│ Network ID (8 bytes)                           │
├────────────────────────────────────────────────┤
│ Flags (1 byte)                                 │
├────────────────────────────────────────────────┤
│ Source MAC (6 bytes)                           │
├────────────────────────────────────────────────┤
│ Dest MAC (6 bytes)                             │
├────────────────────────────────────────────────┤
│ EtherType (2 bytes)                            │
├────────────────────────────────────────────────┤
│ Ethernet Payload                               │
└────────────────────────────────────────────────┘
```

### Network Membership and Security

```
Network Join Process:

1. Node requests to join network (via API or config)
   │
   ▼
2. Send NETWORK_CONFIG_REQUEST to controller
   │
   ▼
3. Controller validates:
   ├─ Node authorization
   ├─ Network existence
   └─ Access control rules
   │
   ▼
4. Controller responds with NETWORK_CONFIG
   ├─ Network ID
   ├─ Network name
   ├─ Certificate of Membership (COM)
   ├─ IP assignments
   ├─ Routes
   ├─ Rules
   └─ Capabilities/Tags
   │
   ▼
5. Node creates Network object and TAP device
   │
   ▼
6. Announce membership to peers
   └─ Send NETWORK_CREDENTIALS with COM

Certificate of Membership (COM):
┌─────────────────────────────────────────────────┐
│ Network ID (8 bytes)                            │
│ Timestamp (8 bytes)                             │
│ Flags (8 bytes)                                 │
│ Issued To (ZT address, 5 bytes)                │
│ Signature (by controller)                       │
└─────────────────────────────────────────────────┘

Purpose: Proves authorization to participate in network
Validity: Short-lived, must be renewed periodically
```


---

## End-to-End Communication Flow

### Complete Communication Example

Let's trace a complete packet from Application A to Application B:

```
Application A                                                                                          Application B
(10.147.20.5)                                                                                        (10.147.20.10)
on ZT Network 8056c2e21c000001                                                                      on ZT Network 8056c2e21c000001
     │                                                                                                      │
     │ 1. App sends TCP packet to 10.147.20.10:80                                                         │
     │    (e.g., HTTP request)                                                                             │
     ▼                                                                                                      │
┌──────────────────────────────────────────────────────────┐                                              │
│ OS Network Stack                                         │                                              │
│ - Creates IP packet                                      │                                              │
│ - Determines dest on ZT network                          │                                              │
│ - Looks up dest MAC via ARP (handled by ZT)             │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 2. ZT TAP Device Receives Ethernet Frame                │                                              │
│    ┌──────────────────────────────────────────────────┐ │                                              │
│    │ Dest MAC: 28:66:83:aa:bb:cc (from ZT address)   │ │                                              │
│    │ Src MAC:  28:66:83:11:22:33 (from ZT address)   │ │                                              │
│    │ Type: 0x0800 (IPv4)                             │ │                                              │
│    │ Payload: IP packet                               │ │                                              │
│    └──────────────────────────────────────────────────┘ │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 3. Network Layer (VL2)                                   │                                              │
│ - Verify membership in network 8056c2e21c000001         │                                              │
│ - Look up dest ZT address from MAC                      │                                              │
│ - Apply egress rules                                     │                                              │
│ - Classify QoS (examine IP headers)                     │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 4. Switch: Create ZT Packet                             │                                              │
│    Verb: FRAME (0x06)                                    │                                              │
│    Network ID: 8056c2e21c000001                          │                                              │
│    EtherType: 0x0800                                     │                                              │
│    Payload: [IP packet]                                  │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 5. Peer Lookup                                           │                                              │
│ - Get Peer for dest ZT address                          │                                              │
│ - If peer unknown: send WHOIS to root                   │                                              │
│ - Wait for WHOIS response                                │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 6. Path Selection                                        │                                              │
│ - Query peer for best path                              │                                              │
│ - Check if direct path exists                           │                                              │
│ - If no direct: use relay via root server               │                                              │
│ Selected: Direct path via WiFi                           │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 7. Compression (if beneficial)                           │                                              │
│ - LZ4 compress if payload > threshold                   │                                              │
│ - Set compressed flag                                    │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 8. Encryption (VL1)                                      │                                              │
│ - Generate random packet ID / IV                        │                                              │
│ - Encrypt with peer shared key (AES-GMAC-SIV)          │                                              │
│ - Generate MAC authentication tag                       │                                              │
│ - Apply ephemeral key (extended armor)                  │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 9. AQM Enqueue                                           │                                              │
│ - Add to per-network, per-flow queue                    │                                              │
│ - CoDel algorithm manages queue                          │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       ▼                                                                                   │
┌──────────────────────────────────────────────────────────┐                                              │
│ 10. Physical Transmission                                │                                              │
│ - Dequeue from AQM                                       │                                              │
│ - Send UDP packet to peer's endpoint                    │                                              │
│ - Track for potential retransmission                     │                                              │
└──────────────────────┬───────────────────────────────────┘                                              │
                       │                                                                                   │
                       │════════════════════════════════════════════════════════════════════════════      │
                       │                     Physical Network (Internet)                                   │
                       │════════════════════════════════════════════════════════════════════════════      │
                       │                                                                                   │
                       └──────────────────────────────────────────────────────────────────────────────────>│
                                                                                                           │
                                                                                                           ▼
                                                                              ┌─────────────────────────────────────────────┐
                                                                              │ 11. Physical Reception (Node B)              │
                                                                              │ - Receive UDP packet                         │
                                                                              │ - Extract ZT packet                          │
                                                                              └──────────────────┬──────────────────────────┘
                                                                                                 │
                                                                                                 ▼
                                                                              ┌─────────────────────────────────────────────┐
                                                                              │ 12. Decryption & Authentication              │
                                                                              │ - Verify MAC                                 │
                                                                              │ - Decrypt payload                            │
                                                                              │ - Verify ephemeral key                       │
                                                                              └──────────────────┬──────────────────────────┘
                                                                                                 │
                                                                                                 ▼
                                                                              ┌─────────────────────────────────────────────┐
                                                                              │ 13. Decompression (if needed)                │
                                                                              │ - LZ4 decompress                             │
                                                                              └──────────────────┬──────────────────────────┘
                                                                                                 │
                                                                                                 ▼
                                                                              ┌─────────────────────────────────────────────┐
                                                                              │ 14. Verb Processing                          │
                                                                              │ - FRAME verb handler                         │
                                                                              │ - Extract network ID                         │
                                                                              │ - Verify membership                          │
                                                                              └──────────────────┬──────────────────────────┘
                                                                                                 │
                                                                                                 ▼
                                                                              ┌─────────────────────────────────────────────┐
                                                                              │ 15. Network Layer Processing                 │
                                                                              │ - Apply ingress rules                        │
                                                                              │ - Learn source MAC                           │
                                                                              │ - Update peer statistics                     │
                                                                              └──────────────────┬──────────────────────────┘
                                                                                                 │
                                                                                                 ▼
                                                                              ┌─────────────────────────────────────────────┐
                                                                              │ 16. TAP Device Injection                     │
                                                                              │ - Reconstruct Ethernet frame                 │
                                                                              │ - Write to TAP device                        │
                                                                              └──────────────────┬──────────────────────────┘
                                                                                                 │
                                                                                                 ▼
                                                                              ┌─────────────────────────────────────────────┐
                                                                              │ 17. OS Network Stack (Node B)                │
                                                                              │ - Receive Ethernet frame                     │
                                                                              │ - Process IP packet                          │
                                                                              │ - Deliver to TCP stack                       │
                                                                              └──────────────────┬──────────────────────────┘
                                                                                                 │
                                                                                                 ▼
                                                                                          Application B
                                                                                        (HTTP server receives request)
```

### Performance Characteristics

```
Typical Latencies:

Direct P2P (same LAN):
  ├─ Overhead: 0.1-0.5 ms
  └─ Total: ~1 ms

Direct P2P (cross-internet):
  ├─ Overhead: 0.5-2 ms
  ├─ Physical: varies by ISP
  └─ Total: Physical + 0.5-2 ms

Relayed (via root server):
  ├─ Overhead: 1-3 ms
  ├─ Two hops to root and back
  └─ Total: 2x Physical + 1-3 ms

Encryption Overhead:
  ├─ AES-GMAC-SIV: ~50 µs (modern CPU)
  ├─ Salsa20/12: ~30 µs (modern CPU)
  └─ Negligible on gigabit links

Throughput:
  ├─ Direct P2P: Limited by physical link
  ├─ Relayed: Limited by root server capacity
  └─ CPU-bound at 1-10 Gbps (depending on cipher)
```

