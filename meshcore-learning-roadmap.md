# MeshCore Learning Roadmap

*A practical, source-grounded curriculum for understanding the MeshCore protocol, documentation, and public codebase as a beginner in C/C++ with prior JavaScript, web-development, and business-logic experience.*

---

## Purpose

This guide is designed to help you understand:

- how MeshCore packets are structured;
- how LoRa settings affect range, airtime, and network capacity;
- how flooding and direct routing work;
- how repeaters, companions, room servers, sensors, and standalone devices differ;
- how MeshCore handles identity, encryption, signatures, and acknowledgements;
- how the public C++ codebase is organized;
- how to connect your existing JavaScript knowledge to embedded C++;
- which topics are essential now and which can wait.

The goal is **not** to memorize every constant or radio register. The goal is to develop a reliable mental model of what happens when a message is created, encrypted, transmitted, repeated, received, acknowledged, and stored.

---

# 1. Important Scope Clarification

The original roadmap mixed together three separate layers:

1. **MeshCore protocol and core C++ library**
   - Packet formats
   - Routing
   - Identity
   - Encryption
   - Radio scheduling
   - Forwarding behavior
   - Public APIs

2. **Public MeshCore example firmwares**
   - Companion radio
   - Repeater
   - Room server
   - Secure chat
   - Sensor
   - KISS modem

3. **MeshOS**
   - The standalone graphical application used on devices such as the LilyGO T-Deck and T-Pager
   - Device-specific user interface
   - Screen flows
   - Mapping
   - GPS integration
   - Input handling
   - Application-specific behavior

## Why this distinction matters

[Certain] The first two layers are publicly documented and represented in the open-source MeshCore repository.

[Certain] MeshOS itself is not fully open source. That means studying LVGL internals, exact task pinning, or its complete screen architecture is **not required** to understand the public MeshCore protocol and library.

You may still learn LVGL, FreeRTOS, GPS, maps, and display drivers later if your goal becomes building your own MeshOS-like firmware.

---

# 2. Corrections to the Initial Roadmap

Several claims in the original roadmap should be corrected before using it as a foundation.

## 2.1 Encryption

[Certain] Current MeshCore core code uses:

- **AES-128**
- **HMAC-SHA-256**
- a truncated MAC
- Ed25519-style public/private identities and signatures
- shared-secret calculation between identities

It does **not** use AES-256 in the public core implementation.

Relevant source:

- `src/Utils.cpp`
- `src/Identity.cpp`

## 2.2 Compression

[Certain] LZW compression is not currently part of the core packet-processing path in the public MeshCore source.

Do not treat compression as a prerequisite for understanding the current packet protocol.

## 2.3 MQTT

[Certain] MQTT is not part of the central MeshCore routing model.

MQTT may be used by external bridges or community integrations, but it is not required to understand:

- packets;
- repeaters;
- companion devices;
- direct routes;
- flooding;
- contacts;
- channels;
- room servers.

## 2.4 KISS

[Certain] KISS is a separate modem interface.

It matters when connecting MeshCore-compatible radio hardware to external packet-radio software. It is not the default communication protocol used between an ordinary companion radio and a phone application.

## 2.5 FreeRTOS and Core Pinning

[Certain] The public MeshCore core is mainly written around:

- `begin()`;
- repeated `loop()` calls;
- packet queues;
- callbacks;
- abstract interfaces;
- polling;
- scheduled transmissions.

FreeRTOS knowledge is useful later, but it is not the first thing you need in order to read `Packet`, `Dispatcher`, or `Mesh`.

## 2.6 Direct Routing

[Certain] MeshCore direct routing is best understood as **source routing**.

The sender supplies the repeater path inside the packet. Each matching repeater consumes its part of that path and forwards the packet.

This differs from an IP router independently consulting a routing table at every hop.

---

# 3. Your Existing Knowledge Advantage

You are a beginner in C/C++, but you already understand:

- JavaScript;
- application state;
- business logic;
- event-driven systems;
- APIs;
- data flow;
- front-end component architecture;
- debugging;
- asynchronous behavior.

That means you do **not** need a broad beginner programming course.

You need a translation layer between web development and embedded systems.

---

# 4. Phase 1: Embedded C++ Through a JavaScript Lens

## 4.1 Core topics to learn

Learn these first:

- `.h` files versus `.cpp` files;
- declarations versus implementations;
- fixed-width integers;
- arrays;
- pointers;
- references;
- `const`;
- classes and constructors;
- inheritance;
- virtual methods;
- access modifiers;
- stack memory;
- static memory;
- explicit ownership;
- object lifetime;
- macros;
- bitwise operations;
- C-style strings;
- `memcpy`;
- `memset`;
- `memcmp`.

## 4.2 Important integer types

```cpp
uint8_t
uint16_t
uint32_t
int8_t
size_t
```

Unlike JavaScript's general-purpose `number`, these types have fixed sizes.

Examples:

```cpp
uint8_t value = 255;
uint16_t port = 4242;
uint32_t timestamp = 1785005766;
int8_t snrQuarterDb = -12;
```

## 4.3 JavaScript-to-C++ translation table

| JavaScript concept | Embedded C++ equivalent |
|---|---|
| `Uint8Array` | `uint8_t*` plus a length |
| Plain object | `struct` or `class` |
| Event callback | Virtual method override |
| Browser event loop | Repeated `loop()` call |
| Garbage collection | Explicit ownership or fixed pools |
| `number` | Multiple fixed-width integer types |
| JSON object | Manually serialized bytes |
| Promise/event listener | Polling, queue, callback, or state machine |
| Array length property | Separate explicit length field |
| Module export | Header declaration plus implementation |

