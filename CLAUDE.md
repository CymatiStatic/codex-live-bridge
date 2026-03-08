# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Before modifying code, consult these companion documents:**
> - **[SPEC.md](SPEC.md)** -- Full application specification (pipelines, data types, constraints, config)
> - **[REPO_INDEX.md](REPO_INDEX.md)** -- Complete file map, API reference, dependency graph
>
> **After modifying code, update SPEC.md and REPO_INDEX.md** if you changed APIs, added/removed files, or modified architecture.

## Project Overview

**codex-live-bridge** -- Open-source, local-first Codex-to-Ableton Live control bridge. Ships a Max for Live device, a JavaScript command router, and Python OSC client/CLI scripts that drive Ableton Live through LiveAPI (Live Object Model) over OSC/UDP. Includes a local compositional memory + eval governance loop. Started at the OpenAI 2026 Hackathon.

## Commands

```bash
# Verify bridge connectivity (requires Live + M4L device loaded)
python3 bridge/ableton_udp_bridge.py --ack --status --no-tempo --no-signature

# Full-surface smoke test (requires Live)
python3 bridge/full_surface_smoke_test.py

# Run unit tests
python3 -m unittest discover -s bridge -p "test_*.py"

# First-time memory template setup
mkdir -p memory && rsync -a music-preferences/ memory/

# Build retrieval index
python3 -m memory.retrieval index

# Query context brief
python3 -m memory.retrieval brief --meter 4/4 --bpm 120 --mood contemplative

# Summarize governance signals
python3 -m memory.eval_governance summarize --lookback 30

# Plan memory updates (dry-run)
python3 -m memory.eval_governance apply --date YYYY-MM-DD --dry-run

# Apply memory updates
python3 -m memory.eval_governance apply --date YYYY-MM-DD
```

## Architecture

### Two Layers

1. **Bridge Layer** (`bridge/`): OSC/UDP transport between Python CLI and Max for Live device
   - Python CLI encodes OSC packets (stdlib only, zero deps) -> UDP `:9000`
   - M4L device (`live_udp_bridge.js`) routes commands to LiveAPI -> ACKs on `:9001`
   - 24 commands: 18 legacy + 6 generic LiveAPI RPC (`/api/*`)

2. **Memory Layer** (`memory/`): Local compositional memory, retrieval index, eval governance
   - `compositional_memory.py`: TOML index loader + topic brief
   - `retrieval.py`: SQLite FTS5 index over markdown + eval artifacts
   - `eval_governance.py`: Signal extraction -> rule matching -> session/promotion/demotion

### Communication

```
Python CLI  --OSC/UDP :9000-->  M4L Device (live_udp_bridge.js)  --LiveAPI-->  Ableton Live
            <--OSC/UDP :9001--  ACK responses
```

## Key Conventions

- **All indices are 0-based** (LiveAPI convention; UI Track 1 = `track_index=0`)
- **Zero external Python dependencies** for bridge (stdlib-only OSC encoding)
- **Python 3.10+** required (`tomllib` stdlib; `tomli` fallback on 3.10)
- **Track 0 is protected** from `/delete_midi_tracks`
- **No trained models, no third-party music** -- memory templates ship blank
- **Pre-1.0.0** -- breaking changes possible between releases

## Configuration

- Bridge defaults: `127.0.0.1:9000` (commands), `:9001` (ACKs), 40ms inter-message delay
- Memory index: `music-preferences/index.toml` (11 fundamentals)
- Governance policy: `music-preferences/eval_governance_policy.toml` (5 rules)
- Eval artifacts: `music-preferences/evals/composition_index.json`

## Testing

- Framework: `unittest` (stdlib)
- Test file: `bridge/test_ableton_udp_bridge.py`
- Mock strategy: In-process mocks for socket/select (no live Ableton needed)
- Benchmark: `bridge/benchmark_midi_write.py` (deterministic, imports external workflow scripts)
- Smoke test: `bridge/full_surface_smoke_test.py` (requires live Ableton + device)
