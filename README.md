# Lingua-Franca: Reactor-Oriented Temporal Semantics & NTP

A research paradigm for evaluating how **reactor-oriented programming models** handle **temporal semantics** and **concurrency** by implementing distributed clock synchronization using the **Network Time Protocol (NTP)**.

## Overview

This project demonstrates the capabilities and constraints of Lingua Franca's reactor model when dealing with distributed systems that require precise timing and synchronization. By implementing an NTP-inspired client-server system, it showcases how reactor-based concurrency handles:

- **Temporal reasoning**: Tracking and comparing local vs. network time
- **Asynchronous communication**: Event-driven reactor interactions
- **Distributed coordination**: Multi-reactor systems communicating over logical connections
- **Clock synchronization**: Real-time protocol implementation in a declarative model

## What is Lingua Franca?

Lingua Franca is a polyglot reactive framework designed for building distributed, time-sensitive systems. It provides:

- **Reactor model**: Units of concurrent computation with typed ports (inputs/outputs)
- **Deterministic concurrency**: Logical time semantics for reproducible execution
- **Multi-target compilation**: Write once, compile to Python, C, C++, TypeScript, Rust, and more
- **Time-aware scheduling**: Built-in support for temporal actions and reactions

## Repository Structure

```
Lingua-Franca/
├── NTP_System.lf          Example NTP implementation in Lingua Franca
└── README.md              This file
```

## The NTP System Example

### Architecture

The implementation consists of three main reactors:

#### **Server Reactor**
- Listens for incoming NTP requests
- Responds with the current system time
- Acts as the source of truth for time synchronization

```lf
reactor Server {
    input ntp_request      # Receives sync requests
    output ntp_response    # Sends back current time
    
    reaction(ntp_request) -> ntp_response {=
        import time
        ntp_response.set(time.time())
    =}
}
```

#### **Client Reactor**
- Initiates NTP requests on startup
- Receives time from the server
- Calculates local clock skew (difference between local and server time)
- Logs the synchronization result

```lf
reactor Client {
    output ntp_request     # Sends sync request to server
    input ntp_response     # Receives server time
    
    logical action send_request   # Triggers initial request
    
    reaction(startup) -> send_request {=
        send_request.schedule(0)
    =}
    
    reaction(send_request) -> ntp_request {=
        ntp_request.set(True)
    =}
    
    reaction(ntp_response) {=
        import time
        diff = time.time() - ntp_response.value
        print(f"Local clock is ahead by {diff} seconds.")
    =}
}
```

#### **NTP_System Main Reactor**
Wires the client and server together, establishing the communication topology:

```lf
main reactor NTP_System {
    s = new Server()
    c = new Client()
    
    c.ntp_request -> s.ntp_request    # Client request flows to server
    s.ntp_response -> c.ntp_response  # Server response flows back to client
}
```

### How It Works

1. **Startup**: The system initializes both Server and Client reactors
2. **Request**: On startup, the Client schedules a logical action that triggers an NTP request
3. **Response**: The Server receives the request and responds with the current time
4. **Synchronization**: The Client receives the response and calculates the clock offset
5. **Output**: The calculated time difference is printed to console

## Getting Started

### Prerequisites

- **Lingua Franca compiler** (visit [lf-lang.org](https://lf-lang.org))
- **Python 3.7+** (or C/C++/TypeScript/Rust, depending on target)

### Running the Example

```bash
# Compile and run the NTP system (Python target)
lfc NTP_System.lf

# Execute the generated code
python src-gen/NTP_System.py
```

### Expected Output

```
Local clock is ahead by 0.0001234 seconds.
```

The exact value depends on system timing and precision. In most cases, the difference will be very small since both client and server run on the same machine.

## Key Concepts Demonstrated

| Concept | Implementation |
|---------|-----------------|
| **Asynchronous reactions** | Client/Server respond to input events without blocking |
| **Logical actions** | `send_request` action triggers deterministically after startup |
| **Port connections** | Typed data flows through reactor ports |
| **Temporal semantics** | Time measurements and clock synchronization |
| **Python integration** | Embedded Python code for time operations |
| **Determinism** | Reactions execute in a well-defined order within a time step |

## Potential Extensions

This example can be extended to explore:

- **Multi-client synchronization**: Multiple clients coordinating with a single server
- **Distributed consensus**: Implementing more complex protocols like Paxos or Raft
- **Fault tolerance**: Handling delayed/lost messages and timeouts
- **Precision timing**: Using microsecond-level timing with real hardware clocks
- **Multiple targets**: Comparing how the system behaves when compiled to different languages

## Lingua Franca Resources

- 📖 [Official Documentation](https://github.com/lf-lang/lingua-franca)
- 🎓 [Tutorials & Examples](https://lf-lang.org/docs)
- 💬 [Community & Support](https://github.com/lf-lang/lingua-franca/discussions)

## About This Project

This repository is part of a research initiative to evaluate the **expressive power and correctness properties** of reactor-oriented models when applied to distributed systems with strict timing requirements. By studying how Lingua Franca handles NTP-like protocols, we can better understand:

- The adequacy of logical time semantics for distributed synchronization
- How reactor models reason about causality in temporal systems
- Trade-offs between determinism and expressiveness

## License

This project is provided as-is for research and educational purposes.

---

**Questions?** Feel free to open an issue or reach out to the repository maintainer.