## 4.4 Pointer-and-length pairs

A common embedded pattern is:

```cpp
void process(const uint8_t* data, size_t len);
```

This means:

- `data` points to the first byte;
- `len` tells the function how many bytes are valid.

JavaScript analogy:

```js
function process(data) {
  // data is a Uint8Array
}
```

C++ does not automatically carry array length information with a raw pointer.

## 4.5 Why fixed arrays are common

```cpp
uint8_t payload[MAX_PACKET_PAYLOAD];
```

Embedded systems often avoid frequent dynamic memory allocation because it can create:

- fragmentation;
- unpredictable failures;
- timing inconsistency;
- difficult ownership bugs.

## 4.6 Bitwise operations

Example:

```cpp
uint8_t routeType = header & 0x03;
uint8_t payloadType = (header >> 2) & 0x0F;
```

Interpretation:

- `0x03` keeps only the lowest two bits;
- shifting right by two moves the payload bits into place;
- `0x0F` keeps four bits.

### Checkpoint

Explain this header format:

```text
0bVVPPPPRR
```

- `VV`: version
- `PPPP`: payload type
- `RR`: route type

---

# 5. Phase 2: Bits, Bytes, and Wire Formats

This phase is more important than advanced C++.

## 5.1 Topics to learn

- binary notation;
- hexadecimal notation;
- bits and bytes;
- nibbles;
- masks;
- shifts;
- little-endian encoding;
- serialization;
- deserialization;
- framing;
- fixed-length fields;
- variable-length fields;
- payload lengths;
- maximum transmission unit;
- UTF-8;
- binary blobs;
- zero padding;
- hashes;
- CRCs;
- MACs;
- signatures.

## 5.2 Hexadecimal

One byte ranges from:

```text
0x00 to 0xFF
```

Each hexadecimal digit represents four bits.

Examples:

```text
0x0F = 00001111
0xF0 = 11110000
0xA5 = 10100101
```

## 5.3 Endianness

A 32-bit value such as:

```text
0x12345678
```

may appear in little-endian memory as:

```text
78 56 34 12
```

You must learn to distinguish:

- the numeric value;
- the order bytes appear in memory or on the wire.

## 5.4 Serialization

Serialization means converting structured information into bytes.

Example conceptual object:

```js
{
  timestamp: 1785005766,
  text: "hello"
}
```

Possible compact binary layout:

```text
[4-byte timestamp][UTF-8 text bytes]
```

MeshCore avoids JSON on the radio because JSON wastes airtime on:

- field names;
- quotes;
- punctuation;
- repeated structure.

## 5.5 Padding

AES works on fixed-size blocks.

A payload shorter than a complete block may be padded with zero bytes before encryption.

This means decrypted buffers may contain trailing zeroes that are not part of the original logical message.

## 5.6 CRC, hash, MAC, and signature

| Mechanism | Main purpose |
|---|---|
| CRC | Detect accidental corruption |
| Packet hash | Identify or deduplicate packets |
| HMAC | Verify authenticity and integrity using a shared secret |
| Signature | Verify that a public identity signed content |

### Checkpoint

Given a raw packet dump, identify:

1. header;
2. optional transport codes;
3. path data;
4. payload.

---

# 6. Phase 3: LoRa and RF Fundamentals

## 6.1 Learn these concepts in order

1. Frequency
2. Wavelength
3. dBm
4. Noise floor
5. RSSI
6. SNR
7. Receiver sensitivity
8. Link budget
9. Antenna gain
10. Antenna polarization
11. Antenna height
12. Cable loss
13. Fresnel clearance
14. Spreading factor
15. Bandwidth
16. Coding rate
17. Preamble
18. Sync word
19. Time on air
20. Half-duplex communication
21. Collisions
22. Hidden-node problem
23. Near-far problem
24. Channel Activity Detection
25. Regional legal limits

## 6.2 Frequency

The frequency determines the radio band being used.

Examples include portions of:

- 433 MHz;
- 868 MHz;
- 915 MHz.

The correct frequency depends on region and hardware.

## 6.3 dBm

dBm is a logarithmic power scale.

Examples:

```text
0 dBm = 1 mW
10 dBm = 10 mW
20 dBm = 100 mW
30 dBm = 1 W
```

An increase of 3 dB is approximately double the power.

## 6.4 RSSI

RSSI estimates total received signal power.

A value closer to zero is stronger:

```text
-65 dBm is stronger than -110 dBm
```

RSSI includes:

- wanted signal;
- noise;
- interference.

## 6.5 SNR

SNR compares the wanted signal with the noise floor.

LoRa can decode signals at negative SNR values.

Examples:

```text
+10 dB: signal clearly above noise
0 dB: signal and noise similar
-10 dB: signal below noise, but LoRa may still decode it
```

## 6.6 Link budget

A simplified link budget is:

```text
TX power
+ TX antenna gain
- TX cable loss
- path loss
+ RX antenna gain
- RX cable loss
= received power
```

A packet is likely decodable when the received power and SNR are sufficient for the selected LoRa settings.

## 6.7 Spreading factor

Higher SF:

- increases symbol duration;
- often improves sensitivity;
- increases time on air;
- decreases throughput;
- increases collision exposure.

Lower SF:

- reduces airtime;
- increases throughput;
- requires a stronger or cleaner signal.

## 6.8 Bandwidth

