# P2P Packet Order and Loss Handling Explanation

## Table of Contents
1. [Overview](#overview)
2. [Packet Order Preservation](#packet-order-preservation)
3. [Flow-Based Routing](#flow-based-routing)
4. [Multipath Bonding and Ordering](#multipath-bonding-and-ordering)
5. [Packet Loss Handling](#packet-loss-handling)
6. [Fragmentation and Reassembly](#fragmentation-and-reassembly)
7. [Quality of Service (QoS)](#quality-of-service-qos)
8. [Active Queue Management (AQM)](#active-queue-management-aqm)

---

## Overview

ZeroTier implements a sophisticated peer-to-peer (P2P) networking system that maintains packet order and handles packet loss through several mechanisms. The system operates at the virtual layer 1 (VL1) level, providing an encrypted P2P transport that can utilize multiple physical paths simultaneously while preserving packet order for individual flows.

**Key Principles:**
- **Flow-based routing**: Packets belonging to the same flow (e.g., TCP connection) always use the same path
- **No automatic retransmission**: ZeroTier operates at layer 2; packet loss is handled by higher-layer protocols
- **Multipath capable**: Can use multiple physical paths simultaneously without breaking packet order
- **QoS-aware**: Includes active queue management to prevent bufferbloat and packet loss

---

## Packet Order Preservation

### Basic Mechanism

ZeroTier preserves packet order through **flow hashing**. When multipath bonding is enabled, packets are not sent arbitrarily across available paths. Instead, packets are routed based on their flow characteristics.

### Flow Identification

A flow is identified by hashing the following packet characteristics:

**For IPv4 packets:**
```cpp
flowId = destinationPort ^ sourcePort ^ protocol
```

**For IPv6 packets:**
```cpp
flowId = destinationPort ^ sourcePort ^ protocol
```

This hash computation occurs in `IncomingPacket.cpp` and `Switch.cpp`:

```cpp
if (peer->flowHashingSupported()) {
    // Extract protocol, source port, and destination port from packet
    _flowId = dstPort ^ srcPort ^ proto;
}
```

### Path Selection Per Flow

Once a flow ID is determined, the bonding policy ensures that all packets with the same flow ID use the same path:

1. **Flow Creation**: When a packet arrives with a new flow ID, a flow object is created and assigned to a specific path
2. **Consistent Routing**: Subsequent packets with the same flow ID always use the same path
3. **Flow Lifetime**: Flows persist throughout the connection lifetime

This is implemented in `Bond.cpp`:

```cpp
SharedPtr<Path> Bond::getAppropriatePath(int64_t now, int32_t flowId)
{
    if (flowId == -1) {
        // No specific flow, use default path selection
        // ...
    }
    
    std::map<int16_t, SharedPtr<Flow>>::iterator it = _flows.find(flowId);
    if (it == _flows.end()) {
        // Create new flow and assign it to a path
        SharedPtr<Flow> flow = createFlow(ZT_MAX_PEER_NETWORK_PATHS, flowId, entropy, now);
        _flows[flowId] = flow;
    }
    
    // Return the path associated with this flow
    return it->second->assignedPath();
}
```

### Why This Preserves Order

**Within a single path**, packets naturally arrive in order (or close to it) because:
- They traverse the same physical route
- Experience similar latency
- Are not reordered by the transport mechanism

**Across multiple paths**, order is preserved because:
- Each TCP connection (or UDP flow) is a separate flow
- All packets in that flow use the same path
- Different flows can use different paths without affecting each other

### Example Scenario

Consider a peer with two physical paths (Path A and Path B):

```
TCP Connection 1 (SSH, port 22)
  flowId = 22 ^ 54321 ^ 6 = X
  → All packets use Path A

TCP Connection 2 (HTTP, port 80)
  flowId = 80 ^ 54322 ^ 6 = Y
  → All packets use Path B

TCP Connection 3 (HTTPS, port 443)
  flowId = 443 ^ 54323 ^ 6 = Z
  → All packets use Path A
```

Within each connection, packets remain ordered because they all use the same path.

---

## Flow-Based Routing

### Flow Structure

Each flow maintains:
- **Flow ID**: Unique identifier based on port/protocol hash
- **Assigned Path**: The physical path this flow uses
- **Statistics**: Bytes in/out, packet counts, latency measurements

```cpp
struct Flow {
    int32_t id;               // Flow identifier
    SharedPtr<Path> path;     // Assigned path
    uint64_t bytesIn;         // Bytes received
    uint64_t bytesOut;        // Bytes sent
    int64_t lastActivity;     // Last packet timestamp
};
```

### Flow Assignment Strategies

Different bonding policies handle flow assignment differently:

#### 1. **BALANCE_RR (Round Robin)**
Flows are assigned to paths in round-robin fashion:
- Flow 1 → Path A
- Flow 2 → Path B
- Flow 3 → Path C
- Flow 4 → Path A (wraps around)

#### 2. **BALANCE_XOR**
Flows are assigned based on XOR of addresses:
- Destination-based: Packets to same destination use same path
- Ensures consistent routing per peer

#### 3. **BALANCE_AWARE**
Flows are assigned based on path quality metrics:
- New flows assigned to best-performing path
- Considers latency, packet loss, and path capacity
- Flows can be moved if path quality degrades significantly

#### 4. **ACTIVE_BACKUP**
Only one path is active at a time:
- All flows use the primary path
- Failover to backup if primary fails
- No packet reordering during normal operation

#### 5. **BROADCAST**
All packets sent on all paths:
- Provides redundancy
- Receiver deduplicates based on packet ID
- Highest reliability but uses more bandwidth

### Flow Tracking

Flows are tracked in both directions:

**Outgoing Packets:**
```cpp
void Bond::recordOutgoingPacket(const SharedPtr<Path>& path, uint64_t packetId, 
                                 uint16_t payloadLength, const Packet::Verb verb, 
                                 const int32_t flowId, int64_t now)
{
    if (flowId != ZT_QOS_NO_FLOW) {
        if (_flows.count(flowId)) {
            _flows[flowId]->bytesOut += payloadLength;
        }
    }
}
```

**Incoming Packets:**
```cpp
void Bond::recordIncomingPacket(const SharedPtr<Path>& path, uint64_t packetId, 
                                 uint16_t payloadLength, Packet::Verb verb, 
                                 int32_t flowId, int64_t now)
{
    if (flowId != ZT_QOS_NO_FLOW) {
        if (!_flows.count(flowId)) {
            flow = createFlow(pathIdx, flowId, 0, now);
        } else {
            flow = _flows[flowId];
        }
    }
}
```

---

## Multipath Bonding and Ordering

### Bonding Policies

ZeroTier supports multiple bonding policies that affect how packets are distributed across paths while maintaining order:

| Policy | Description | Order Guarantee |
|--------|-------------|-----------------|
| **NONE** | Single path, no bonding | Full order preserved |
| **ACTIVE_BACKUP** | One active path with failover | Full order preserved (except during failover) |
| **BROADCAST** | All packets on all paths | Order preserved via deduplication |
| **BALANCE_RR** | Flows striped across paths | Per-flow order preserved |
| **BALANCE_XOR** | Flows hashed by destination | Per-flow order preserved |
| **BALANCE_AWARE** | Flows assigned by path quality | Per-flow order preserved |

### Path Selection Algorithm

The path selection algorithm in `Bond::getAppropriatePath()` considers:

1. **Flow Affinity**: If flowId is specified, use the flow's assigned path
2. **Path Alive Status**: Only consider paths that are currently alive
3. **Path Quality**: For new flows, select based on:
   - Latency (lower is better)
   - Packet loss ratio (lower is better)
   - Packet error ratio (lower is better)
   - Packet delay variation/jitter (lower is better)
4. **Load Balancing**: Distribute flows across available paths

### Quality Metrics

Each path maintains quality metrics:

```cpp
class Path {
    double _latencyMean;        // Average latency
    double _latencyVariance;    // Latency variation (jitter)
    double _packetLossRatio;    // % of packets lost
    double _packetErrorRatio;   // % of packets with errors
    uint8_t _assignedFlowCount; // Number of flows using this path
};
```

### Path Failover

When a path fails:

1. **Detection**: Path marked as dead after missing heartbeats
2. **Flow Reassignment**: Active flows are moved to another path
3. **Potential Reordering**: Brief reordering possible during failover
4. **Recovery**: Applications using TCP/QUIC recover automatically

**Failover Process:**

```
Time T0: Flow using Path A (packets 1, 2, 3 sent)
Time T1: Path A fails (detected after timeout)
Time T2: Flow moved to Path B (packets 4, 5, 6 sent via Path B)

Possible arrival order: 1, 2, 4, 3, 5, 6
(Packet 3 delayed, arrives after 4)

TCP handles this: Reorders at receiver using sequence numbers
```

---

## Packet Loss Handling

### Detection Mechanisms

ZeroTier detects packet loss through several mechanisms:

#### 1. **Packet Loss Ratio (PLR) Calculation**

Paths track sent and acknowledged packets to compute loss ratio:

```cpp
double calculatePacketLossRatio(Path* path) {
    uint64_t sent = path->packetsSent;
    uint64_t received = path->packetsAcknowledged;
    if (sent == 0) return 0.0;
    return 1.0 - ((double)received / (double)sent);
}
```

#### 2. **ACK Packets (VERB_ACK)**

Peers periodically send ACK messages containing:
- Number of bytes received since last ACK
- Allows sender to detect loss by comparing sent vs. acknowledged

```cpp
// ACK response format:
// <[4] 32-bit number of bytes received since last ACK>
```

#### 3. **QoS Measurement (VERB_QOS_MEASUREMENT)**

Tracks packet journey times to detect delays and infer loss:

```cpp
// QoS record format:
// <[8] 64-bit packet ID of previously-received packet>
// <[1] 8-bit packet sojourn time>
// ...repeat for multiple packets...
```

### No Automatic Retransmission

**Important**: ZeroTier operates at layer 2 (Ethernet emulation) and does **not** automatically retransmit lost packets. This is by design:

- **Upper Layer Responsibility**: TCP, QUIC, and other protocols handle retransmission
- **Avoid Double Retransmission**: If ZT retransmitted, TCP would also retransmit, wasting bandwidth
- **Lower Latency**: No waiting for ZT-level retransmission timeouts

### Packet Loss Response

When packet loss is detected on a path:

1. **Quality Score Degradation**: Path's quality score decreases
2. **Flow Rebalancing**: New flows avoid the lossy path
3. **Existing Flows**: Continue on current path (to maintain order)
4. **Failover Threshold**: If loss exceeds threshold, path may be marked dead

### Loss Tolerance

Different bonding policies have different loss tolerance:

- **ACTIVE_BACKUP**: Switches to backup path on consistent loss
- **BALANCE_AWARE**: Redistributes new flows away from lossy paths
- **BROADCAST**: Most tolerant; packet received if any path succeeds

---

## Fragmentation and Reassembly

### Why Fragmentation Is Needed

ZeroTier packets may exceed the MTU of underlying physical links. When this happens, packets are fragmented.

### Fragment Structure

Fragments contain:
- **Packet ID**: Links fragments to original packet
- **Fragment Number**: Position in sequence (0-15)
- **Total Fragments**: Total number of fragments
- **Payload**: Portion of original packet

```cpp
// Fragment format:
// <[8] packet ID of packet whose fragment this belongs to>
// <[5] destination ZT address>
// <[1] 0xff, signals this is a fragment>
// <[1] total fragments (MS 4 bits), fragment no (LS 4 bits)>
// <[1] ZT hop count>
// <[...] fragment data>
```

### Reassembly Process

Fragments are reassembled in `Switch.cpp`:

1. **Fragment Reception**: Fragment arrives
2. **RX Queue Lookup**: Find or create RX queue entry for this packet ID
3. **Fragment Storage**: Store fragment in appropriate slot
4. **Completeness Check**: Check if all fragments received
5. **Assembly**: Combine fragments into complete packet
6. **Decryption**: Decrypt and MAC-verify complete packet
7. **Processing**: Pass to protocol handler

```cpp
struct RXQueueEntry {
    volatile int64_t timestamp;
    volatile uint64_t packetId;
    IncomingPacket frag0;                              // First fragment
    Packet::Fragment frags[ZT_MAX_PACKET_FRAGMENTS-1]; // Other fragments
    unsigned int totalFragments;
    uint32_t haveFragments;                            // Bitmask of received
    volatile bool complete;
    Mutex lock;
};
```

### Fragment Order

**Key Point**: Fragments can arrive out of order, and ZeroTier handles this correctly:

- Each fragment is independent
- Bitmask tracks which fragments received
- Reassembly waits for all fragments before processing
- No requirement for in-order fragment arrival

### Fragment Loss

If any fragment is lost:

1. **Timeout**: RX queue entry expires after timeout
2. **Packet Discarded**: Entire packet is lost (not just fragment)
3. **Upper Layer Handles**: TCP or other protocol retransmits complete message

**Maximum Fragments**: Protocol supports 16 fragments (4-bit counter)

---

## Quality of Service (QoS)

### Per-Flow QoS

Each flow can be assigned to a QoS bucket based on network rules:

```cpp
void Switch::aqm_enqueue(void* tPtr, const SharedPtr<Network>& network, 
                         Packet& packet, const bool encrypt, 
                         const int qosBucket, const uint64_t nwid, 
                         const int32_t flowId)
```

### QoS Buckets

Networks can define multiple QoS buckets with different priorities:
- Real-time traffic (VoIP, gaming)
- Interactive traffic (SSH, RDP)
- Bulk transfer (file downloads)
- Background (backups, updates)

### Traffic Shaping

QoS is enforced through:

1. **Separate Queues**: Each QoS bucket has its own queue
2. **Priority Scheduling**: Higher priority queues serviced first
3. **Fair Queuing**: Within same priority, flows get equal share
4. **Rate Limiting**: Buckets can have maximum rate limits

---

## Active Queue Management (AQM)

### CoDel Algorithm

ZeroTier implements the CoDel (Controlled Delay) AQM algorithm to prevent bufferbloat:

**CoDel Principles:**
- Measures sojourn time (time packet spends in queue)
- Drops packets if sojourn time consistently exceeds target
- Adapts drop rate to control queue length
- More effective than simple queue length limits

### Implementation

Located in `Switch.cpp`:

```cpp
dqr Switch::dodequeue(ManagedQueue* q, uint64_t now) {
    // Dequeue packet from front of queue
    TXQueueEntry* packet = q->q.front();
    
    // Calculate sojourn time
    uint64_t sojournTime = now - packet->creationTime;
    
    // CoDel algorithm
    if (sojournTime > TARGET_DELAY) {
        // Sojourn time too high
        if (!q->dropping) {
            // Start dropping mode
            q->dropping = true;
            q->drop_next_time = now + control_law(now, q->count);
        } else if (now >= q->drop_next_time) {
            // Drop this packet and schedule next drop
            q->count++;
            q->drop_next_time = now + control_law(now, q->count);
            return {packet, true}; // ok_to_drop = true
        }
    } else {
        // Sojourn time acceptable
        q->dropping = false;
        q->count = 0;
    }
    
    return {packet, false}; // ok_to_drop = false
}
```

### Control Law

The control law determines drop intervals:

```cpp
uint64_t Switch::control_law(uint64_t t, int count) {
    return t + INTERVAL / sqrt(count);
}
```

This creates a curve where:
- Drop rate increases as congestion persists
- Drop rate decreases as congestion clears
- Prevents constant drops that hurt throughput

### FQ-CoDel

ZeroTier implements FQ-CoDel (Fair Queuing CoDel):

1. **Multiple Queues**: One queue per flow
2. **Fair Scheduling**: Queues serviced in round-robin
3. **Per-Queue CoDel**: Each queue runs CoDel independently
4. **Flow Isolation**: Heavy flow doesn't starve light flows

```cpp
struct ManagedQueue {
    int id;                   // Queue identifier (flowId)
    int byteCredit;           // Bytes this queue can send
    int byteLength;           // Current queue length in bytes
    uint64_t first_above_time;
    uint32_t count;           // CoDel drop count
    uint64_t drop_next;
    bool dropping;            // In dropping mode?
    uint64_t drop_next_time;  // When to drop next
    std::list<TXQueueEntry*> q; // Actual packet queue
};
```

### Benefits of AQM

1. **Reduced Latency**: Prevents queue buildup
2. **Better Interactive Performance**: VoIP, gaming, SSH stay responsive
3. **Fair Bandwidth Sharing**: Multiple flows coexist fairly
4. **Automatic Adaptation**: Responds to changing network conditions
5. **No Manual Tuning**: Works without configuration

### Interaction with Packet Order

AQM can drop packets, but this doesn't break ordering:

- **Drops are per-flow**: Only affect one TCP connection
- **TCP handles drops**: Retransmits missing packets in order
- **Flow isolation**: One flow's drops don't affect others
- **Selective drops**: Drops newest packets when in dropping mode

---

## Summary

### How Packet Order Is Preserved

1. **Flow Hashing**: Packets with same source/dest/protocol use same path
2. **Consistent Routing**: Each flow always uses its assigned path
3. **Per-Flow Queuing**: Packets within a flow stay ordered in queue
4. **In-Path Ordering**: Single path naturally preserves order

### What Happens When Packets Are Lost

1. **Detection**: Loss detected via ACKs, QoS measurements, and statistics
2. **Path Quality Update**: Lossy paths get lower quality scores
3. **Flow Rebalancing**: New flows avoid lossy paths
4. **No ZT Retransmission**: Upper-layer protocols (TCP) handle retransmission
5. **Failover**: If loss exceeds threshold, path may failover

### Key Takeaways

- **Order preservation is per-flow, not global**: Different connections can use different paths
- **Flow hashing is automatic**: Based on packet headers, no configuration needed
- **Loss handling is delegated**: TCP and other protocols handle retransmission
- **Multipath doesn't break order**: Each flow uses one path at a time
- **Failover may cause brief reordering**: But TCP handles this transparently
- **AQM prevents congestion**: Better than dealing with aftermath of packet loss

### Design Philosophy

ZeroTier's approach follows the **end-to-end principle**:

- Layer 2 provides **connectivity** and **encryption**
- Layer 4 (TCP) provides **reliability** and **ordering**
- Don't duplicate functionality across layers
- Keep each layer focused on its responsibilities

This results in a **clean, efficient, and robust** system that scales well and performs optimally across diverse network conditions.
