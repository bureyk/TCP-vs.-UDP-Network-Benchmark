# TCP vs. UDP Network Benchmark

A Python benchmarking suite that empirically measures and compares TCP and UDP performance across a range of payload sizes and concurrency levels, rather than relying on theoretical assumptions about which protocol is "faster."

## What it measures

- **Per-request RTT** (round-trip time), measured client-side
- **Throughput** (bytes transferred per second)
- **TCP connection-setup overhead**, isolated separately from per-request latency
- Behavior across **5 payload sizes**: 64B, 512B, 1KB, 4KB, 8KB
- Behavior across **concurrency levels**: 1, 10, and 100 simultaneous clients

All traffic was run between two physically separate machines on a real network connection (not localhost), so the results reflect genuine network conditions rather than a loopback test.

## How it works

`client.py` and `server.py` implement matching TCP and UDP echo programs. For TCP, the client opens one persistent connection per client thread and sends `--requests` payloads over it sequentially, so the 3-way handshake cost is paid once and amortized across all requests on that connection. For UDP, there's no connection state, so the client fires datagrams sequentially with a timeout on each one. Both sides log structured JSONL events (connection start, each request received, errors, completion) with wall-clock timestamps.

`run_experiments.py` automates the full sweep: for every combination of protocol, payload size, and client count, it starts the server as a subprocess, runs the client against it, then parses the resulting JSONL logs to compute average, median, and p99 RTT plus throughput. It saves a summary CSV and regenerates 8 comparative matplotlib plots (latency and throughput vs. payload size, vs. client count, and time-to-finish variants) from the raw data.

## Project structure

```
tcp_udp_benchmark/
    client.py            # TCP/UDP echo client, measures RTT per request
    server.py             # TCP/UDP echo server
    run_experiments.py    # automated sweep across all configurations
    results/               # JSONL logs + summary.csv (generated on run)
    plots/                  # generated matplotlib plots
```

## Requirements

- Python 3.8+
- `matplotlib`, `numpy` (for `run_experiments.py`)

```bash
pip3 install matplotlib numpy --break-system-packages
```

## Usage

### Run the automated sweep

```bash
python3 run_experiments.py --server-host <server-ip> --port 5001
```

This starts the server for each configuration, runs the client, and writes results and plots automatically.

### Run a single configuration manually

Server:
```bash
python3 server.py --proto tcp --bind 0.0.0.0 --port 5001 --payload-bytes 1024 --requests 100 --clients 10 --log results/server.jsonl
```

Client:
```bash
python3 client.py --proto tcp --host <server-ip> --port 5001 --payload-bytes 1024 --requests 100 --clients 10 --log results/client.jsonl
```

Swap `--proto tcp` for `--proto udp` to run the UDP variant.

## Key findings

- **TCP starts out slower at small payloads** (e.g., 64B) due to the handshake cost, but the latency gap closes as payload size increases and transmission time begins to dominate.
- **UDP showed higher throughput at mid-range payloads** (peaking around 4KB) since it carries less per-message overhead, but throughput dropped off sharply at 8KB, likely due to fragmentation.
- **TCP latency increased with client count** more than UDP did, since the server handles each TCP client in its own thread, introducing contention; UDP has no per-client state and no thread overhead.
- A few results stood out as anomalies (e.g., an unexplained throughput spike for TCP at one specific client/payload combination) and are discussed candidly in the full report rather than smoothed over.

## Design notes

- RTT is measured entirely on the client side, avoiding the need for synchronized clocks between machines.
- UDP's total request count is set to `clients × requests` to match the TCP workload size fairly, since UDP has no persistent connections to distribute requests across.
