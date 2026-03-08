# SPEC.md -- codex-live-bridge Application Specification

## Product Identity

| Field        | Value |
|-------------|-------|
| **Name**    | codex-live-bridge |
| **Version** | pre-1.0.0 (Unreleased) |
| **Platform** | macOS (primary), Windows/Linux (untested) |
| **Purpose** | Open-source, local-first Codex-to-Ableton Live control bridge via OSC/UDP with compositional memory and eval governance |
| **License** | MIT |
| **Origin**  | OpenAI 2026 Hackathon, San Francisco |

## Architecture Overview

Three layers: **Bridge** (OSC transport + LiveAPI routing), **Memory** (local compositional memory index + retrieval), and **Eval Governance** (eval-driven bounded memory updates).

```
+------------------------------------------+
|          User / Codex Surface            |
|  (Codex App, CLI, IDE Extension)         |
+------------------------------------------+
         |                    |
         v                    v
+------------------+  +-------------------+
| Bridge Layer     |  | Memory Layer      |
| (OSC/UDP :9000)  |  | (SQLite + TOML)   |
|                  |  |                   |
| Python CLI       |  | compositional_    |
| -> OSC encode    |  |   memory.py       |
| -> UDP send      |  | retrieval.py      |
+--------+---------+  | eval_governance.py|
         |            +-------------------+
         v
+------------------+
| M4L Device       |
| (JS router)      |
| udpreceive 9000  |
| udpsend 9001     |
+--------+---------+
         |
         v
+------------------+
| Ableton LiveAPI  |
| (Live Object     |
|  Model)          |
+------------------+
```

## Communication Protocol

### Transport
- **Protocol**: OSC over UDP (standard library encoding, no dependencies)
- **Command channel**: `127.0.0.1:9000` (Python -> M4L device)
- **ACK channel**: `127.0.0.1:9001` (M4L device -> Python)
- **Encoding**: OSC type tags: `i` (int32), `f` (float32), `s` (string)

### Request/Response Pattern
- Commands sent as OSC messages to `:9000`
- ACKs returned as `/ack <event_name> <args...>` to `:9001`
- Optional `request_id` trailing argument echoed in `/api/*` ACKs
- No connection state; each UDP packet is independent

## Command Surface (24 commands)

### Legacy Commands (18)

| # | Command | Args | Description |
|---|---------|------|-------------|
| 1 | `/ping` | none | Connectivity check |
| 2 | `/tempo` | `<bpm:float>` | Set session tempo |
| 3 | `/sig_num` | `<num:int>` | Set time signature numerator |
| 4 | `/sig_den` | `<den:int>` | Set time signature denominator |
| 5 | `/create_midi_track` | none | Create one MIDI track at end |
| 6 | `/add_midi_tracks` | `<count:int> [name:str]` | Create + rename N MIDI tracks |
| 7 | `/create_audio_track` | none | Create one audio track at end |
| 8 | `/add_audio_tracks` | `<count:int> [prefix:str]` | Create + rename N audio tracks |
| 9 | `/delete_audio_tracks` | `<count:int>` | Delete N audio tracks from end |
| 10 | `/delete_midi_tracks` | `<count:int>` | Delete N MIDI tracks from end (track 0 protected) |
| 11 | `/rename_track` | `<index:int> <name:str>` | Rename track at index |
| 12 | `/set_session_clip_notes` | `<track:int> <slot:int> <len:float> <notes_json:str> [name:str]` | Destructive clip write |
| 13 | `/append_session_clip_notes` | `<track:int> <slot:int> <notes_json:str>` | Append to existing clip |
| 14 | `/inspect_session_clip_notes` | `<track:int> <slot:int>` | Report note count, pitch range, clip length |
| 15 | `/ensure_midi_tracks` | `<target:int>` | Ensure at least N MIDI tracks exist |
| 16 | `/midi_cc` | `<ctrl:int> <val:int> [ch:int]` | Send MIDI CC message |
| 17 | `/cc64` | `<val:int> [ch:int]` | Sustain pedal shortcut (CC64) |
| 18 | `/status` | none | Report track counts and live_set info |

### LiveAPI RPC Commands (6)

| # | Command | Args | Description |
|---|---------|------|-------------|
| 19 | `/api/ping` | `[request_id]` | RPC ping with optional correlation |
| 20 | `/api/get` | `<path> <property> [request_id]` | Read any LOM property |
| 21 | `/api/set` | `<path> <property> <value_json> [request_id]` | Write any LOM property |
| 22 | `/api/call` | `<path> <method> <args_json> [request_id]` | Call any LOM method |
| 23 | `/api/children` | `<path> <child_name> [request_id]` | Enumerate child objects |
| 24 | `/api/describe` | `<path> [request_id]` | Describe object (id, name, type, capabilities) |

## Data Types

### notes_json Format
```json
{"notes": [
  {"pitch": 60, "start_time": 0.0, "duration": 0.5, "velocity": 100, "mute": 0}
]}
```
- `pitch`: 0-127 (required)
- `start_time`: >= 0 beats (required)
- `duration`: > 0 beats (required)
- `velocity`: 1-127 (default 100)
- `mute`: 0 or 1 (default 0)
- Accepts bare array `[...]` or wrapper `{"notes": [...]}`

### api_children Response
```json
[{"index": 0, "id": 1234, "path": "live_set tracks 0", "name": "Track 1"}]
```

