# Multi-Threaded Deep Packet Inspection Engine

A **C++17 Deep Packet Inspection (DPI) engine** for processing PCAP network captures, tracking flows, classifying application traffic, extracting TLS SNI / HTTP host information, and applying rule-based traffic filtering.

The project includes both a simple single-threaded implementation for learning and a multi-threaded processing pipeline designed for larger packet captures.

---

## Features

- Parses **PCAP** packet captures directly
- Extracts **Ethernet, IPv4, TCP, and UDP** information
- Tracks network flows using the standard **5-tuple**
  - Source IP
  - Destination IP
  - Source Port
  - Destination Port
  - Protocol
- Performs **Deep Packet Inspection**
- Extracts **TLS Server Name Indication (SNI)**
- Extracts **HTTP Host** information
- Classifies traffic into application categories
- Supports blocking by:
  - IP address
  - Application
  - Domain / SNI
- Maintains **flow-level blocking state**
- Provides configurable **multi-threaded packet processing**
- Generates packet, thread, application, and domain statistics
- Writes allowed packets to a filtered output PCAP

---

## Architecture

The multi-threaded engine follows a staged packet-processing architecture:

```text
                     +------------------+
                     |    Input PCAP    |
                     +---------+--------+
                               |
                               v
                     +------------------+
                     |   Reader Thread  |
                     +---------+--------+
                               |
                  +------------+------------+
                  |                         |
                  v                         v
          +---------------+         +---------------+
          | Load Balancer |   ...   | Load Balancer |
          +-------+-------+         +-------+-------+
                  |                         |
            hash 5-tuple              hash 5-tuple
                  |                         |
          +-------+-------+         +-------+-------+
          |               |         |               |
          v               v         v               v
      +--------+       +--------+ +--------+       +--------+
      | Fast   |       | Fast   | | Fast   |       | Fast   |
      | Path   |       | Path   | | Path   |       | Path   |
      +---+----+       +---+----+ +---+----+       +---+----+
          |                |          |                |
          +----------------+----------+----------------+
                               |
                               v
                     +------------------+
                     |  Filtered PCAP   |
                     +------------------+
```

Packets belonging to the same connection are routed consistently using the **5-tuple**, allowing flow state to remain coherent while traffic is processed in parallel.

---

## Packet Processing Pipeline

```text
PCAP Packet
    |
    v
Ethernet Header
    |
    v
IPv4 Header
    |
    +-------------------+
    |                   |
    v                   v
   TCP                 UDP
    |
    v
Application Payload
    |
    +------------------------+
    |                        |
    v                        v
TLS Client Hello          HTTP Request
    |                        |
    v                        v
Extract SNI             Extract Host
    |                        |
    +------------+-----------+
                 |
                 v
        Application Classification
                 |
                 v
          Rule Evaluation
                 |
          +------+------+
          |             |
          v             v
       FORWARD         DROP
```

---

## Flow Tracking

Each connection is identified using a **5-tuple**:

```text
(Source IP, Destination IP, Source Port, Destination Port, Protocol)
```

Example:

```text
192.168.1.100:54321 -> 172.217.14.206:443 / TCP
```

Packets with the same tuple belong to the same flow.

Once a flow is classified or blocked, that state can be reused for subsequent packets instead of repeating all inspection work.

---

## TLS SNI Inspection

For HTTPS connections, the engine inspects the **TLS Client Hello** and extracts the **Server Name Indication (SNI)** when available.

Example:

```text
TLS Client Hello
└── Extensions
    └── SNI
        └── www.youtube.com
```

The extracted hostname can then be mapped to an application category or checked against domain-blocking rules.

---

## Rule-Based Filtering

The engine supports three main filtering mechanisms.

### Block an application

```bash
--block-app YouTube
```

### Block an IP address

```bash
--block-ip 192.168.1.50
```

### Block a domain

```bash
--block-domain facebook
```

Blocking is maintained at the **flow level**. Once a connection is identified as blocked, later packets belonging to that flow are dropped.

---

