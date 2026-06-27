# DPI Engine — Deep Packet Inspection System

A C++17 deep packet inspection engine that parses raw network traffic, classifies connections by application (YouTube, Facebook, etc.) using TLS SNI extraction, and applies rule-based blocking — available in both single-threaded and multi-threaded versions.

## How It Works

```
PCAP Input → Parse Headers → Extract SNI → Classify App → Apply Rules → PCAP Output
```

1. **Parse** raw Ethernet/IP/TCP headers from captured packets
2. **Extract** the domain name (SNI) from TLS Client Hello messages — even though HTTPS is encrypted, the destination domain is sent in plaintext during the handshake
3. **Classify** each connection (flow) by application based on its SNI
4. **Filter** flows against configurable rules (block by IP, app, or domain)
5. **Report** traffic breakdown and blocked connections

## Architecture

**Simple version** (`main_working.cpp`) — single-threaded, processes packets sequentially. Good for learning and small captures.

**Multi-threaded version** (`dpi_mt.cpp`) — a producer-consumer pipeline for high-throughput processing:

```
Reader Thread → Load Balancer Threads → Fast Path Threads → Output Writer Thread
```

- Packets are hashed by 5-tuple (src IP, dst IP, src port, dst port, protocol) and routed through load balancers to fast-path workers
- Consistent hashing ensures all packets of the same connection are processed by the same worker, preserving flow state
- Custom thread-safe queues (mutex + condition variables) connect each stage

## Features

- Raw packet parsing (Ethernet, IPv4, TCP/UDP)
- TLS SNI and HTTP Host header extraction
- Flow tracking via 5-tuple hashing
- Rule-based blocking by IP, application, or domain
- Multi-threaded pipeline for scalable processing
- Traffic classification reports with per-thread statistics

## Build

**Simple version:**
```bash
g++ -std=c++17 -O2 -I include -o dpi_simple \
    src/main_working.cpp src/pcap_reader.cpp \
    src/packet_parser.cpp src/sni_extractor.cpp src/types.cpp
```

**Multi-threaded version:**
```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp src/pcap_reader.cpp \
    src/packet_parser.cpp src/sni_extractor.cpp src/types.cpp
```

## Usage

```bash
# Basic
./dpi_engine input.pcap output.pcap

# With blocking rules
./dpi_engine input.pcap output.pcap \
    --block-app YouTube \
    --block-domain facebook \
    --block-ip 192.168.1.50

# Configure thread pool (multi-threaded only)
./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

Generate sample test data:
```bash
python3 generate_test_pcap.py
```

## Sample Output

```
╔════════════════════════════════════════════╗
║          PROCESSING REPORT                  ║
╠════════════════════════════════════════════╣
║ Total Packets:        77                    ║
║ Forwarded:            69                    ║
║ Dropped:               8                    ║
╠════════════════════════════════════════════╣
║ HTTPS          39   50.6%                   ║
║ YouTube         4    5.2%  (BLOCKED)         ║
║ Facebook        3    3.9%                   ║
╚════════════════════════════════════════════╝
```

## Project Structure

```
packet_analyzer/
├── include/        # Header files (types, parser, SNI extractor, thread classes)
├── src/            # Implementation files
│   ├── main_working.cpp   # Single-threaded version
│   └── dpi_mt.cpp          # Multi-threaded version
├── generate_test_pcap.py  # Test data generator
└── test_dpi.pcap           # Sample capture
```

## Key Concepts

- **5-Tuple**: (src IP, dst IP, src port, dst port, protocol) — uniquely identifies a connection
- **SNI (Server Name Indication)**: The domain name sent in plaintext during the TLS handshake, before encryption begins
- **Flow-based blocking**: Once a flow is classified and matched against a rule, all subsequent packets in that flow are dropped

## Requirements

- C++17 compiler (g++ or clang++)
- pthread (for multi-threaded version)
- No external libraries
