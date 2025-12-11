# ZeroTier Node Core Files Documentation

This document describes each file in the `node/` directory and its purpose within the ZeroTier network virtualization engine.

## Core System Files

### Node.hpp / Node.cpp
The main implementation of the ZeroTier node as defined in the C API. This is the top-level class that coordinates all other components and implements the core `ZT_Node_*` functions exposed to the outside world. It manages the node's lifecycle, configuration, and interactions with the virtual network.

### Switch.hpp / Switch.cpp
Core of the distributed Ethernet switch and protocol implementation. This is where transport-layer ZeroTier packets and virtual network packets from tap devices meet. It handles packet routing, wrapping/unwrapping, queuing, and timeouts. Despite the name, it's the central hub where everything converges.

### RuntimeEnvironment.hpp
Holds the global state for a ZeroTier node instance. Contains pointers to all major subsystems (Switch, Topology, Multicaster, etc.) and manages their lifecycle. This is the shared context passed around to most major functions.

### Constants.hpp
Defines system-wide constants and auto-detects environment information like operating system, architecture, and byte order. Contains fundamental limits and configuration values used throughout the codebase.

## Network and Topology Management

### Network.hpp / Network.cpp
Represents a virtual LAN (network) that the node has joined. Manages network membership, configuration, credentials, multicast groups, and handles incoming/outgoing frames for that specific network.

### Topology.hpp / Topology.cpp
Database of network topology, managing the collection of known peers. Handles peer discovery, addition, lookup, and maintains connections to root servers (via World). Coordinates peer-to-peer connectivity.

### Peer.hpp / Peer.cpp
Represents a peer on the P2P network (virtual layer 1). Maintains cryptographic keys, paths to the peer, packet queues, and handles encryption/decryption of communications with that peer. Manages multiple paths and bonding.

### Path.hpp / Path.cpp
Represents a single path across the physical network to reach a peer. Contains IP address, port, latency metrics, and other path quality indicators. Multiple paths can exist to the same peer.

### Bond.hpp / Bond.cpp
Implements multipath bonding policies for aggregating multiple physical paths to a peer. Supports various bonding modes including active-backup, broadcast, and load balancing. Manages path quality monitoring and failover.

### World.hpp
Defines the "World" configuration, which contains information about root servers (planets). The World defines the global network topology including planet addresses and their stable endpoints.

## Packet Handling

### Packet.hpp / Packet.cpp
Base class for ZeroTier protocol packets. Handles encryption, authentication (via Poly1305), compression, and serialization of protocol messages. Contains packet type definitions and protocol version information.

### IncomingPacket.hpp / IncomingPacket.cpp
Subclass of Packet that handles the decoding of incoming packets. Processes various packet types (HELLO, OK, WHOIS, frames, etc.) and dispatches them to appropriate handlers. Manages authentication and retry logic.

### OutboundMulticast.hpp / OutboundMulticast.cpp
Manages an outbound multicast packet, tracking which peers should receive it and handling delivery to multiple destinations efficiently.

### PacketMultiplexer.hpp / PacketMultiplexer.cpp
Multiplexes packet processing across multiple threads for improved performance. Manages worker threads and queues for parallel packet handling on multicore systems.

## Multicast Support

### Multicaster.hpp / Multicaster.cpp
Database of known multicast group members within networks. Handles multicast group subscriptions, member tracking, and efficient multicast packet distribution using gather/propagate algorithms.

### MulticastGroup.hpp
Represents a multicast group composed of a multicast MAC address and an ADI (Additional Distinguishing Information) field. Used for selective multicast and broadcast handling, especially for protocols like ARP.

## Network Configuration and Credentials

### NetworkConfig.hpp / NetworkConfig.cpp
Contains the configuration for a virtual network including IP assignments, routes, rules, capabilities, tags, and certificates. This is the authoritative configuration provided by network controllers.

### NetworkController.hpp
Interface for network controller implementations. Controllers are responsible for authorizing members and providing network configurations. This defines the API that controllers must implement.

### CertificateOfMembership.hpp / CertificateOfMembership.cpp
Certificate proving membership in a private network. Contains qualifiers (id/value/maxDelta tuples) that must "agree" between peers for them to communicate on a private network.

### CertificateOfOwnership.hpp / CertificateOfOwnership.cpp
Certificate indicating ownership of network identifiers (MAC addresses, IP addresses). Used to prevent spoofing by proving that a member legitimately owns an address.

### Capability.hpp / Capability.cpp
A set of grouped and signed network flow rules that can be presented by a sender to prove authorization to send certain traffic. Implements capability-based security for fine-grained access control.