## Project Structure

```text
.
├── include/
│   ├── connection_tracker.h
│   ├── dpi_engine.h
│   ├── fast_path.h
│   ├── load_balancer.h
│   ├── packet_parser.h
│   ├── pcap_reader.h
│   ├── rule_manager.h
│   ├── sni_extractor.h
│   ├── thread_safe_queue.h
│   └── types.h
│
├── src/
│   ├── dpi_mt.cpp
│   ├── main_working.cpp
│   ├── packet_parser.cpp
│   ├── pcap_reader.cpp
│   ├── sni_extractor.cpp
│   ├── types.cpp
│   └── ...
│
├── CMakeLists.txt
├── WINDOWS_SETUP.md
├── generate_test_pcap.py
├── test_dpi.pcap
├── output.pcap
└── README.md
```

---

## Requirements

For macOS or Linux:

- A **C++17** compatible compiler
- `g++` or `clang++`
- POSIX thread support for the multi-threaded build
- Python 3 only if you want to generate the sample PCAP

The core C++ implementation does not require an external packet-processing library.

---

## Build

### Multi-Threaded Version

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

### Simple Single-Threaded Version

```bash
g++ -std=c++17 -O2 -I include -o dpi_simple \
    src/main_working.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

For Windows-specific setup instructions, see [`WINDOWS_SETUP.md`](WINDOWS_SETUP.md).

---

## Usage

### Basic packet processing

```bash
./dpi_engine test_dpi.pcap output.pcap
```

### Apply filtering rules

```bash
./dpi_engine test_dpi.pcap output.pcap \
    --block-app YouTube \
    --block-app TikTok \
    --block-ip 192.168.1.50 \
    --block-domain facebook
```

### Configure worker threads

```bash
./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

This configuration creates:

```text
4 Load Balancer threads
x
4 Fast Path workers per Load Balancer
=
16 Fast Path processing workers
```

---

## Generate Test Traffic

A Python script is included for generating sample test traffic:

```bash
python3 generate_test_pcap.py
```

It creates a test PCAP that can be passed to the DPI engine.

Then run:

```bash
./dpi_engine test_dpi.pcap output.pcap
```

---

## Output

The engine reports information such as:

```text
DPI ENGINE (Multi-threaded)

Total Packets
Total Bytes
TCP Packets
UDP Packets

Forwarded Packets
Dropped Packets

Thread Statistics
Application Breakdown
Detected Domains / SNIs
```

Example application classifications can include traffic such as:

```text
HTTPS
DNS
YouTube
Facebook
Google
GitHub
Unknown
```

The filtered packets are written to the output PCAP file supplied on the command line.

---

## Concurrency Design

The multi-threaded implementation demonstrates several systems-programming concepts:

- **Producer-consumer architecture**
- **Thread-safe queues**
- **Mutex-based synchronization**
- **Condition variables**
- **Parallel worker pipelines**
- **Flow-aware packet distribution**
- **Hash-based load balancing**

A key design goal is to parallelize packet processing without losing per-connection state.

---

## Core Concepts Demonstrated

This project covers:

- Computer Networks
- Deep Packet Inspection
- Packet Parsing
- Stateful Flow Tracking
- TLS Handshake Inspection
- Application-Layer Traffic Classification
- Rule-Based Network Filtering
- C++ Systems Programming
- Multithreading
- Synchronization
- Producer-Consumer Design
- Hash-Based Load Balancing

---

## Possible Extensions

Potential improvements include:

- Live packet capture from a network interface
- IPv6 support
- QUIC / HTTP/3 traffic inspection
- Persistent rule configuration using JSON or YAML
- Flow timeout and connection cleanup
- Additional application signatures
- Real-time statistics dashboard
- Throughput and latency benchmarking
- Automated unit and integration tests
- CI/CD with GitHub Actions

---

## Repository

**Multi-Threaded Deep Packet Inspection Engine**

Built with **C++17** for learning and experimenting with packet processing, network security, concurrency, and Deep Packet Inspection.
