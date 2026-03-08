# REPO_INDEX.md -- codex-live-bridge Repository Index

## Quick Reference

| What you need | Where to look |
|---------------|---------------|
| OSC command client / CLI | `bridge/ableton_udp_bridge.py` |
| M4L device JS router | `bridge/m4l/live_udp_bridge.js` |
| M4L device (frozen) | `bridge/m4l/LiveUdpBridge.amxd` |
| M4L patch source | `bridge/m4l/LiveUdpBridge.maxpat` |
| Command reference | `bridge/commands.md` |
| Bridge unit tests | `bridge/test_ableton_udp_bridge.py` |
| Smoke test (requires Live) | `bridge/full_surface_smoke_test.py` |
| MIDI write benchmark | `bridge/benchmark_midi_write.py` |
| Memory index loader + brief | `memory/compositional_memory.py` |
| Retrieval index + search | `memory/retrieval.py` |
| Eval governance loop | `memory/eval_governance.py` |
| Memory index TOML | `music-preferences/index.toml` |
| Governance policy TOML | `music-preferences/eval_governance_policy.toml` |
| Eval artifact index | `music-preferences/evals/composition_index.json` |
| User preference templates | `music-preferences/fundamentals/*.md` |

## Directory Tree

```
codex-live-bridge/
|-- README.md                         (370 lines) Project overview + quickstart
|-- SPEC.md                           (---) Application specification
|-- REPO_INDEX.md                     (---) This file
|-- CLAUDE.md                         (---) Claude Code instructions
|-- CHANGELOG.md                      (26 lines) Release history
|-- CONTRIBUTING.md                   Contribution guide
|-- SUPPORT.md                        Support scope
|-- SECURITY.md                       Vulnerability reporting
|-- LICENSE                           MIT license
|-- .gitignore                        Ignore rules
|
|-- bridge/                           OSC bridge layer
|   |-- ableton_udp_bridge.py         (1210 lines) OSC client CLI + encoder/decoder
|   |-- test_ableton_udp_bridge.py    (223 lines) Unit tests for bridge CLI
|   |-- full_surface_smoke_test.py    (270 lines) End-to-end smoke test (needs Live)
|   |-- benchmark_midi_write.py       (311 lines) Deterministic MIDI write benchmark
|   |-- commands.md                   (290 lines) Command + ACK reference doc
|   |-- m4l/
|       |-- live_udp_bridge.js        (1364 lines) Max JS command router
|       |-- LiveUdpBridge.maxpat      Max patch source (JSON)
|       |-- LiveUdpBridge.amxd        Frozen M4L device
|
|-- memory/                           Compositional memory package
|   |-- __init__.py                   (1 line) Package marker
|   |-- compositional_memory.py       (167 lines) TOML index loader + topic brief CLI
|   |-- retrieval.py                  (766 lines) SQLite retrieval index + search + brief CLI
|   |-- eval_governance.py            (919 lines) Eval-to-memory governance loop CLI
|
|-- music-preferences/                Blank user template pack
|   |-- README.md                     Template pack overview
|   |-- index.toml                    (83 lines) Memory index with 11 fundamentals
|   |-- eval_governance_policy.toml   (55 lines) Governance policy with 5 rules
|   |-- canon.md                      Canon preferences (blank)
|   |-- ensemble.md                   Ensemble preferences (blank)
|   |-- instruments.md                Instrument preferences (blank)
|   |-- moods.md                      Mood palette (blank)
|   |-- fundamentals/
|   |   |-- arrangement.md            Arrangement fundamental (blank)
|   |   |-- evaluation.md             Evaluation fundamental (blank)
|   |   |-- harmony.md                Harmony fundamental (blank)
|   |   |-- key.md                    Key fundamental (blank)
|   |   |-- meter.md                  Meter fundamental (blank)
|   |   |-- mood.md                   Mood fundamental (blank)
|   |   |-- rhythm.md                 Rhythm fundamental (blank)
|   |   |-- silence.md                Silence fundamental (blank)
|   |   |-- tempo.md                  Tempo fundamental (blank)
|   |   |-- timbre.md                 Timbre fundamental (blank)
|   |   |-- velocity.md               Velocity fundamental (blank)
|   |-- governance/
|   |   |-- active.md                 Active governance guidance (generated)
|   |-- instruments/
|   |   |-- instrument_template.md    Per-instrument template
|   |-- sessions/
|   |   |-- SESSION_TEMPLATE.md       Session log template
|   |-- evals/
|   |   |-- README.md                 Eval artifact layout docs
|   |   |-- composition_index.json    Eval artifact index (empty at clone)
|   |   |-- compositions/
|   |       |-- .gitkeep              Placeholder
|   |-- archive/
|       |-- demoted_guidance.md       Demoted governance archive
```

