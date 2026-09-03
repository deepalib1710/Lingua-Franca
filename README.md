# Lingua-Franca: Reactor-Oriented Temporal Semantics & NTP

A example that demonstrates how the Lingua Franca reactor model can be used to implement an NTP-inspired clock synchronization system with an explicit network-delay emulator.

## Overview

This repository contains a Python-target Lingua Franca example (NTP_System.lf) that shows how reactor-oriented programs reason about time, scheduling, and message delays in a distributed setting.

In addition to the original NTP_System example, this repository now includes three extended scenarios that explore bandwidth throttling, multipath routing, and partial network failures. Each scenario reuses the reactor-based architecture (Client, Server, Network) while modifying the network model to demonstrate different temporal and reliability behaviors.

Key features demonstrated:

- Temporal reasoning: tracking and comparing local vs. network (server) time
- Asynchronous communication: event-driven reactor interactions
- Distributed coordination: multi-reactor systems communicating via ports
- Network emulation: per-link bandwidth-based packet delay calculation
- Clock synchronization: a simple NTP-like request/response flow

## Examples in this repository

### 1) NTP_System.lf (base example)

The original NTP system demonstrates the basic flow of an NTP-like request/response with a network reactor that computes per-hop delays based on a fixed packet size and link bandwidths.

See the Architecture, Server, Client, and Network sections below for details on this example.

### 2) NTP_BandwidthThrottling.lf

Purpose:

- Demonstrates how limited link bandwidth affects packet serialization delays and end-to-end latency.
- Shows the effects of different bandwidth values on measured clock skew and scheduling.

What changes compared to the base example:

- Link bandwidths can be changed to low values to simulate throttled links.
- The network reactor may compute larger per-hop delays due to lower bandwidth, demonstrating how serialization delay contributes to total latency.

### 3) NTP_MultipathRouting.lf

Purpose:

- Demonstrates routing a request/response over multiple possible paths and aggregating delays from a chosen path.
- Explores how alternate paths with different bandwidths affect delivered latency and jitter.

What changes compared to the base example:

- The Network reactor can select between multiple paths for forwarding a packet (client → server and server → client).
- Path selection logic can be deterministic or randomized to emulate load balancing.

### 4) NTP_PartialNetworkFailure.lf

Purpose:

- Models flaky links and node outages, demonstrating how partial failures affect message delivery and clock synchronization.
- Shows strategies for handling missing responses or delayed retransmissions within a reactor model.

What changes compared to the base example:

- The Network reactor can drop packets or temporarily mark links as down.
- The Client reactor may implement simple retry logic or timeout handling in response to missing responses.

## The NTP System Example

### Architecture

The system implements three reactor types and a `main` that wires them together:

- Server: receives requests and responds with the current system time.
- Client: schedules a logical action on startup to send an NTP request and computes the difference between the local clock and the server time on response.
- Network: emulates a path between client and server and computes per-link delays based on a fixed packet size and link bandwidths; it forwards requests and responses with the computed delay (fractional delays are preserved when the runtime supports them).

### Server Reactor

Responsibilities:

- Listen for incoming `ntp_request` events
- On request, read `time.time()`, print debug messages, and set `ntp_response` to that timestamp

Behavioral notes from `NTP_System.lf`:

- The server prints a message when it receives a request and another showing the server time.

Example (from runtime):

Server: NTP request received.
Server: server time: 1626180000.123456

### Client Reactor

Responsibilities:

- On startup, schedule a logical action `send_request` at logical time 0
- `send_request` raises `ntp_request`
- On receiving `ntp_response`, compute local time minus server time and print the skew

Behavioral notes from `NTP_System.lf`:

- The client uses `time.time()` (aliased as `phyTime`) to get local time when the response arrives
- The printed output uses 4 decimal places, e.g.:

Local clock is ahead by 0.0123 seconds.

### Network Reactor (emulator)

Responsibilities:

- Emulate packet forwarding from client → server and server → client
- Compute per-link delay as `packet_size / bandwidth` for each hop
- Sum per-hop delays to obtain `total_delay` and schedule forwarding after `total_delay` seconds (fractional delays are preserved)
- Print per-link delays and the total delay

Implementation details from `NTP_System.lf`:

- The packet size used in the example is 20 (bytes; unitless in this simple model)
- Links for Client → Server are defined as:
  - ("Client", "Router", 40)
  - ("Router", "Node1", 50)
  - ("Node1", "Node2", 20)
  - ("Node2", "Server", 25)

  Path: ["Client", "Router", "Node1", "Node2", "Server"]