Narrower bandwidth generally:

- improves sensitivity;
- reduces throughput;
- increases time on air.

Wider bandwidth generally:

- reduces airtime;
- increases throughput;
- requires stronger signal conditions.

## 6.9 Coding rate

More error-correction redundancy may improve resilience, but increases airtime.

## 6.10 Time on air

Every parameter eventually affects airtime.

Airtime matters because LoRa is a shared medium.

Longer packets:

- occupy the channel longer;
- delay other transmissions;
- increase collision risk;
- reduce regional capacity;
- slow flood propagation.

## 6.11 Channel Activity Detection

CAD checks whether LoRa-like activity is present before transmitting.

It is not perfect carrier sensing. It helps reduce collisions but cannot eliminate:

- hidden nodes;
- simultaneous starts;
- distant interference;
- near-far issues.

### Checkpoint

Explain why moving from SF7 to SF11 might improve one marginal link while reducing overall network health in a busy region.

---

# 7. Phase 4: SX1262 and Radio Driver Behavior

You do not need to memorize the SX1262 datasheet. Learn the boundary between the microcontroller and radio.

## 7.1 Core hardware concepts

- SPI;
- chip select or NSS;
- BUSY pin;
- DIO interrupt pins;
- RESET;
- radio states;
- transmit mode;
- receive mode;
- standby;
- CAD mode;
- RadioLib;
- radio abstraction.

## 7.2 SPI

The ESP32 communicates with the SX1262 over SPI.

Typical signals:

- SCK;
- MOSI;
- MISO;
- NSS or CS.

## 7.3 BUSY

The BUSY pin tells the microcontroller when the radio is not ready to accept another command.

## 7.4 DIO1

DIO1 may signal events such as:

- packet received;
- packet transmitted;
- timeout;
- CAD result.

## 7.5 RadioLib

RadioLib hides many low-level register and command details.

MeshCore adds another abstraction on top through a `Radio` interface.

This allows the protocol and dispatcher logic to depend on common operations such as:

```cpp
recvRaw()
startSendRaw()
isSendComplete()
getLastRSSI()
getLastSNR()
getEstAirtimeFor()
```

## 7.6 Why this abstraction matters

It separates:

```text
MeshCore routing behavior
```

from:

```text
specific radio hardware behavior
```

### Checkpoint

Explain which layer is responsible for each item:

- deciding the next repeater;
- calculating time on air;
- moving bytes over SPI;
- encrypting a direct message;
- reading RSSI.

---

# 8. Phase 5: MeshCore Device Roles

## 8.1 Companion

A companion radio connects a phone, browser, or computer to the mesh.

It typically handles:

- local radio communication;
- BLE or USB communication;
- contacts;
- channels;
- messages;
- device configuration.

Companions normally do not act as infrastructure repeaters.

## 8.2 Repeater

A repeater:

- receives eligible packets;
- checks whether they should be forwarded;
- appends or consumes path information;
- delays retransmission;
- suppresses duplicates;
- retransmits packets.

Repeaters should usually be:

- fixed;
- elevated;
- powered reliably;
- positioned for broad coverage.

## 8.3 Room Server

A room server acts as an offline message service.

It can:

- store posts;
- allow clients to retrieve missed history;
- provide BBS-like rooms;
- support asynchronous communication.

This differs from ordinary group channels, which are live broadcasts.

## 8.4 Standalone Device

A standalone device combines:

- local user interface;
- radio;
- contacts;
- messages;
- possibly GPS;
- possibly maps;
- local storage.

MeshOS is an example of this category.

## 8.5 Sensor

A sensor firmware may:

- advertise;
- send telemetry;
- accept requests;
- return measurements;
- operate with low power.

## 8.6 KISS Modem

A KISS modem exposes raw packet-radio framing to external software.

It is useful for interoperability and experimentation, not ordinary MeshCore messaging.

### Checkpoint

Explain why mobile repeaters can make source routes less reliable than fixed repeaters.

---

# 9. Phase 6: MeshCore Packet Structure

The central public type is `Packet`.

Important source files:

- `src/Packet.h`
- `src/Packet.cpp`
- `docs/packet_format.md`

## 9.1 Packet fields

The public `Packet` class includes:

```cpp
uint8_t header;
uint16_t payload_len;
uint16_t path_len;
uint16_t transport_codes[2];
uint8_t path[MAX_PATH_SIZE];
uint8_t payload[MAX_PACKET_PAYLOAD];
int8_t _snr;
```

## 9.2 Header

The header stores:

- route type;
- payload type;
- payload version.

Conceptual layout:

```text
0bVVPPPPRR
```

## 9.3 Route types

Current public constants include:

```cpp
ROUTE_TYPE_TRANSPORT_FLOOD
ROUTE_TYPE_FLOOD
ROUTE_TYPE_DIRECT
ROUTE_TYPE_TRANSPORT_DIRECT
```

## 9.4 Payload types

Current public constants include:

```cpp
PAYLOAD_TYPE_REQ
PAYLOAD_TYPE_RESPONSE
PAYLOAD_TYPE_TXT_MSG
PAYLOAD_TYPE_ACK
PAYLOAD_TYPE_ADVERT
PAYLOAD_TYPE_GRP_TXT
PAYLOAD_TYPE_GRP_DATA
PAYLOAD_TYPE_ANON_REQ
PAYLOAD_TYPE_PATH
PAYLOAD_TYPE_TRACE
PAYLOAD_TYPE_MULTIPART
PAYLOAD_TYPE_CONTROL
PAYLOAD_TYPE_RAW_CUSTOM
```