## Module API Reference

### bridge/ableton_udp_bridge.py (1210 lines)

| Symbol | Type | Signature | Purpose |
|--------|------|-----------|---------|
| `OscCommand` | dataclass | `address: str, args: Tuple[OscArg, ...]` | Immutable OSC command |
| `BridgeConfig` | dataclass | 40+ fields | Full CLI configuration |
| `SendMetrics` | dataclass | `command_count, send_durations_ms, ack_wait_durations_ms, ...` | Timing results |
| `parse_args` | function | `(argv: Iterable[str]) -> BridgeConfig` | Parse CLI args |
| `build_commands` | function | `(cfg: BridgeConfig) -> List[OscCommand]` | Build command list from config |
| `encode_osc_message` | function | `(address: str, args: Sequence[OscArg]) -> bytes` | Encode OSC packet |
| `decode_osc_message` | function | `(data: bytes) -> Tuple[str, List[OscArg]]` | Decode OSC packet |
| `send_commands` | function | `(cfg: BridgeConfig, commands: Sequence[OscCommand]) -> SendMetrics` | Send + collect ACKs |
| `open_ack_socket` | function | `(cfg: BridgeConfig) -> socket.socket \| None` | Bind ACK listener |
| `wait_for_acks` | function | `(sock, timeout_s, quiet_window_s) -> List[...]` | Receive ACK packets |
| `summarize_ack` | function | `(address: str, args: Sequence[OscArg]) -> List[str]` | Human-readable ACK summary |
| `main` | function | `(argv: Iterable[str]) -> int` | CLI entry point |

### bridge/m4l/live_udp_bridge.js (1364 lines)

| Symbol | Type | Purpose |
|--------|------|---------|
| `init` / `loadbang` | function | Initialize LiveAPI at `live_set` |
| `ping` | function | Respond with `/ack pong` |
| `tempo` / `sig_num` / `sig_den` | function | Set session tempo/time sig |
| `create_midi_track` / `create_audio_track` | function | Create single track |
| `add_midi_tracks` / `add_audio_tracks` | function | Batch create + rename tracks |
| `delete_midi_tracks` / `delete_audio_tracks` | function | Batch delete tracks |
| `rename_track` | function | Rename track by index |
| `set_session_clip_notes` | function | Destructive clip note write |
| `append_session_clip_notes` | function | Non-destructive clip note append |
| `inspect_session_clip_notes` | function | Report clip note metadata |
| `ensure_midi_tracks` | function | Ensure minimum MIDI track count |
| `midi_cc` / `cc64` | function | Emit MIDI CC messages |
| `status` | function | Report track counts |
| `api_ping` / `api_get` / `api_set` / `api_call` | function | Generic LiveAPI RPC |
| `api_children` / `api_describe` | function | LOM enumeration + introspection |
| `buildNotesDict` / `buildGenericDict` | function | Convert JSON to Max Dict for LiveAPI |
| `parseApiCapabilities` | function | Parse LiveAPI info text for validation |
| `ack` | function | Emit OSC ACK via outlet 0 |

### memory/compositional_memory.py (167 lines)

| Symbol | Type | Signature | Purpose |
|--------|------|-----------|---------|
| `load_index` | function | `(path: Path) -> dict` | Load TOML memory index |
| `list_topics` | function | `(index: Mapping) -> list[str]` | List fundamental names |
| `current_focus` | function | `(index: Mapping) -> str \| None` | Get current focus fundamental |
| `topic_brief` | function | `(index: Mapping, topic: str \| None) -> str` | Generate topic brief text |
| `main` | function | `(argv) -> int` | CLI: `--list`, `--fundamental`, `--topic` |

### memory/retrieval.py (766 lines)