- Links for Server → Client are defined as:
  - ("Router", "Client", 40)
  - ("Node1", "Router", 50)
  - ("Node2", "Node1", 20)
  - ("Server", "Node2", 25)

  Path: ["Server", "Node2", "Node1", "Router", "Client"]

- Each per-hop delay is printed (e.g. `Client -> Router : 0.500 sec`) and the total is printed as well.
- The code schedules forwarding with `fwd_request.schedule(total_delay, c_request_in.value)` (and similarly for responses), which preserves fractional delays when the runtime supports floating-point logical times.

Example runtime lines from the network:

Client -> Router : 0.500 sec
Router -> Node1 : 0.400 sec
Node1 -> Node2 : 1.000 sec
Node2 -> Server : 0.800 sec
Total Client → Server delay: 2.700 sec
Network: delivering request to server

Network: delivered response to client

## How it works (end-to-end)

1. `main` creates Server, Client, and Network reactors and wires ports:
   - `c.ntp_request -> n.c_request_in`
   - `n.c_request_out -> s.ntp_request`
   - `s.ntp_response -> n.s_response_in`
   - `n.s_response_out -> c.ntp_response`
2. On startup, Client schedules `send_request` at logical time 0
3. `send_request` raises `ntp_request` which goes into the `Network` reactor
4. `Network` computes per-link delays, prints them, sums them, and schedules `fwd_request` after `total_delay` seconds
5. `Network` forwards the request to the Server; Server prints the receipt and server time and sets `ntp_response`
6. `s_response_in` on the Network triggers the response-forwarding path (with its own per-link delays)
7. When the Client receives `ntp_response`, it computes localTime - server_time and prints the clock skew

## Getting Started

### Prerequisites

- Lingua Franca compiler (visit https://lf-lang.org)
- Python 3.7+ (example target)

### Run the examples (Python target)

## Compile the LF source to Python
lfc NTP_System.lf

## Run the generated Python program
python src-gen/NTP_System.py

Replace `NTP_System.lf` above with any of the example sources to compile and run the other scenarios (e.g. `NTP_BandwidthThrottling.lf`, `NTP_MultipathRouting.lf`, `NTP_PartialNetworkFailure.lf`). The generated Python files will be created under `src-gen/` with matching base names.

Example:

```bash
# Compile bandwidth throttling scenario
lfc NTP_BandwidthThrottling.lf
python src-gen/NTP_BandwidthThrottling.py
```

## Expected output (illustrative)

```
Server: NTP request received.
Server: server time: 1626180000.1234
Client -> Router : 0.500 sec
Router -> Node1 : 0.400 sec
Node1 -> Node2 : 1.000 sec
Node2 -> Server : 0.800 sec
Total Client → Server delay: 2.700 sec
Network: delivering request to server
Network: delivered response to client
Local clock is ahead by 0.0123 seconds.
```

The network emulator preserves fractional delays when the runtime supports them; scheduling uses the computed `total_delay` (float).

## Experiments and Extensions

These examples were designed to be small, modifiable experiments. Some ideas:

- Multiple clients: instantiate more client reactors to measure aggregate effects on the network and server.
- More realistic per-packet serialization and propagation models: add propagation latency or variable packet sizes.
- Introducing jitter, packet loss, or variable bandwidth: the `NTP_PartialNetworkFailure.lf` example already shows packet drops/failures; try adding stochastic jitter to per-hop delays.
- Multipath selection strategies: in `NTP_MultipathRouting.lf`, try deterministic vs. probabilistic path selection and measure variance in measured skew.
- Compiling to other targets supported by Lingua Franca: Java, C, etc.

## Files

- NTP_System.lf — Base example demonstrating the simple NTP-like exchange and delay emulator.
- NTP_BandwidthThrottling.lf — Variant demonstrating the effect of limited bandwidth on end-to-end delay.
- NTP_MultipathRouting.lf — Variant demonstrating multiple alternate paths between client and server.
- NTP_PartialNetworkFailure.lf — Variant that models packet loss and flaky links/nodes.

## Resources

- Lingua Franca official repo: https://github.com/lf-lang/lingua-franca
- Language docs and tutorials: https://lf-lang.org/docs
- Research paper (in preparation): Deepali Banka and Anupam Chattopadhyay, "From Imperative to Reactive Time-Synchronized Systems: An NTP Case Study" (manuscript fully written; email for a private draft copy).
---
