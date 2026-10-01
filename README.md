# Truth Pipeline

A low-latency C++ pipeline that reads posts from a mocked Truth Social API, detects company names, scores the sentiment around each mention, and places **simulated** buy/sell orders, measuring the latency of every step along the way.

> **This is a learning project.** All trading is simulated through a paper broker. Nothing here connects to a real broker or trades real money, and nothing here is financial advice.

---

## Motivation

In July 2026, Trump Media & Technology Group announced **Truth API**, a paid business-to-business data feed (live since August 1, 2026) that delivers posts from Truth Social's most influential accounts, including the U.S. President's, to institutional customers. It is aimed at trading firms and reportedly delivers posts a few milliseconds before the public sees them, for up to $100,000 per month.

Posts on Truth Social regularly move markets: tariffs, trade deals and comments about individual companies. A feed that arrives milliseconds earlier is only valuable if the software consuming it can **read, understand and act on a post faster than everyone else**. That is a classic low-latency engineering problem.

This project rebuilds that problem end to end, as a way to learn low-latency programming in practice:

- **Networking:** receiving a push feed over TCP with minimal delay.
- **Text processing:** parsing, cleaning and searching text in microseconds.
- **ML inference under latency constraints:** deciding "good or bad for this company?" fast enough to matter.
- **Systems engineering:** memory, caches, threads and lock-free queues, and measuring what each of them actually costs.

The real Truth API's protocol and message format are not public, so the feed is mocked. The mock is designed to be realistic (push stream, sequence numbers, HTML content) so the pipeline faces the same problems it would face against the real thing.

---

## Architecture overview

The project consists of two independent programs that talk only over the network:

```
┌──────────────────────┐        TCP (localhost)        ┌──────────────────────────┐
│   mock_api (Python)  │  ───── JSON lines, push ────▶ │   pipeline (C++)         │
│                      │                               │                          │
│  replays posts from  │                               │  receive → parse → clean │
│  data files, stamps  │                               │  → match → context →     │
│  send time + seq     │                               │  score → decide → trade  │
└──────────────────────┘                               └──────────────────────────┘
```

- **`mock_api/`** plays the role of the Truth API. It is written in Python because it is outside the latency-critical path.
- **`pipeline/`** is the system under test. It is written in C++ because this is where every microsecond counts.

The C++ code never imports anything from the Python code. The only contract between them is the message format (below). This keeps the mock swappable: connecting to a real feed would only require a new adapter in the receiver.

---

## The mock API

### Data sources

The mock replays posts from data files rather than hardcoded values. Three kinds of data serve different purposes:

| Source | File | Purpose |
|---|---|---|
| **Historical posts** | `mock_api/data/archive.jsonl` | Realistic content and timing; lets signals be evaluated against real price moves. Built from public archives of Truth Social posts. |
| **Handwritten fixtures** | `mock_api/data/fixtures.jsonl` | Edge cases with known expected results (ambiguous names like "Apple pie", several companies in one post, sarcasm, HTML-heavy content). Used by automated tests. Uses a fictional account name. |
| **Synthetic load** | generated at runtime | Bursts and high message rates to stress the pipeline and expose latency under pressure. |

### Timing modes

| Mode | Behavior | Used for |
|---|---|---|
| `replay` | Real gaps between posts, with a speed-up factor | Realistic end-to-end runs and evaluation |
| `interval` | One post every N milliseconds | Debugging |
| `burst` | N posts as fast as possible | Stress and latency testing |

### Message format

One JSON object per line (JSON Lines), pushed over a persistent TCP connection with `TCP_NODELAY` enabled:

```json
{"seq": 42, "sent_ns": 183746529384, "post": {"id": "114132050804394743", "created_at": "2025-03-09T10:41:28.605Z", "account": "realDonaldTrump", "content": "<p>...</p>"}}
```

| Field | Meaning |
|---|---|
| `seq` | Increases by 1 per message. A gap means a message was missed. |
| `sent_ns` | The mock's monotonic clock (`time.monotonic_ns()`) at send time. Both processes run on the same machine and share this clock, so the pipeline can compute exact wire-to-decision latency. |
| `post` | The post as stored in the archive. `content` is **HTML** (Truth Social is Mastodon-based), which the pipeline must strip. |

The full specification lives in `docs/message_format.md`.

---

## The C++ pipeline

### Stages

Each post flows through a chain of stages. Each stage is either a **function** (no memory between posts; the same input always gives the same output) or an **object** (holds state: built once at startup, then called per post).

| # | Stage | Kind | Input → Output | State it holds |
|---|---|---|---|---|
| 1 | `Receiver` | object | TCP bytes → JSON line | Open socket, partial-message buffer |
| 2 | `parse_message` | function | JSON line → `Post` | — |
| 3 | `strip_html` | function | HTML → plain text | — |
| 4 | `Matcher` | object | text → `Mention`s (ticker + position) | Company name/alias table |
| 5 | `extract_context` | function | text + mentions → `Snippet`s | — |
| 6 | `Scorer` | object | `Snippet` → `Signal` (sentiment, confidence) | Lexicon, later an ONNX model |
| 7 | `Strategy` | object | `Signal`s → `Order`s | Positions, risk limits |
| 8 | `PaperBroker` | object | `Order` → simulated fill, P&L | Trade log, running P&L |

Objects follow a **cold path / hot path** split: expensive setup (building lookup tables, loading models) happens in the constructor at startup; the per-post method does only the minimum.

