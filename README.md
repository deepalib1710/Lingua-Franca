# Lingua-Franca: Reactor-Oriented Temporal Semantics & NTP

A example that demonstrates how the Lingua Franca reactor model can be used to implement an NTP-inspired clock synchronization system with an explicit network-delay emulator.

## Overview

This repository contains a Python-target Lingua Franca example (NTP_System.lf) that shows how reactor-oriented programs reason about time, scheduling, and message delays in a distributed setting.

Key features demonstrated:

- Temporal reasoning: tracking and comparing local vs. network (server) time
- Asynchronous communication: event-driven reactor interactions
- Distributed coordination: multi-reactor systems communicating via ports
- Network emulation: per-link bandwidth-based packet delay calculation
- Clock synchronization: a simple NTP-like request/response flow

## The NTP System Example

### Architecture

The system implements three reactor types and a `main` that wires them together:

- Server: receives requests and responds with the current system time.
- Client: schedules a logical action on startup to send an NTP request and computes the difference between the local clock and the server time on response.
- Network: emulates a path between client and server and computes per-link delays based on a fixed packet size and link bandwidths; it forwards requests and responses with the computed delay (fractional seconds are preserved).

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
- The code schedules forwarding with `fwd_request.schedule(total_delay, c_request_in.value)` (and similarly for responses), which preserves fractional delays when the runtime supports floating-point schedule times.

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

### Run the example (Python target)

```bash
# Compile the LF source to Python
lfc NTP_System.lf

# Run the generated Python program
python src-gen/NTP_System.py
```

Note: The example uses prints for debugging and demonstration of timing and network emulation; the exact output and timing values will differ by platform and execution timing.

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

## Extensions

The example is intentionally simple and can be extended to explore:

- Multiple clients
- More realistic per-packet serialization and propagation models
- Introducing jitter, packet loss, or variable bandwidth
- Compiling to other targets supported by Lingua Franca

## Resources

- Lingua Franca official repo: https://github.com/lf-lang/lingua-franca
- Language docs and tutorials: https://lf-lang.org/docs

---