## 9.5 Path encoding

The public packet structure encodes both:

- path hash size;
- path hash count.

Wider hashes reduce collision risk but consume more bytes.

Narrower hashes save space and allow more hops but increase ambiguity.

## 9.6 Transport codes

Transport routes can include codes used for additional scoping or forwarding behavior.

Treat transport codes as an advanced routing filter after understanding ordinary flood and direct routing.

### Checkpoint

Given a packet header byte, decode:

- protocol version;
- payload type;
- route type.

---

# 10. Phase 7: Flood Routing

Flooding is used when the sender does not already have a direct route or when a message is intended for broad distribution.

## 10.1 Simplified flood lifecycle

```text
Sender creates packet
        ↓
Sender marks route as flood
        ↓
Nearby repeaters receive packet
        ↓
Eligible repeater checks duplicate table
        ↓
Repeater appends its compact identity hash
        ↓
Repeater waits a calculated delay
        ↓
Repeater retransmits
        ↓
Process continues outward
```

## 10.2 Duplicate suppression

Repeaters track packet identifiers or hashes.

If the same packet arrives again, the node can avoid forwarding it multiple times.

This is essential because flooding naturally creates duplicate paths.

## 10.3 Random delays

Repeaters do not all retransmit immediately.

Randomized or score-based delays help:

- reduce synchronized collisions;
- favor better packet copies;
- spread retransmissions in time.

## 10.4 First-packet-wins behavior

The public implementation includes a first-valid-arrival style of handling for some received packets.

This means the first copy that arrives may determine the accepted flood path, even if another copy later has fewer hops.

## 10.5 Broadcast storms

Flooding can become destructive without:

- duplicate suppression;
- hop/path limits;
- retransmission delay;
- forwarding restrictions;
- airtime budgets;
- stable repeater placement.

### Checkpoint

Explain why duplicate suppression is required even if every repeater is well behaved.

---

# 11. Phase 8: Direct Source Routing

Direct routing uses a supplied sequence of repeater hashes.

## 11.1 Simplified direct lifecycle

```text
Sender already knows route
        ↓
Sender places route in packet
        ↓
First repeater checks first path entry
        ↓
Matching repeater removes its own entry
        ↓
Repeater forwards remaining route
        ↓
Next repeater repeats process
        ↓
Destination receives packet
```

## 11.2 Route ownership

The sender carries the route.

Repeaters do not need a global routing table describing the entire network.

## 11.3 Path discovery

A route can be learned through:

- flooded adverts;
- path-return packets;
- contact discovery;
- successful incoming packet paths;
- reciprocal route exchange.

## 11.4 Reverse and reciprocal paths

A received flood can contain the path accumulated through repeaters.

The destination may return a usable path to the sender.

Do not assume the same physical route is always equally good in both directions. RF links can be asymmetric because of:

- antenna differences;
- transmit power;
- local interference;
- receiver quality;
- terrain;
- changing noise.

## 11.5 Route breakage

A direct route may fail when:

- a repeater goes offline;
- a mobile repeater moves;
- the RF environment changes;
- a compact hash collides;
- a path becomes too long;
- a region filter changes.

### Checkpoint

Explain why direct routing reduces mesh noise compared with flooding every direct message.

---

# 12. Phase 9: Zero-Hop Communication

Zero-hop means the packet is intended only for directly reachable neighbors.

No repeater path is used.

Use cases can include:

- local control;
- nearby discovery;
- direct device interaction;
- restricted control packets;
- testing.

Zero-hop does not mean:

- encrypted by default;
- guaranteed delivery;
- immune to collision;
- physically private.

### Checkpoint

Explain the difference between:

- zero-hop delivery;
- one-hop repeater delivery;
- direct multi-hop source routing;
- flood routing.

---

# 13. Phase 10: Dispatcher and Scheduling

The `Dispatcher` is the low-level engine that coordinates receiving and transmitting.

Important files:

- `src/Dispatcher.h`
- `src/Dispatcher.cpp`

## 13.1 Responsibilities

The dispatcher manages:

- incoming raw packets;
- packet parsing;
- outbound queues;
- inbound queues;
- transmission scheduling;
- priorities;
- CAD;
- send completion;
- airtime statistics;
- duty-cycle budget;
- radio state;
- retry timing.

## 13.2 Packet manager

The packet manager abstracts:

- allocating packet objects;
- freeing packet objects;
- queueing outbound packets;
- queueing inbound packets;
- selecting the next packet.

This avoids casual heap allocation.

## 13.3 Cooperative loop

The architecture repeatedly calls:

```cpp
mesh.loop();
```

Internally the loop checks:

- whether a packet has arrived;
- whether a queued packet is ready;
- whether the channel appears available;
- whether transmission completed;
- whether budgets and delays permit another send.

## 13.4 Priorities

Different packets can have different priorities.

For example, direct routed traffic may receive higher priority than distant flood traffic.

## 13.5 Airtime budget

The dispatcher tracks how much transmission time remains in a configured window.

This helps prevent one node from monopolizing the channel.

### Checkpoint

Describe how a packet moves from:

```text
application request
```

to:

```text
outbound queue
```

to:

```text
radio transmission
```

---

# 14. Phase 11: Payloads

Important files:

- `docs/payloads.md`
- `src/Mesh.cpp`
- `src/Mesh.h`

## 14.1 Advertisements

Advertisements contain identity-related information such as:

- public key;
- timestamp;
- signature;
- optional application data.

