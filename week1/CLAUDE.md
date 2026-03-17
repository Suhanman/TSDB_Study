# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Week 1 study project simulating a time-series data pipeline with binary file processing and Prometheus metrics export. The three scripts are designed to run concurrently as separate processes.

## Running the scripts

A `.venv` is already present. Each script runs as a standalone process:

```bash
.venv/bin/python generator.py   # Generates binary .bin files into /data/incoming
.venv/bin/python loader.py      # Validates and routes files from /data/incoming to /done or /error
.venv/bin/python exporter.py    # Exposes Prometheus metrics at http://localhost:8000/metrics
```

To install the single external dependency into the venv:

```bash
.venv/bin/pip install prometheus_client
```

## Architecture

The pipeline has three stages connected by a shared `/data/` directory on the filesystem:

1. **generator.py** — Produces binary files (`data_NNNNN.bin`) in `/data/incoming/` at random 3–20s intervals. Deliberately injects errors: ~10% bad magic, ~5% wrong version, ~5% wrong checksum, ~3% size mismatch, ~2% partial body (~25% total error rate).

2. **loader.py** — Polls `/data/incoming/` every 2 seconds. Moves each file to `/data/processing/` while validating (header → layout → checksum), then routes to `/data/done/` (success) or `/data/error/` (failure). On success, **also copies the file to the date-partitioned data lake** at `/data/lake/year=YYYY/month=MM/day=DD/`. Persists cumulative stats to `/data/logs/loader_stats.json`.

3. **exporter.py** — Polls every 5 seconds. Tails `app.log` for keyword-based log-level counts (`[SUCCESS]`, `[ERROR]`, `[WARNING]`, `[ALL]`) and reads `loader_stats.json` for validation counters. Also scans pipeline dirs and last-7-days lake partitions. Exposes everything as Prometheus gauges/counters at `:8000/metrics`.

### Inter-process shared state

| File | Written by | Read by |
|---|---|---|
| `/data/logs/app.log` | generator, loader | exporter (`collect_log`) |
| `/data/logs/loader_stats.json` | loader | exporter (`collect_loader_stats`) |
| `/data/incoming/`, `/data/done/`, `/data/error/` | generator, loader | loader, exporter |
| `/data/lake/year=.../...` | loader (on success) | exporter (`collect_partition`) |

## Binary file format

All `.bin` files use big-endian encoding:

```
Header (14 bytes): magic[4s] + version[H=uint16] + record_count[I=uint32] + checksum[i=int32]
Body:              record_count × 8 bytes (each record is one int64)
```

- `magic` must be `b'DAT1'`, `version` must be `2`
- `checksum` = `sum(body_bytes) % 2^32`
- Body size validation: `(file_size - 14) // 8` must equal `record_count`
