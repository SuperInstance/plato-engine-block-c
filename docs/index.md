# Plato Engine Block — C Reference Implementation

> A tiny, embeddable sensor→history→alarm engine in C99. Zero dynamic allocation. Runs on bare metal.

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

## Overview

The Plato Engine Block is the reference implementation of the Plato monitoring philosophy: **read sensors, store history, fire alarms, stream it all**. Designed to run anywhere — from POSIX servers to bare-metal MCUs to game loops — with zero dynamic allocation after initialization.

## Quick Start

```bash
# Clone and build
git clone https://github.com/SuperInstance/plato-engine-block-c.git
cd plato-engine-block-c
make

# Run tests
make test

# Interactive daemon
./plato_engine

# Auto-tick daemon (ticks every 1000ms)
./plato_engine -a 1000

# TCP server (auto-ticks every 2000ms)
./plato_server          # default port 7070
./plato_server 9090     # custom port
```

## Architecture

```
Sensors → Tick Loop (plato_tick) → History Ring Buffer
                    ↓
                Alarms → Fire / Cooldown / Re-arm
                    
Actuators     Subscribers ← TCP Server (broadcast ticks)
                    
         Text Protocol: tick | history | actuator | alarm | subscribe
```

## Text Protocol

Commands via stdin or TCP:

```
tick sensor=0 value=23.5        # Feed sensor data
history sensor=0                 # Query history
actuator list                    # List actuators
actuator set=0 value=1           # Control actuator
alarm list                       # List alarms
subscribe                        # Subscribe to tick broadcasts
```

## Key Properties

- **Zero malloc** after init — safe for bare metal
- **Fixed-size ring buffers** — predictable memory
- **C99 standard** — portable to virtually any platform
- **TCP server** — broadcast ticks to subscribers
- **Sub-400 LOC** — auditable, teachable, hackable

## Resources

- [GitHub Repository](https://github.com/SuperInstance/plato-engine-block-c)
- [PLATO Server](https://github.com/SuperInstance/plato-server)
- [PLATO Wire Protocol](https://github.com/SuperInstance/AI-Writings/blob/main/PLATO_WIRE_PROTOCOL.md)
- [SuperInstance Ecosystem](https://github.com/SuperInstance/SuperInstance)

---

*Part of the [SuperInstance](https://github.com/SuperInstance) ecosystem.*