Repeaters can forward adverts without knowing the sender's private key.

The destination or receiving application can verify the signature.

## 14.2 Direct text messages

Direct text packets contain:

- compact destination selector;
- compact source selector;
- MAC;
- encrypted data.

## 14.3 Requests and responses

Generic request and response packets allow application-defined exchanges.

Examples:

- server commands;
- sensor queries;
- feature negotiation;
- custom actions.

## 14.4 Anonymous requests

Anonymous requests can include an ephemeral sender public key.

They permit a node to receive a protected request before the sender exists in the normal contact database.

These require careful anti-spam and abuse controls at the application layer.

## 14.5 Group text

Group text uses a channel hash and a shared channel secret.

Anyone with the secret can generally authenticate and decrypt group traffic.

Group channels do not provide individual sender authentication in the same way that signed identity adverts do.

## 14.6 Group data

Group data carries application-specific binary datagrams.

## 14.7 ACK

An acknowledgement indicates receipt of a referenced packet or message identifier.

ACKs do not automatically prove that a human read a message.

## 14.8 Path payloads

Path packets communicate usable routes and optional extra data.

## 14.9 Trace

Trace packets collect path and signal information.

They can help inspect:

- repeater sequence;
- per-hop SNR;
- route behavior.

## 14.10 Raw custom packets

Raw custom packets allow applications to define their own payload handling.

They should not be treated as permission to overload reserved packet types in incompatible ways.

### Checkpoint

For each payload type, state:

- who creates it;
- who needs to understand it;
- whether repeaters decrypt it;
- whether it is flood or direct.

---

# 15. Phase 12: Identity and Cryptography

Important files:

- `src/Identity.h`
- `src/Identity.cpp`
- `src/Utils.h`
- `src/Utils.cpp`

## 15.1 Public and private keys

Each identity has:

- a public key that can be shared;
- a private key that must remain secret.

## 15.2 Signatures

The private key signs data.

The public key verifies the signature.

Advertisements use signatures to prove that the advertised content was created by the holder of the corresponding private key.

## 15.3 Shared secrets

Two identities can derive a shared secret.

That secret can be used for symmetric encryption and authentication.

## 15.4 AES-128

MeshCore's public utility code uses AES-128 block encryption.

## 15.5 HMAC-SHA-256

The encrypted data is authenticated with HMAC-SHA-256, truncated to a smaller MAC field.

This helps detect:

- modification;
- invalid keys;
- corrupted encrypted data.

## 15.6 Encrypt-then-MAC

The public utility path follows the general structure:

```text
plaintext
   ↓
AES encryption
   ↓
ciphertext
   ↓
HMAC over ciphertext
   ↓
MAC + ciphertext
```

On receipt:

```text
MAC + ciphertext
   ↓
recalculate HMAC
   ↓
compare MAC
   ↓
decrypt only when valid
```

## 15.7 Compact hashes are not identities

A one-byte source or destination hash is only a compact selector.

It can collide.

The receiver may:

1. find all contacts matching the compact hash;
2. derive or load each shared secret;
3. attempt MAC validation and decryption;
4. accept the candidate that authenticates.

### Checkpoint

Explain why knowing a one-byte contact hash is not enough to impersonate that contact.

---

# 16. Phase 13: Contacts, Channels, and Local Databases

## 16.1 Contacts

A contact record may include:

- public key;
- display name;
- direct path;
- last-seen time;
- shared-secret cache;
- metadata;
- permissions.

## 16.2 Channels

A channel record may include:

- name;
- channel secret;
- compact channel hash;
- message history;
- unread status.

## 16.3 Seen-packet table

A seen table stores identifiers for recently handled packets.

It supports:

- duplicate suppression;
- flood control;
- ACK suppression;
- avoiding repeated processing.

## 16.4 Path storage

Applications can store one or more paths associated with contacts.

Path choice may be based on:

- recency;
- success;
- hop count;
- SNR;
- manual preference;
- observed reliability.

### Checkpoint

Design a JavaScript-like contact object that could represent the same logical information, then identify which fields would need fixed-size C++ storage.

---

# 17. Phase 14: Companion Protocol

Important file:

- `docs/companion_protocol.md`

This is the best bridge from your web-development experience into embedded networking.

## 17.1 Topics to learn

- BLE;
- GATT;
- services;
- characteristics;
- write operations;
- notifications;
- USB serial;
- binary framing;
- command IDs;
- responses;
- asynchronous push events;
- reconnect behavior;
- synchronization.

## 17.2 Command-response model

A companion application may:

1. send a binary command;
2. wait for a binary response;
3. receive unsolicited push events later.

This resembles a combination of:

- REST requests;
- WebSocket events;
- device RPC.

## 17.3 BLE GATT model

A BLE peripheral exposes:

- services;
- characteristics;
- read/write behavior;
- notifications.

The phone or browser acts as the central client.

## 17.4 Synchronization

After reconnecting, the application may need to synchronize:

- device information;
- contacts;
- channels;
- messages;
- pending status;
- configuration.

## 17.5 JavaScript advantage

A JavaScript companion library can help you understand the protocol before reading the C++ firmware because:

- typed arrays are familiar;
- promises and events are familiar;
- UI state is familiar;
- command parsing is easier to inspect.

### Checkpoint

Describe how a typed-array command from a browser becomes a radio packet.

---

# 18. Phase 15: Room Servers and Store-and-Forward

A room server is not just a repeater with storage.

## 18.1 Core concepts

Learn:

