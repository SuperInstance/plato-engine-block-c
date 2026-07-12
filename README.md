# Plato Engine Block — C Reference Implementation

[![CI](https://github.com/SuperInstance/plato-engine-block-c/actions/workflows/ci.yml/badge.svg)](https://github.com/SuperInstance/plato-engine-block-c/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Language](https://img.shields.io/badge/language-C99-blue.svg)](https://en.wikipedia.org/wiki/C99)
[![Binary Size](https://img.shields.io/badge/binary-~15KB-success.svg)](#performance-characteristics)

> **A tiny, embeddable sensor→history→alarm engine in C99. Zero dynamic allocation. Runs on bare metal.**

---

## Quick Start

```bash
git clone https://github.com/SuperInstance/plato-engine-block-c.git
cd plato-engine-block-c
make && make test
```

```bash
./plato_engine
> tick
tick 1: cpu_temp=62.34 random=47.81 constant=42.00
> history 5
history (5):
  cpu_temp: 62.34
  random: 47.81
  constant: 42.00
> alarm list
alarms (3):
  [0] overheat  sensor=cpu_temp  > 80.00  sev=CRIT  armed=yes
```

---

## What It Does

The Plato Engine Block is the reference implementation of the Plato monitoring philosophy: **read sensors, store history, fire alarms, stream it all**. It's designed to run anywhere — from POSIX servers to bare-metal MCUs to game loops — with zero dynamic allocation after initialization.

The engine implements a tick-driven loop: each tick reads all registered sensors, stores values in a ring-buffer history, evaluates alarm thresholds, and optionally broadcasts results to TCP subscribers. The entire implementation fits in a single C99 header file (~400 lines) with a companion daemon and TCP server. No heap allocation. No threads. No external dependencies. Override the configuration constants before `#include` to tune for your target — from 800 bytes of RAM on an ESP8266 to the full 8 KB default configuration on a server.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Plato Engine Block                        │
│                                                              │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │ Sensors  │───▶│  Tick Loop   │───▶│  History     │        │
│  │ (N max)  │    │  (plato_tick)│    │  Ring Buffer │        │
│  └──────────┘    └──────┬───────┘    └──────────────┘        │
│                         │                                     │
│                    ┌────▼────┐                                │
│                    │ Alarms  │──▶ Fire / Cooldown / Re-arm    │
│                    │ (N max) │                                │
│                    └─────────┘                                │
│                                                              │
│  ┌──────────────┐  ┌───────────────┐                         │
│  │  Actuators   │  │  Subscribers  │◀── TCP Server            │
│  │  (N max)     │  │  (N max)      │    (broadcast ticks)     │
│  └──────────────┘  └───────────────┘                         │
└──────────────────────────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
   ┌──────────┐                        ┌────────────┐
   │  stdin   │                        │ TCP :7070  │
   │  daemon  │                        │  server    │
   └──────────┘                        └────────────┘
```

This is the **C flagship implementation** in the SuperInstance PLATO ecosystem. Each engine block is one cell in the tensor grid managed by [plato-runtime-kernel](https://github.com/SuperInstance/plato-runtime-kernel). The engine handles the physical layer (sensors, actuators, ticks); the runtime kernel handles the spatial layer (topology, traversals, contracts).

### Performance Characteristics

| Metric | Value |
|---|---|
| Single tick (16 sensors) | < 1 μs |
| History lookup | O(1) |
| Alarm evaluation (16 alarms) | < 1 μs |
| Memory (default config) | ~8 KB |
| Memory (embedded config) | ~1.5 KB |
| Binary size (stripped) | ~15 KB |
| Dynamic allocations | 0 (after init) |
| Thread safety | None (single-threaded by design) |

---

## API / Usage

### Header Library (`plato_engine.h`)

Include in one `.c` file with `#define PLATO_ENGINE_IMPL` for the implementation.

#### Core Functions

```c
void plato_init(plato_engine_t *eng);

int plato_add_sensor(plato_engine_t *eng, const char *name,
                     plato_sensor_fn fn, void *user_data);

int plato_add_actuator(plato_engine_t *eng, const char *name,
                       plato_actuator_fn fn, void *user_data);

int plato_add_alarm(plato_engine_t *eng, const char *name,
                    int sensor_idx, plato_cmp_t cmp,
                    double threshold, plato_severity_t severity);

void plato_tick(plato_engine_t *eng);

int plato_handle_command(plato_engine_t *eng, const char *cmd,
                         char *resp, size_t resp_len);
```

#### Configuration Constants

| Constant | Default | Description |
|---|---|---|
| `PLATO_MAX_SENSORS` | 16 | Maximum sensor count |
| `PLATO_MAX_ACTUATORS` | 8 | Maximum actuator count |
| `PLATO_MAX_ALARMS` | 16 | Maximum alarm count |
| `PLATO_MAX_HISTORY` | 256 | History buffer depth per sensor |
| `PLATO_MAX_SUBSCRIBERS` | 32 | TCP subscriber limit |

### Engine Room Example

```c
plato_engine_t eng;
plato_init(&eng);

int temp = plato_add_sensor(&eng, "exhaust_temp", read_thermocouple, &tc1);
plato_add_alarm(&eng, "overtemp", temp, PLATO_GT, 220.0, PLATO_CRIT);
plato_add_actuator(&eng, "shutdown_valve", write_solenoid, &sv1);

// Tick loop — call at your desired rate
while (running) {
    plato_tick(&eng);
    usleep(1000000); // 1 second
}
```

### ESP32 / Bare Metal

```c
#define PLATO_MAX_SENSORS     8
#define PLATO_MAX_HISTORY     64
#define PLATO_MAX_SUBSCRIBERS 4
#define PLATO_ENGINE_IMPL
#include "plato_engine.h"
// ~1.5 KB RAM, ~2 KB flash
```

### Protocol Commands

| Command | Response | Description |
|---|---|---|
| `tick` | `tick N: name=val ...` | Read sensors, advance tick |
| `history [N]` | Formatted history | Show last N readings per sensor |
| `<actuator> <value>` | `ok name=val` / `err` | Set actuator value |
| `alarm list` | Alarm table | Show all alarms and state |
| `subscribe` | `ok subscribed` | Enable tick broadcasts |
| `help` | Command list | Show available commands |
| `quit` | `bye` | Disconnect |

---

## Testing

```bash
make test    # Builds and runs all tests

# Run daemon interactively
./plato_engine

# TCP server (auto-ticks every 2000ms)
./plato_server

# Auto-tick daemon at custom rate
./plato_engine -a 100   # 10 Hz
```

---

## Contributing

Contributions are welcome! See the [SuperInstance Contributing Guide](https://github.com/SuperInstance/SuperInstance/blob/main/CONTRIBUTING.md).

1. Fork the repo
2. Create a feature branch
3. Add tests for new functionality
4. Ensure `make test` passes
5. Keep the header under 400 lines — the constraint is the point
6. Submit a PR

---

## PLATO Engine Block Family

| Implementation | Language | Repo | Focus |
|---|---|---|---|
| **C Reference** ← you are here | C99 | [plato-engine-block-c](https://github.com/SuperInstance/plato-engine-block-c) | Embedded, bare-metal, zero heap alloc |
| **Rust (Original)** | Rust | [plato-engine-block](https://github.com/SuperInstance/plato-engine-block) | `no_std` + alloc, builder pattern, tokio server |
| **Elixir/OTP** | Elixir | [plato-engine-block-elixir](https://github.com/SuperInstance/plato-engine-block-elixir) | BEAM supervision trees, fault tolerance, hot reload |
| **Runtime Kernel** | Rust | [plato-runtime-kernel](https://github.com/SuperInstance/plato-runtime-kernel) | Spatial model: tensor grid, batons, assertion traps |
| **Server** | Python | [plato-server](https://github.com/SuperInstance/plato-server) | Knowledge tiles, fleet sync via Matrix, HTTP API |

---

## Ecosystem

This repo is part of the **SuperInstance** flagship ecosystem — agent-first computation, constraint theory, and self-improving runtimes.

### FLUX Runtime Family

| Repo | Language | Description |
|------|----------|-------------|
| [flux-runtime](https://github.com/SuperInstance/flux-runtime) | Python | Full FLUX runtime: markdown→bytecode, 2037 tests, zero deps |
| [flux-core](https://github.com/SuperInstance/flux-core) | Rust | Register-based bytecode VM, deterministic agent computation |
| [flux-js](https://github.com/SuperInstance/flux-js) | JavaScript | FLUX VM for Node.js and browsers, ~400ns/iter |
| [flux-compiler](https://github.com/SuperInstance/flux-compiler) | Rust/Python | Formal-methods compiler for safety-critical codegen |
| [flux-vm](https://github.com/SuperInstance/flux-vm) | Rust | Stack-based constraint-checking VM, 50 opcodes, Turing-incomplete |

### PLATO Engine Family

| Repo | Language | Description |
|------|----------|-------------|
| [plato-server](https://github.com/SuperInstance/plato-server) | Python | Knowledge tiles, fleet sync via Matrix, HTTP API |
| [plato-engine-block](https://github.com/SuperInstance/plato-engine-block) | Rust | Original room runtime: no_std + alloc, builder pattern |
| [plato-engine-block-c](https://github.com/SuperInstance/plato-engine-block-c) | C99 | Embedded reference: zero heap alloc, bare-metal portable |
| [plato-engine-block-elixir](https://github.com/SuperInstance/plato-engine-block-elixir) | Elixir | BEAM supervision trees, fault tolerance, hot reload |
| [plato-runtime-kernel](https://github.com/SuperInstance/plato-runtime-kernel) | Rust | Spatial model: tensor grid, batons, assertion traps |

### Constraint / Theory Family

| Repo | Language | Description |
|------|----------|-------------|
| [categorical-agents](https://github.com/SuperInstance/categorical-agents) | Rust | Category theory for agent composition (functors, naturality) |
| [cuda-constraint-engine](https://github.com/SuperInstance/cuda-constraint-engine) | CUDA/C | GPU constraint checking at 1B+ constraints/sec |
| [grand-pattern-rs](https://github.com/SuperInstance/grand-pattern-rs) | Rust | Fibonacci dual-direction cellular graph architecture |
| [lau-hodge-theory](https://github.com/SuperInstance/lau-hodge-theory) | Rust | Hodge decomposition, Betti numbers, spectral sequences |
| [ternary-science](https://github.com/SuperInstance/ternary-science) | Rust | Experimental evidence for ternary intelligence, 5 conservation laws |

### Agent / Infrastructure Family

| Repo | Language | Description |
|------|----------|-------------|
| [construct-core](https://github.com/SuperInstance/construct-core) | Rust | Layered trait system: bare-metal → alloc → async agent runtime |
| [crab](https://github.com/SuperInstance/crab) | Bash | Agent shell for repo entry/leave (MUD-room metaphor) |
| [exocortex](https://github.com/SuperInstance/exocortex) | Rust | Persistent cognitive substrate, S3-compatible memory |
| [git-agent](https://github.com/SuperInstance/git-agent) | Python | The repo IS the agent — autonomous lifecycle via Git |
| [capitaine-1](https://github.com/SuperInstance/capitaine-1) | TypeScript | Git-native repo-agent, Cloudflare Workers heartbeat |
| [codespace-edge-rd](https://github.com/SuperInstance/codespace-edge-rd) | Research | Codespace→Edge agent lifecycle and yoke transfer protocols |
| [git-agent-codespace](https://github.com/SuperInstance/git-agent-codespace) | DevContainer | One-click Codespace template for Git-Agent runtimes |

### Registries

| Registry | Package | Install |
|----------|---------|---------|
| **PyPI** | `flux-vm` | `pip install flux-vm` |
| **crates.io** | `fluxvm` | `cargo add fluxvm` |
| **npm** | `flux-js` | `npm install flux-js` |

### Philosophy & Architecture

- 📖 [AI-Writings](https://github.com/SuperInstance/AI-Writings) — Philosophy, essays, and design rationale
- 📦 [PACKAGES.md](https://github.com/SuperInstance/SuperInstance/blob/main/PACKAGES.md) — Full package index

---

## License

MIT — use it for anything. See [LICENSE](LICENSE).