### Eval Artifact Schema
```json
{
  "run_id": "string",
  "timestamp_utc": "ISO-8601",
  "status": "success|save_failed",
  "composition": {"mood": "", "key_name": "", "tempo_bpm": 0, "signature": ""},
  "fingerprints": {"meter_bpm": "4/4@120"},
  "reflection": {
    "novelty_score": 0.0,
    "similarity_to_reference": 0.0,
    "repetition_flags": [],
    "merit_rubric": {"repetition_risk": 0.0},
    "instrument_identity": {"status": "", "flags": []},
    "prompts": []
  }
}
```

## Indexing Conventions

- **All bridge indices are 0-based** (LiveAPI convention)
- Ableton UI labels are 1-based: UI "Track 1" = `track_index=0`
- First clip slot = `slot_index=0`

## Configuration

### Bridge Defaults

| Parameter | Default | Description |
|-----------|---------|-------------|
| `host` | `127.0.0.1` | UDP target |
| `port` | `9000` | Command port |
| `ack_port` | `9001` | ACK listen port |
| `ack_timeout` | `0.6s` | Per-command ACK wait |
| `ack_mode` | `per_command` | `per_command`, `flush_end`, `flush_interval` |
| `ack_flush_interval` | `10` | Commands between flushes |
| `delay_ms` | `40` | Inter-message delay |
| `tempo` | `120.0` | Default BPM (skip with `--no-tempo`) |
| `sig_num` / `sig_den` | `4 / 4` | Default time sig (skip with `--no-signature`) |

### Memory Index (index.toml)

- 11 fundamentals: rhythm, harmony, timbre, velocity, key, meter, tempo, mood, arrangement, evaluation, silence
- Each has: summary, principles, heuristics, checklist, references
- `current_focus` tracks active compositional focus

### Eval Governance Policy (eval_governance_policy.toml)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `lookback` | 30 | Recent artifact window |
| `statuses` | `["success", "save_failed"]` | Status filter |
| `session_capture_min_count` | 1 | Min count for session capture |
| `promotion_repeat_threshold` | 3 | Default promotion threshold |
| `stale_days` | 21 | Days before demotion eligibility |
| `max_promotions_per_apply` | 8 | Max promotions per run |
| `max_session_updates_per_apply` | 6 | Max session updates per run |
| `max_active_items_per_file` | 5 | Max gov markers per file |
| `low_novelty_max` | 0.2 | Novelty threshold |
| `high_repetition_risk_min` | 0.85 | Repetition risk threshold |

## Critical Constraints

1. **Local-only operation** -- Not operable from Codex cloud/browser surfaces
2. **Zero external Python dependencies** for bridge (stdlib-only OSC encoding)
3. **Python 3.10+ required** (uses `tomllib`; fallback to `tomli` on 3.10)
4. **M4L device must be loaded** on a track in Ableton Live while commands are sent
5. **Track 0 is protected** from deletion by `/delete_midi_tracks`
6. **No trained model weights** -- this repo is control layer only
7. **No third-party music** -- memory templates ship blank; all content is user-authored
8. **Pre-1.0.0** -- breaking changes may happen between releases

## External Dependencies

### Python (bridge + memory)
- **None** (stdlib only for bridge OSC encoding)
- `tomli` (only for Python 3.10 TOML fallback)
- `sqlite3` (stdlib, used by retrieval index)

### Max for Live (device)
- LiveAPI (bundled with Ableton Live + Max for Live)
- `Dict` object (Max JS stdlib)
- `udpreceive` / `udpsend` Max objects

### Runtime Requirements
- Ableton Live 12 with Max for Live
- Python 3.10+
- Local UDP access on ports 9000, 9001
- One of: Codex App, Codex CLI, Codex IDE extension

## Memory & Eval Pipeline

```
1. User fills templates     music-preferences/ -> memory/
2. Build retrieval index    python3 -m memory.retrieval index
3. Query context brief      python3 -m memory.retrieval brief --meter 4/4 --bpm 120 --mood contemplative
4. Compose (bridge)         python3 bridge/ableton_udp_bridge.py ...
5. Eval artifact saved      memory/evals/compositions/<date>/<run_id>.json
6. Summarize signals        python3 -m memory.eval_governance summarize --lookback 30
7. Plan updates (dry-run)   python3 -m memory.eval_governance apply --date YYYY-MM-DD --dry-run
8. Apply updates            python3 -m memory.eval_governance apply --date YYYY-MM-DD
```

## Governance Loop

Repeated eval signals are promoted from session notes to stable memory files:

```
Eval Artifacts -> Signal Extraction -> Rule Matching -> Session Capture -> Promotion/Demotion
                                                                              |
                                                        memory/fundamentals/*.md (promoted rules)
                                                        memory/sessions/*.md (session notes)
                                                        memory/archive/demoted_guidance.md (demoted)
                                                        memory/governance/active.md (active summary)
                                                        memory/governance/state.json (state)
```

## Testing

- Framework: `unittest` (stdlib)
- Discovery: `python3 -m unittest discover -s bridge -p "test_*.py"`
- Mock strategy: In-process mocks for socket/select (no live Ableton needed)
- Benchmark: `bridge/benchmark_midi_write.py` (deterministic, no Live needed)
- Smoke test: `bridge/full_surface_smoke_test.py` (requires Live + device loaded)