- message identifiers;
- server-side retention;
- pagination;
- retrieval requests;
- duplicate prevention;
- expiration;
- synchronization;
- authentication;
- authorization;
- replay behavior.

## 18.2 Live group versus stored room

| Live group channel | Room server |
|---|---|
| Broadcast at send time | Stores messages |
| Offline clients may miss it | Clients can retrieve history |
| Shared channel secret | Server-specific request/response flow |
| No central message history | Centralized retained history |

## 18.3 BBS mental model

A room server is closer to:

- a small bulletin board;
- a mailbox;
- a message API over LoRa.

It is not equivalent to an always-connected cloud chat service.

### Checkpoint

Explain the sequence an offline client would use to retrieve missed room messages.

---

# 19. Phase 16: Repeater Design and Network Health

## 19.1 Stable infrastructure

Good repeaters are typically:

- fixed;
- elevated;
- weather protected;
- continuously powered;
- accurately configured;
- placed to cover meaningful geographic gaps.

## 19.2 Why mobile repeaters are problematic

A moving repeater can:

- appear in learned paths;
- move out of range;
- break future direct messages;
- create unpredictable coverage;
- generate unstable topology.

## 19.3 Too many repeaters

More repeaters are not automatically better.

Too many overlapping repeaters can create:

- duplicate receptions;
- collision pressure;
- excessive flooding;
- more airtime use;
- path ambiguity.

## 19.4 Repeater forwarding controls

Understand:

- forwarding eligibility;
- duplicate tables;
- retransmission delays;
- path limits;
- transport filters;
- airtime budget;
- local network policy.

### Checkpoint

Evaluate whether a proposed repeater location improves:

- coverage;
- reliability;
- capacity;
- topology stability.

---

# 20. Phase 17: Reliability and Acknowledgements

## 20.1 What an ACK means

An ACK usually means a particular packet or message was received and processed far enough to generate a response.

It does not necessarily mean:

- the user saw it;
- the message was stored permanently;
- every route worked;
- the destination UI displayed it.

## 20.2 ACK path

ACKs may return through direct paths and can be repeated according to implementation rules.

## 20.3 Multi-ACK

Multipart or multi-ACK behavior can improve reliability or communicate multiple acknowledgement attempts.

## 20.4 Retries

Application retry logic must avoid creating duplicate messages or unnecessary airtime.

Useful concepts:

- idempotency;
- message IDs;
- retry counters;
- retry delay;
- exponential backoff;
- duplicate detection.

### Checkpoint

Design a retry policy for a direct text message that avoids immediate flood spam.

---

# 21. Phase 18: Security and Abuse Resistance

Cryptography alone does not stop abuse.

## 21.1 Threats to study

- spam;
- stalking;
- unsolicited requests;
- replay attacks;
- malicious flooding;
- forged metadata;
- hash collisions;
- identity scraping;
- location leakage;
- denial of service;
- compromised channel secrets;
- stolen private keys.

## 21.2 Application-layer controls

Possible controls include:

- contact approval;
- request rate limits;
- block lists;
- zero-hop restrictions;
- capability negotiation;
- expiry timestamps;
- replay caches;
- user confirmation;
- per-feature permissions;
- silent rejection;
- anti-amplification limits.

## 21.3 Anonymous requests

Anonymous or first-contact requests are useful but especially sensitive.

A safe design should consider:

- how often one identity can request;
- how the receiver identifies repeated abuse;
- whether requests are forwarded;
- whether they trigger automatic responses;
- whether they reveal location or presence;
- whether the receiver must approve continuation.

## 21.4 Metadata

Even when payloads are encrypted, observers may still infer:

- packet timing;
- route length;
- repeated contacts;
- traffic volume;
- approximate source region;
- advert identity;
- repeater activity.

### Checkpoint

Threat-model a “friend request over MeshCore” feature.

Identify:

- assets;
- attackers;
- abuse cases;
- mitigations;
- remaining risks.

---

# 22. Phase 19: Reading the Public Codebase

Read the repository in this order.

## Step 1: Packet structure

1. `src/Packet.h`
2. `src/Packet.cpp`
3. `docs/packet_format.md`

Goal:

- understand the wire unit;
- decode route and payload types;
- understand path encoding.

## Step 2: Payload definitions

4. `docs/payloads.md`

Goal:

- know what each payload means;
- understand encrypted and unencrypted portions.

## Step 3: Identity and crypto

5. `src/Identity.h`
6. `src/Identity.cpp`
7. `src/Utils.h`
8. `src/Utils.cpp`

Goal:

- understand keys;
- signatures;
- shared secrets;
- encryption;
- MAC validation.

## Step 4: Radio dispatch

9. `src/Dispatcher.h`
10. `src/Dispatcher.cpp`

Goal:

- understand queues;
- CAD;
- delays;
- priorities;
- radio state;
- airtime budgets.

## Step 5: Mesh routing

11. `src/Mesh.h`
12. `src/Mesh.cpp`

Goal:

- understand receive handling;
- flood forwarding;
- direct forwarding;
- ACKs;
- paths;
- payload dispatch.

## Step 6: Chat and application abstraction

13. `src/helpers/BaseChatMesh.h`
14. `src/helpers/BaseChatMesh.cpp`

Goal:

- understand contacts;
- channels;
- message handling;
- higher-level callbacks.

## Step 7: Examples

15. `examples/simple_secure_chat`
16. `examples/simple_repeater`
17. `examples/simple_room_server`
18. `examples/companion_radio`
19. `examples/kiss_modem`

