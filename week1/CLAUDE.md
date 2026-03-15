# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Week 1 study project simulating a time-series data pipeline with binary file processing and Prometheus metrics export. The three scripts are designed to run concurrently as separate processes.

## Running the scripts

Each script runs as a standalone process:

```bash
python generator.py   # Generates binary .bin files into /data/incoming every 10s
python loader.py      # Validates and routes files from /data/incoming to /done or /error
python exporter.py    # Exposes Prometheus metrics at http://localhost:8000/metrics
```

Install the single external dependency:

```bash
pip install prometheus_client
```

## Architecture

The pipeline has three stages connected by a shared `/data/` directory on the filesystem:

1. **generator.py** — Produces binary files (`data_NNNNN.bin`) in `/data/incoming/`. ~10% are corrupted (bad magic bytes), ~5% have wrong version numbers. Also copies every file into a date-partitioned data lake at `/data/lake/year=YYYY/month=MM/day=DD/`.

2. **loader.py** — Polls `/data/incoming/` every 2 seconds. Moves each file to `/data/processing/` while validating, then routes to `/data/done/` (success) or `/data/error/` (failure). Logs all results to `/data/logs/app.log`.

3. **exporter.py** — Polls every 5 seconds. Tails `app.log` for log-level counts and inspects binary files in `/data/incoming/` for header/layout metrics. Exposes everything as Prometheus gauges and counters.

## Binary file format

All `.bin` files use big-endian encoding:

```
Header (14 bytes): magic[4s] + version[H=uint16] + record_count[I=uint32] + checksum[i=int32]
Body:              record_count × 8 bytes (each record is one int64)
```

- `magic` must be `b'DAT1'`, `version` must be `2`
- `checksum` = `sum(body_bytes) % 2^32`
- Body size validation: `(file_size - 14) // 8` must equal `record_count`
