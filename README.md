<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logprism-wordmark-dark.svg">
    <img src="assets/logprism-wordmark-light.svg" alt="LogPrism" width="440">
  </picture>
</p>

<p align="center"><b>Any perimeter log in. One lossless, traceable record out. Nothing is ever dropped.</b></p>

---

## What it does

- Collects firewall, IDS, proxy and VPN logs in any format: syslog, JSON, CSV, CEF, LEEF, XML.
- Converts them all into one schema (**OCSF 1.3**).
- Keeps every original log, untouched and hashed (SHA-256).
- Quarantines unknown formats instead of dropping them, then learns new parsers from them.

## Why

- Every device speaks its own log format, so analysts juggle dozens of them.
- A new device means writing new parser code.
- Unknown or changed formats get silently dropped.
- The raw log is lost after parsing, so there's no evidence trail.
- Government SOCs are air-gapped and can't use cloud tools.

## How

```
Ingest → Archive → Buffer → Parse → Normalize → Validate → Deliver
                              │ fail
                              ▼
                 DLQ → Cluster → Propose parser → Test + Approve → Replay
```

| Step | What happens |
|---|---|
| Ingest | Syslog UDP/TCP `:5514`, HTTP `/ingest`, file tail |
| Archive | Raw bytes gzipped + SHA-256, append-only |
| Buffer | Redis Streams (optional) |
| Parse | One generic engine + one YAML file per vendor |
| Normalize | Mapped to OCSF 1.3; unmapped fields kept |
| Validate | JSON Schema check on every event |
| Deliver | JSONL, Elastic, Kafka, SIEM (CEF), MinIO, Parquet |
| Self-heal | Failed logs are clustered and a parser is drafted (rules or local LLM); it goes live only after tests pass and a human approves |

**Built-in parsers:** pfSense, Cisco ASA, FortiGate, Palo Alto, Suricata, Squid, generic CEF and LEEF.
**New device:** add one YAML file to `src/parser/configs/`, no code.

## Run it

**1. Install** (Python 3.10+)
```bash
python -m venv .venv
.venv\Scripts\activate          # Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
```

**2. Start**
```bash
python -m src.main
```
Open **http://localhost:8080**

**3. Send sample logs** (new terminal, venv activated)
```bash
python tools/generate_sample_logs.py --scenario demo
```

**4. Try self-heal:** open the **Self-heal** tab, review the proposed parser and click **Promote to live**.

### Optional

| Want | Do |
|---|---|
| Redis + MinIO | `docker compose up -d redis minio`, then `cp .env.example .env` (Windows: `copy .env.example .env`) |
| Everything in Docker | `docker compose up --build` |
| LLM-drafted parsers | Install [Ollama](https://ollama.com), then `ollama pull qwen2.5-coder` |
| Run tests | `python -m pytest tests` |
| Benchmark | `python tools/benchmark.py` |
| Parquet export | `python tools/export_parquet.py` |

Without Redis or MinIO it still runs: it falls back to an in-process buffer and local disk (`data/`).

---

<p align="center"><sub>Lowkey Coders · Smart India Hackathon 2026</sub></p>