Goal:

- see how different roles subclass and configure the same core.

---

# 23. Architecture Map

```text
Application or firmware role
        ↓
BaseChatMesh or specialized Mesh subclass
        ↓
Mesh
  routing
  payload dispatch
  crypto integration
  ACKs
  path handling
        ↓
Dispatcher
  queues
  scheduling
  CAD
  send/receive state
  airtime budget
        ↓
Radio abstraction
        ↓
RadioLib or platform-specific implementation
        ↓
SX1262 hardware
```

Parallel storage and identity services:

```text
Contact database
Channel database
Seen-packet table
Path storage
Packet pool
Clock
Random-number generator
```

---

# 24. Topics to Learn Later

These are useful, but not required for understanding the public MeshCore protocol.

## 24.1 FreeRTOS

Learn later when working on:

- multiple independent tasks;
- queue synchronization;
- semaphores;
- mutexes;
- core pinning;
- high-rate UI or sensor processing.

## 24.2 LVGL

Learn later when building a standalone graphical interface.

Topics:

- object tree;
- events;
- styles;
- display buffers;
- input devices;
- screen navigation;
- partial rendering;
- SPI flushing.

## 24.3 GPS

Learn later for:

- NMEA;
- latitude and longitude;
- fix quality;
- cold start;
- stale position;
- time synchronization;
- power management.

## 24.4 Offline maps

Learn later for:

- Web Mercator;
- slippy-map tiles;
- zoom levels;
- tile indexing;
- SD-card layout;
- caching;
- image decoding.

## 24.5 MQTT and internet bridges

Learn later if building:

- regional gateways;
- internet relays;
- message mirrors;
- dashboards;
- remote telemetry services.

## 24.6 AX.25 and amateur packet radio

Learn later if using:

- KISS modem mode;
- external terminal-node software;
- APRS-like tools;
- amateur radio integrations.

---

# 25. Recommended Study Order

Use this sequence:

```text
Embedded C++ essentials
        ↓
Bits and bytes
        ↓
Packet header
        ↓
Payload types
        ↓
Flood routing
        ↓
Direct source routing
        ↓
Identity and encryption
        ↓
Dispatcher and airtime
        ↓
Companion protocol
        ↓
Room servers
        ↓
Advanced RF
        ↓
Optional UI, GPS, maps, and FreeRTOS
```

Do not begin with:

- FreeRTOS internals;
- LVGL internals;
- MQTT bridges;
- advanced cryptographic mathematics;
- the full SX1262 register map.

Those topics create cognitive fog before the packet model is stable.

---

# 26. Suggested Study Sessions

## Session 1: C++ byte literacy

Learn:

- fixed-width integers;
- arrays;
- pointers;
- lengths;
- `memcpy`;
- masks;
- shifts.

Exercise:

Decode several artificial header bytes.

## Session 2: Packet anatomy

Read:

- `Packet.h`;
- packet-format documentation.

Exercise:

Draw a packet from memory.

## Session 3: Flood routing

Read:

- flood-related sections of `Mesh.cpp`.

Exercise:

Trace one flood across three repeaters.

## Session 4: Direct routing

Read:

- direct-routing sections of `Mesh.cpp`.

Exercise:

Show how each repeater removes its path entry.

## Session 5: Payloads

Read:

- `payloads.md`.

Exercise:

Classify ten example actions by payload type.

## Session 6: Identity and crypto

Read:

- `Identity.cpp`;
- `Utils.cpp`.

Exercise:

Draw encrypt-then-MAC and signature verification.

## Session 7: Dispatcher

Read:

- `Dispatcher.h`;
- `Dispatcher.cpp`.

Exercise:

Follow a queued packet through CAD and transmission.

## Session 8: Companion protocol

Read:

- `companion_protocol.md`.

Exercise:

Map device commands to a browser application's state.

## Session 9: Repeater and room-server examples

Compare:

- simple repeater;
- simple room server;
- secure chat.

Exercise:

Identify what each subclass overrides.

## Session 10: Your own feature design

Design one feature such as:

- friend request;
- path return;
- forced discovery;
- direct ACK;
- presence request.

Document:

- payload type;
- route type;
- encryption;
- abuse controls;
- retry behavior;
- backward compatibility.

---

# 27. Practical Exercises

## Exercise 1: Header decoder

Write a JavaScript function that accepts one byte and returns:

```js
{
  version,
  payloadType,
  routeType
}
```

## Exercise 2: Packet visualizer

Build a small browser page that displays:

- header byte;
- path length;
- path hashes;
- payload bytes;
- decoded type names.

## Exercise 3: Airtime comparison

Create a table comparing several combinations of:

- SF;
- BW;
- CR;
- payload length.

Focus on relative airtime rather than perfect RF simulation.

## Exercise 4: Flood simulator

Model:

- five repeaters;
- random retransmission delays;
- duplicate suppression;
- multiple arrival paths.

## Exercise 5: Direct route simulator

Represent a route as:

```js
["A1", "B2", "C3"]
```

Simulate each repeater removing the first matching element.

## Exercise 6: Contact-hash collision

Create two fake identities that share the same one-byte selector.

Simulate trying multiple shared secrets until one MAC validates.

## Exercise 7: Abuse-resistant friend request

Specify:

- request payload;
- route;
- identity;
- rate limit;
- replay protection;
- receiver confirmation;
- block behavior.

---

# 28. Questions You Should Eventually Be Able to Answer

## RF