### Data types

The structs passed between stages are the internal contract of the pipeline (`include/types.hpp`):

- **`Post`**: `seq`, `id`, `text`, `Timestamps`
- **`Mention`**: `ticker`, `position`
- **`Snippet`**: `ticker`, `context`
- **`Signal`**: `ticker`, `sentiment` (−1…+1), `confidence` (0…1)
- **`Order`**: `ticker`, `side` (buy/sell), `quantity`
- **`Timestamps`**: `sent_ns`, `received_ns`, `parsed_ns`, `matched_ns`, `scored_ns`, `decided_ns`

### Separation of logic and wiring

Stages know nothing about threads or queues; they only transform data. How they are connected (a single loop, or several threads with queues in between) lives only in `main.cpp`. This means:

- stage tests stay valid in every phase,
- single-threaded and multi-threaded versions can be compared with identical stage logic,
- components (for example the Scorer) can be upgraded without touching the rest.

### Latency measurement

Every stage records a timestamp on the `Post`. A `LatencyRecorder` collects these per stage and reports **p50, p99 and p99.9**, not averages, because in trading the slow tail decides whether you win or lose a race. Recording happens off the hot path so that measuring does not distort the measurement.

---

## Development phases

The pipeline is built in phases: correctness first, then measurement, then optimization guided by the measurements.

| Phase | Goal | Key techniques |
|---|---|---|
| **1. Simple and correct** | Single thread, one loop from receive to trade; lexicon-based scoring | Structs, `std::string`, `std::vector`, CMake, unit tests |
| **2. Measure** | Per-stage timestamps and latency histograms | Monotonic clocks, percentiles, benchmarks |
| **3. Multi-threaded** | Stages on separate threads connected by queues | `std::thread`, mutex-based queues, move semantics |
| **4. Optimize** | Remove sources of latency the measurements reveal | SPSC lock-free ring buffers, atomics and memory ordering, no allocations on the hot path, `string_view`, cache-line alignment, busy-polling sockets, thread pinning |
| **5. Real model** | Replace the lexicon with a small transformer | ONNX Runtime, int8 quantization, two-stage scoring (fast lexicon first, transformer for ambiguous cases) |

Planned experiments along the way:

- **Push vs. polling:** the mock supports both, to measure the cost of polling.
- **TCP vs. UDP under packet loss:** simulated with `tc netem`, to see TCP turn loss into delay and UDP turn loss into missing messages.
- **Internet-scale delay:** `netem` delay and jitter, to put microsecond optimizations in perspective.

---

## Repository layout

```
truth-pipeline/
├── README.md
├── CLAUDE.md                  # working instructions for Claude Code
├── docs/
│   └── message_format.md      # contract between mock and pipeline
├── mock_api/                  # Python mock of the Truth API
│   ├── server.py
│   └── data/
│       ├── archive.jsonl      # historical posts
│       └── fixtures.jsonl     # handwritten edge cases
└── pipeline/                  # C++ low-latency pipeline
    ├── CMakeLists.txt
    ├── include/               # headers: what exists
    │   ├── types.hpp
    │   ├── stages.hpp
    │   ├── receiver.hpp
    │   ├── matcher.hpp
    │   └── scorer.hpp
    ├── src/                   # implementations: how it works
    │   ├── main.cpp           # wiring of the stages
    │   ├── parse.cpp
    │   ├── receiver.cpp
    │   ├── matcher.cpp
    │   └── scorer.cpp
    ├── tests/
    │   ├── test_matcher.cpp
    │   └── test_strip_html.cpp
    └── data/
        ├── companies.csv      # name, ticker, aliases
        └── lexicon.csv        # word, weight
```

---

## Getting started

**Environment:** Linux or WSL2 (Ubuntu). The optimization phases use Linux-specific APIs (thread affinity, socket options, `tc netem`, `perf`). WSL2 is fine for development; final latency measurements should run on native Linux.

**Requirements:** Python 3, `g++`, CMake, Git.

```bash
sudo apt update && sudo apt install -y python3 python3-venv g++ cmake git
```

**Run the mock API** (terminal 1):

```bash
python3 mock_api/server.py --mode replay --speed 100
```

**Build and run the pipeline** (terminal 2):

```bash
cmake -S pipeline -B pipeline/build -DCMAKE_BUILD_TYPE=Release
cmake --build pipeline/build
./pipeline/build/pipeline
```

For development, build with AddressSanitizer to catch memory errors:

```bash
cmake -S pipeline -B pipeline/build-debug -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS="-fsanitize=address -g"
```

> Commands reflect the planned interface and will be updated as the components are implemented.

---

## Non-goals

- **Real trading.** Orders only ever reach the simulated paper broker.
- **Kernel bypass and special hardware** (DPDK, FPGAs, colocation). These are out of reach on a laptop; the project aims to make their motivation measurable, not to use them.
- **A profitable strategy.** Signal quality is evaluated, but the focus is the engineering of speed.

---

## References

- Alexander Obregon, [Writing Low-Latency C++ Applications](https://medium.com/@AlexanderObregon/writing-low-latency-c-applications-f759c94f52f8) (Medium), the starting point for this project
- [Trump Media announcement of Truth API](https://finance.yahoo.com/technology/articles/trump-media-technology-group-launches-130000261.html)
- [stiles/trump-truth-social-archive](https://github.com/stiles/trump-truth-social-archive), public archive of Truth Social posts

Made by Django and Rapide and God