| Symbol | Type | Signature | Purpose |
|--------|------|-----------|---------|
| `ChunkRecord` | dataclass | `chunk_id, path, source, start_line, end_line, text` | Indexed text chunk |
| `SearchHit` | dataclass | `path, source, start_line, end_line, score, snippet` | Search result |
| `RetrievalIndex` | class | `(repo_root, index_rel_path, session_limit)` | SQLite-backed memory index |
| `.rebuild()` | method | `-> dict` | Full index rebuild (markdown + eval) |
| `.status()` | method | `-> dict` | Index stats (chunk counts, FTS status) |
| `.search()` | method | `(query, max_results, min_score) -> list[SearchHit]` | FTS5 or fallback search |
| `.read_window()` | method | `(rel_path, start_line, lines) -> dict` | Read memory file snippet (sandboxed) |
| `.brief()` | method | `(meter, bpm, mood, key_name, focus, max_results) -> str` | Generate context brief |
| `build_context_brief` | function | `(**kwargs) -> str` | Convenience wrapper |
| `main` | function | `(argv) -> int` | CLI: `index`, `status`, `search`, `get`, `brief` |

### memory/eval_governance.py (919 lines)

| Symbol | Type | Signature | Purpose |
|--------|------|-----------|---------|
| `load_policy` | function | `(path: Path \| None) -> dict` | Load + validate governance policy TOML |
| `load_recent_eval_artifacts` | function | `(repo_root, lookback) -> list[dict]` | Load recent eval JSON artifacts |
| `build_governance_signals` | function | `(artifacts, policy) -> dict` | Extract governance signals from artifacts |
| `plan_memory_actions` | function | `(signals, policy, date_text) -> dict` | Plan session + promotion actions |
| `apply_memory_actions` | function | `(actions, repo_root, dry_run) -> dict` | Execute memory file updates |
| `main` | function | `(argv) -> int` | CLI: `summarize`, `apply` (with `--dry-run`) |

## Data Flow Diagram

```
                          User / Codex
                              |
                    +---------+---------+
                    |                   |
              [Bridge Flow]       [Memory Flow]
                    |                   |
         Python CLI (bridge/)    Python CLI (memory/)
                    |                   |
          OSC encode + UDP       TOML/JSON/SQLite
           :9000 (send)           local files
                    |                   |
         M4L Device (JS)        Retrieval Index
           :9001 (ack)           (SQLite FTS5)
                    |                   |
          LiveAPI (LOM)          Eval Artifacts
                    |            (JSON under
         Ableton Live Set        memory/evals/)
                    |                   |
              [Track/Clip          Governance
               mutations]          Loop (eval_
                    |              governance.py)
              Eval artifact            |
              (optional)         Memory updates
                    |            (markdown files)
                    +-------+-------+
                            |
                     Iteration cycle
```

## Inter-Module Dependency Graph

```
bridge/ableton_udp_bridge.py
  <- bridge/test_ableton_udp_bridge.py (imports as `bridge`)
  <- bridge/full_surface_smoke_test.py (imports as `bridge`)
  <- bridge/benchmark_midi_write.py (imports compose_kick_pattern, compose_arrangement -- external)

memory/compositional_memory.py
  <- tomllib (stdlib 3.11+) or tomli (3.10 fallback)

memory/retrieval.py
  <- sqlite3 (stdlib)
  <- hashlib (stdlib)
  <- json (stdlib)

memory/eval_governance.py
  <- tomllib / tomli
  <- json (stdlib)
  <- re (stdlib)

music-preferences/index.toml
  <- memory/compositional_memory.py (read at runtime)

music-preferences/eval_governance_policy.toml
  <- memory/eval_governance.py (read at runtime)
```

Note: `bridge/benchmark_midi_write.py` imports `compose_kick_pattern` and `compose_arrangement` which are not present in this repo -- they are workflow scripts expected in the user's local environment.

## Configuration File Inventory

| File | Format | Read By | Purpose |
|------|--------|---------|---------|
| `music-preferences/index.toml` | TOML | `compositional_memory.py` | Fundamental definitions + current focus |
| `music-preferences/eval_governance_policy.toml` | TOML | `eval_governance.py` | Governance rules, thresholds, defaults |
| `music-preferences/evals/composition_index.json` | JSON | `retrieval.py`, `eval_governance.py` | Eval artifact registry |
| `memory/governance/state.json` | JSON | `eval_governance.py` | Governance state (promoted signals) |

## Keeping This Index Current

Update this file when you:
- Add, remove, or rename source files
- Add or change public function signatures
- Modify the command surface (new OSC commands, ACK shapes)
- Change configuration file formats or add new config files
- Alter the data flow or module dependencies
- Change the directory structure