### Tag.hpp / Tag.cpp
A signed tag that can be associated with network members and matched in rules. Tags group members (who they are), while capabilities group rules (what they can do). Used for role-based access control.

### Revocation.hpp / Revocation.cpp
Certificate to instantaneously revoke a COM, capability, or tag. Supports fast propagation via rumor mill algorithm to quickly disable compromised credentials across the network.

### Membership.hpp / Membership.cpp
Container for certificates of membership and other network credentials. Acts as a relational join between Peer and Network, managing credential validation and presentation.

### Credential.hpp
Base class for all credential types, defining the common credential type enumeration used across the system.

## Identity and Addressing

### Identity.hpp / Identity.cpp
Represents a ZeroTier identity consisting of a public key, a 40-bit address derived from that key, and a self-signature. The address derivation algorithm makes address collision computationally expensive.

### Address.hpp
A 40-bit (5-byte) ZeroTier address. Provides encoding, decoding, comparison, and hashing operations for ZeroTier addresses.

### MAC.hpp
Represents a 48-bit Ethernet MAC address. Provides conversion to/from ZeroTier addresses and network IDs for virtual network operation.

### InetAddress.hpp / InetAddress.cpp
Extends `sockaddr_storage` with C++ methods for handling physical IP addresses (both IPv4 and IPv6). Includes scope determination, serialization, and comparison operations.

## Cryptography

### AES.hpp / AES.cpp / AES_aesni.cpp / AES_armcrypto.cpp
AES-256 encryption and related operations including GMAC and CTR mode. Includes hardware acceleration for x86 (AES-NI) and ARM (crypto extensions) with software fallback.

### Salsa20.hpp / Salsa20.cpp
Salsa20 stream cipher implementation with optional SSE optimization. Used for high-speed packet encryption in the ZeroTier protocol.

### Poly1305.hpp / Poly1305.cpp
Poly1305 one-time authentication code (MAC) implementation. Used with stream ciphers to provide authenticated encryption for packets. The 32-byte key must never be reused.

### ECC.hpp / ECC.cpp
Elliptic curve cryptography implementation. The standard version uses C25519/Ed25519, while FIPS builds use NIST curves. Handles key generation, signing, and key agreement.

### SHA512.hpp / SHA512.cpp
SHA-384 and SHA-512 hash functions. Uses native implementations on macOS/iOS and custom implementation on other platforms. Used for identity derivation and various cryptographic operations.

## Monitoring and Diagnostics

### Trace.hpp / Trace.cpp
Remote tracing and trace logging handler. Provides debugging and monitoring capabilities, allowing collection of detailed protocol events for troubleshooting.

### Metrics.hpp / Metrics.cpp
Prometheus metrics integration for monitoring packet counts, latency, errors, and other operational statistics. Enables real-time observability of ZeroTier node operation.

### SelfAwareness.hpp / SelfAwareness.cpp
Tracks changes to the node's real-world addresses as reported by trusted peers. Helps the node understand its external network interfaces and detect address changes (e.g., behind NAT).

## Data Structures and Utilities

### Buffer.hpp
A templated variable-length but statically-allocated buffer with bounds checking. Used throughout the codebase for safe serialization/deserialization. Encodes integers in big-endian byte order.

### Dictionary.hpp
A small packed key=value store that serializes to a human-readable (if no binary data) format. Used for network configurations and persistent storage. Uses linear search for simplicity.

### Hashtable.hpp
A minimal hash table implementation optimized for the ZeroTier core. Provides efficient key-value lookup without external dependencies.

### RingBuffer.hpp
A circular buffer for tracking continuously-evolving variables like path quality metrics. Provides statistical operations on sliding windows without expensive memory operations.

### SharedPtr.hpp
An introspective reference-counted smart pointer. Classes that need reference counting must friend this class and include an `__refCount` member. Zero-overhead when used properly.

### AtomicCounter.hpp
Simple atomic counter supporting thread-safe increment and decrement operations. Used for reference counting and other concurrent access scenarios.

### Mutex.hpp
Cross-platform mutex implementation. Uses pthreads on Unix-like systems and Windows critical sections on Windows. Includes RAII lock guard.

### Utils.hpp / Utils.cpp
Miscellaneous utility functions including secure random number generation, byte order conversions, string parsing, timing functions, and platform-specific operations.

### DNS.hpp
DNS data serialization methods for handling virtual network DNS configurations. Provides helpers for encoding/decoding DNS server information.

## Other Files

### README.md
Documentation explaining that this directory contains the core ZeroTier network hypervisor, its design principles (OS-independence, minimal footprint), and coding guidelines.