- What is the difference between RSSI and SNR?
- Why does higher spreading factor increase airtime?
- Why can a strong RSSI still produce poor decoding?
- What is a hidden node?
- What does CAD detect?

## Packets

- How are route type and payload type stored?
- Why is path hash size configurable?
- What is the maximum path tradeoff?
- What portions of a packet can repeaters inspect?
- What portions remain encrypted?

## Routing

- How does a flood path grow?
- How does a direct path shrink?
- Why do clients not normally repeat?
- Why do stable repeaters matter?
- How are duplicates suppressed?

## Security

- What is signed?
- What is encrypted?
- What is authenticated?
- Why is a compact hash not an identity?
- What can an observer infer from metadata?

## Software

- What does `Dispatcher` do?
- What does `Mesh` do?
- What does a firmware subclass add?
- Why are packet pools used?
- How does a companion command become a LoRa packet?

---

# 29. Core Mental Model

When reading any MeshCore code, ask these questions in order:

1. **What role is this device playing?**
2. **What packet type is being handled?**
3. **Is the route flood, direct, transport, or zero-hop?**
4. **Which bytes are routing metadata?**
5. **Which bytes are encrypted?**
6. **Which identity or channel secret authenticates the payload?**
7. **Should this node consume, forward, hold, or discard it?**
8. **Has this packet already been seen?**
9. **When is it scheduled to transmit?**
10. **How much airtime will it consume?**
11. **What callback receives the decoded result?**
12. **What state is saved for future packets?**

This sequence turns a dense C++ file into a business-logic flow.

---

# 30. Glossary

## ACK

Acknowledgement indicating receipt of a referenced packet or message.

## Advert

Signed identity announcement containing a public key and application metadata.

## AES

Symmetric block cipher used to encrypt payload data.

## Airtime

The amount of time a packet occupies the radio channel.

## Bandwidth

The width of the LoRa signal channel.

## CAD

Channel Activity Detection.

## Coding Rate

Error-correction redundancy used by LoRa.

## Companion

A radio device controlled by an external phone, browser, or computer.

## Direct Route

A source route supplied inside the packet.

## Dispatcher

The layer managing packet queues, radio state, scheduling, CAD, and airtime.

## Flood

Routing mode in which eligible repeaters rebroadcast a packet.

## HMAC

Keyed authentication mechanism used to verify integrity and authenticity.

## Identity

Public/private key pair used for signatures and shared-secret calculation.

## KISS

A simple framing protocol for connecting packet-radio modems to external software.

## MAC

Message Authentication Code, not a network hardware address in this context.

## Mesh

The protocol layer that handles routes, payload types, decryption, and callbacks.

## MTU

Maximum Transmission Unit.

## Noise Floor

Background radio energy present in a channel.

## Packet Hash

Compact identifier used for deduplication or acknowledgement references.

## Path Hash

Compact repeater identity fragment stored in a route.

## Repeater

Infrastructure node that forwards eligible packets.

## Room Server

Node that stores and serves message history.

## RSSI

Received Signal Strength Indicator.

## SNR

Signal-to-Noise Ratio.

## Source Routing

Routing method where the sender supplies the route.

## Spreading Factor

LoRa parameter affecting symbol duration, sensitivity, throughput, and airtime.

## Transport Code

Additional routing metadata used for scoped transport behavior.

## Zero-Hop

Transmission intended only for directly reachable neighbors.

---

# 31. Source References

## MeshCore repository

- https://github.com/meshcore-dev/MeshCore

## Important source files

- https://github.com/meshcore-dev/MeshCore/blob/main/src/Packet.h
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Packet.cpp
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Mesh.h
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Mesh.cpp
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Dispatcher.h
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Dispatcher.cpp
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Identity.h
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Identity.cpp
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Utils.h
- https://github.com/meshcore-dev/MeshCore/blob/main/src/Utils.cpp
- https://github.com/meshcore-dev/MeshCore/blob/main/src/helpers/BaseChatMesh.h
- https://github.com/meshcore-dev/MeshCore/blob/main/src/helpers/BaseChatMesh.cpp

## Documentation

- https://docs.meshcore.io/packet_format/
- https://docs.meshcore.io/payloads/
- https://docs.meshcore.io/companion_protocol/
- https://docs.meshcore.io/kiss_modem_protocol/
- https://docs.meshcore.io/faq/

## Examples

- https://github.com/meshcore-dev/MeshCore/tree/main/examples/simple_secure_chat
- https://github.com/meshcore-dev/MeshCore/tree/main/examples/simple_repeater
- https://github.com/meshcore-dev/MeshCore/tree/main/examples/simple_room_server
- https://github.com/meshcore-dev/MeshCore/tree/main/examples/companion_radio
- https://github.com/meshcore-dev/MeshCore/tree/main/examples/kiss_modem

---

# 32. Final Recommendation

Begin with this path:

```text
C++ byte handling
    ↓
Packet header
    ↓
Payload format
    ↓
Flood route
    ↓
Direct route
    ↓
Identity and encryption
    ↓
Dispatcher and airtime
```

Do not try to learn every radio, embedded, UI, cryptographic, and mapping topic simultaneously.

The fastest route to understanding MeshCore is to follow one packet from creation to final delivery, then repeat the exercise for each route and payload type.

---

# 33. First Review Prompt

Without looking back, explain:

1. what the final two bits in `0bVVPPPPRR` represent;
2. how a flood path changes at each repeater;
3. how a direct path changes at each repeater;
4. why a one-byte identity hash can collide;
5. why AES encryption alone is not enough without authentication.

