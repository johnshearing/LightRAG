# Plan: Indexing Electrical Schematics with LightRAG for Troubleshooting

> **Stand-alone project brief.** This file is designed to be the *single source of context*
> for continuing this project in a fresh Claude Code session. The root `CLAUDE.md` contains
> only a pointer to this file. Read this document top-to-bottom, then read the files listed
> in **§7 Files to review before writing code**, before creating any code or content.

- **Created / last substantive update:** 2026-07-23
- **Status:** Planning complete. No project code written yet. Next action is **§8 Step 1**
  (draft the extraction skill in `schematic_skills/`) — awaiting user go-ahead.
- **Primary working directory:** `/home/js/LightRAG-Dev/jrs`
- **Repo root:** `/home/js/LightRAG-Dev` (this is the LightRAG framework itself; `jrs/` is
  John's working area on top of it).

---

## 0. How to use this document (for a fresh session)

1. Read §1–§6 for full context, decisions, and rationale.
2. Read the files in §7 (especially the sample PDF, `construction_skills/`, the two prior
   `circuit_logic.json` attempts, and the verified `ainsert_custom_kg` schema).
3. Pick up at §8 "Open Items / Next Steps."
4. Honor the environment facts in §1 (LightRAG init requirement; **query in `hybrid`
   mode** per user preference).

---

## 1. Environment & Project Facts

- **What LightRAG-Dev/jrs does:** builds RAG-based AI *troubleshooting experts*. Typical
  flow: index technical manuals, then query the index for troubleshooting advice.
- **This sub-project's goal:** make electrical **schematics** (PDF) index well so the
  knowledge base can answer accurately about the *actual circuit* the schematic represents
  — i.e., use schematic connectivity to help troubleshoot.
- **User:** John Shearing (johnshearing@gmail.com).
- **User preference (from persistent memory):** *always query this knowledge base in
  `hybrid` mode.*
- **LightRAG usage rule (from root `CLAUDE.md`):** after constructing a `LightRAG` instance
  you MUST `await rag.initialize_storages()` (and `initialize_pipeline_status()`); finalize
  with `await rag.finalize_storages()`. Forgetting this is the #1 error.
- **Embedding rule:** embedding model must stay consistent; changing it requires clearing
  vector storage. Prior scripts use `text-embedding-3-large`, dim **3072**.
- **Storage:** default file-based (JSON/NetworkX) in a `working_dir`. A `working_dir`
  already exists for mod_linx (see §7).
- **Stack:** Python, async throughout, `uv` for env, `ruff` for lint. LLM calls via
  `lightrag.llm.openai` (prior work uses `gpt-4o-mini` for text, `gpt-4o` for vision).

---

## 2. Problem Statement & History (how we got here)

### 2.1 The core problem
Electrical schematics in PDF do not index well with LightRAG's standard approach. When the
whole drawing is handed to a vision model + LLM extraction, only a small fraction of the
components and connections are captured, and entity properties/relationships are incomplete.

### 2.2 Sample artifact
`work/mod_linx/mod_linx_data/PS20115MLM4-2.pdf` — a **true vector PDF, 1 page** (not a scan).
It is the Mod-Linx Power Supply Assembly schematic: 24VDC 20A output, 115VAC 1-phase input,
Master 4 drive cards, revision D, Convex Corp. Because it is vector-based, **every text
label (component tags, terminal numbers, wire color + gauge) and every wire line is in the
vector layer and can be extracted losslessly** with PyMuPDF — this fact drives the whole
plan.

### 2.3 Two prior indexing attempts (both incomplete — do not repeat as-is)
- **`_1_ra_index.py`** — the *AI-does-the-indexing* path: **RAGAnything** (vision reads the
  whole PDF → LightRAG LLM extracts entities/relations). **This produced the incomplete
  `circuit_logic.json` and only found a small portion of entities.** Root cause: one global
  vision pass cannot enumerate ~50 components and ~150 wires on a dense D-size sheet, and
  extraction then runs on lossy caption text rather than the vector data.
- **`_1_custom_index_01.py`** — the *you-build-the-graph* path: calls
  `rag.ainsert_custom_kg(docs, ...)`, injecting a knowledge graph (entities + relationships
  + chunks) built by us and **skipping LLM extraction entirely**. This is the mechanism we
  will use for the schematic.

### 2.4 Methods considered for reading schematics
1. **OpenCV feature enhancement** — enhances an *image*, emits no text. Only relevant later
   for **scanned** schematics (explicitly out of scope for now). Not standalone.
2. **YOLO symbol detection** — yields *nodes* (symbol locations) but **not edges**
   (connectivity); needs an expensive hand-labeled dataset; symbols vary by vendor/standard.
   At best a component *inside* Method 3.
3. **Adapt `construction_skills` → `schematic_skills`** (vector extraction + AI-vision
   verification). **← CHOSEN.** Output is text/JSON, natively indexable by LightRAG.

**Decision:** pursue **Method 3**. Use OpenCV/YOLO only as optional assists inside it
(OpenCV for future scanned drawings; YOLO only if text-less symbols become a real gap).

---

## 3. Key Decisions (the plan in brief)

1. **Extraction approach = Method 3.** Deterministic PyMuPDF vector extraction + a netlist
   *tracer* + AI-vision *verification on cropped tiles* (never one global pass).
2. **Indexing method = `ainsert_custom_kg` for the schematic.** Do **not** let the LLM
   extract the schematic graph — a schematic is a deterministic netlist; completeness must
   be a property of our script, not of the LLM's attention. We control and can audit every
   node/edge.
3. **AI stays upstream, not in-loop:** vision helps *build and verify* the JSON before
   indexing; it does not build the graph inside LightRAG.
4. **Troubleshooting manual → normal AI extraction** (`ainsert`) into the **same
   working_dir**, so its semantic knowledge merges into the same graph.
5. **⚠️ Entity names must be identical across sources.** Use the drawing's exact designators
   (e.g. `CR-BP`) as canonical entity names; put the plain-language phrase ("bypass relay")
   in `aliases`/`description` so the manual's prose links to the same node. Mismatched names
   → disconnected graph → queries won't traverse symptom → component → wiring.
6. **Query in `hybrid` mode** (user preference).

### What makes query answers *accurate*
- Attach a **prose chunk** (one descriptive sentence) to every entity and relationship;
  LightRAG retrieves chunks + graph together, and terse JSON gives weak vector matches.
- Two relationship types turn a static netlist into a *troubleshooting* graph:
  - **`ON_NET`** — a fault propagates across every point on a net, not just one wire's ends.
  - **`COIL_CONTROLS_CONTACT`** — makes control logic explicit ("PB1 pressed → CR1 coil
    energizes → contacts 11-14 close → RUN signal to cards") instead of leaving it to be
    inferred.

### Reuse boundary from `construction_skills`
- **Reusable:** `extract.py`'s vector-extraction backbone (text-with-coords, lines,
  polylines) and the hybrid extraction + vision-verification workflow.
- **NOT reusable:** the tag dictionary (`L1`, `GPO`, `WC`…) and all scale/area/volume math —
  those serve *quantity takeoff*, a different task.
- **New work:** an interpretation layer that is a **netlist tracer** (associate wire labels
  to segments, follow segments through junctions, bind endpoints to nearest terminal, infer
  nets), not a counter. New skills live in `/home/js/LightRAG-Dev/jrs/schematic_skills`
  (currently empty).

---

## 4. Target Pipeline

```
PDF (vector)
  │
  ├─ (a) PyMuPDF vector extraction  → text labels + coords, lines/polylines   [deterministic]
  ├─ (b) Netlist tracer             → components, terminals, wires, NETS       [deterministic]
  ├─ (c) Tiled AI-vision verify     → classify symbols, confirm/repair edges   [AI, upstream]
  ├─ (d) Enriched circuit_logic.json (master, human-auditable)                 [schema §5]
  ├─ (e) Transform → LightRAG custom-KG {chunks, entities, relationships}      [schema §6]
  ├─ (f) ainsert_custom_kg()        → schematic graph (no LLM extraction)
  └─ (g) normal ainsert() of troubleshooting manual into SAME working_dir      [AI extraction]
        → merged graph; query in hybrid mode
```

---

## 5. Enriched `circuit_logic.json` — Full Schema

The **master, human-auditable** artifact the extraction skill must produce. It is a
*superset* of LightRAG's custom-KG format; a transform (§6) flattens it into
`{chunks, entities, relationships}`.

### 5.1 Top-level structure
```jsonc
{
  "drawing": { /* metadata — one object */ },
  "components": [ /* physical devices */ ],
  "terminals":  [ /* connection points */ ],
  "nets":       [ /* electrically-common nodes */ ],
  "wires":      [ /* physical conductors */ ],
  "cables":     [ /* harnesses grouping wires */ ],
  "subsystems": [ /* functional groups */ ],
  "relationships": [ /* typed edges beyond the raw netlist */ ]
}
```

### 5.2 Field definitions

**`drawing`** (object)
| Field | Type | Notes |
|---|---|---|
| `drawing_number` | string | e.g. `PS20115MLM4-2` |
| `revision` | string | e.g. `D` |
| `title` | string | full title-block text |
| `date` | string | ISO if parseable |
| `assembly` | string | what it documents |
| `proprietary_notice` | string | e.g. "Convex Corp confidential" |
| `references` | string[] | other drawings (`MXCS-P9`, `MXCS-P11`) |
| `notes` | string[] | free-text drawing notes |

**`components[]`** (physical devices)
| Field | Type | Notes |
|---|---|---|
| `id` | string | **canonical** designator (`CR-BP`) |
| `class` | enum | relay / circuit_breaker / fuse / terminal_block / push_button / power_supply / speed_controller / connector_receptacle / ground / drive_card / motor / cable_plug |
| `description` | string | human-readable |
| `ratings` | object | `{voltage, current, poles, ...}` as available |
| `function` | string | role in the circuit |
| `power_domain` | enum | 115VAC / 24VDC / 0V_common / control_signal |
| `normal_state` | string | energized/de-energized; NO/NC — deviation is the fault |
| `location` | object | `{x, y, zone}` on the drawing (citation + "where is it") |
| `part_number` | string\|null | if present |
| `manufacturer` | string\|null | if present |
| `aliases` | string[] | plain-language names so the manual links up |

**`terminals[]`** (connection points)
| Field | Type | Notes |
|---|---|---|
| `id` | string | `COMPONENT:TERMINAL`, e.g. `CR-BP:A1` |
| `parent_component` | string | component `id` |
| `function` | enum | coil / NO_contact / NC_contact / common / line / neutral / ground / input / output |
| `net` | string\|null | net `id` this terminal is tied to |

**`nets[]`** (electrically-common node — the troubleshooting backbone; the drawing labels
these: `24V`, `0V`, `110`, `120`, `121`, `125`, `130`, `N-1`, `L1-A`, `SPD`, `RUN`, `DIR`,
`24E-1`)
| Field | Type | Notes |
|---|---|---|
| `id` | string | net label |
| `signal_type` | enum | power / ground / control |
| `nominal_voltage` | string\|null | e.g. `24VDC`, `115VAC`, `0V` |
| `member_terminals` | string[] | every terminal on the net |

**`wires[]`** (physical conductors)
| Field | Type | Notes |
|---|---|---|
| `id` | string | wire id |
| `color` | string | split from old `wire_label` |
| `gauge` | string | e.g. `18AWG` |
| `from_terminal` | string | terminal `id` |
| `to_terminal` | string | terminal `id` |
| `cable` | string\|null | parent cable `id` |
| `net` | string\|null | net `id` |

**`cables[]`** — `{ "id": string, "description": string, "member_wires": string[] }`

**`subsystems[]`** — `{ "id": string, "description": string, "member_components": string[] }`

**`relationships[]`** (typed edges beyond the raw wire netlist)
| Field | Type | Notes |
|---|---|---|
| `type` | enum | HAS_TERMINAL / CONNECTS_TO / ON_NET / POWERS / PROTECTS / ACTUATES / COIL_CONTROLS_CONTACT / PART_OF / GROUNDED_TO / REFERENCES |
| `src` | string | source entity id |
| `tgt` | string | target entity id |
| `description` | string | prose (becomes a chunk) |
| `properties` | object | e.g. `{wire_color, wire_gauge, net}` for CONNECTS_TO |

### 5.3 Worked example (Mod-Linx `PS20115MLM4-2`, partial)

> NOTE: wire IDs, some terminal numbers, and the date below are **illustrative
> placeholders**; the extraction skill produces the real values from the drawing. The
> *shape* is the concrete target.

```jsonc
{
  "drawing": {
    "drawing_number": "PS20115MLM4-2",
    "revision": "D",
    "title": "MOD-LINX POWER SUPPLY ASSY. 24VDC 20AMP OUTPUT, 115VAC 1 PHASE INPUT, MASTER 4 DRIVE CARDS, SCHEMATIC",
    "date": "2007-05-29",
    "assembly": "Mod-Linx Power Supply Assembly, Master 4 Drive Cards",
    "proprietary_notice": "Confidential property of Convex Corp.",
    "references": ["MXCS-P9", "MXCS-P11"],
    "notes": [
      "Keep all DC wires 4\" minimum clearance from 115VAC wires.",
      "Individual wires to all points of termination; do not double place.",
      "Label all wires and cables 1\" from end."
    ]
  },

  "components": [
    {
      "id": "CB1", "class": "circuit_breaker",
      "description": "20 A main circuit breaker on the 115VAC input line side.",
      "ratings": {"current": "20A", "poles": 1},
      "function": "Protects the 115VAC input feed to the power supply.",
      "power_domain": "115VAC", "normal_state": "closed (ON)",
      "location": {"x": 512, "y": 60, "zone": "top-center"},
      "part_number": null, "manufacturer": null,
      "aliases": ["20 amp breaker", "main breaker"]
    },
    {
      "id": "PS1", "class": "power_supply",
      "description": "24VDC 20A power supply converting 115VAC to 24VDC.",
      "ratings": {"input": "115VAC", "output": "24VDC", "current": "20A"},
      "function": "Supplies 24VDC to control circuits and drive cards.",
      "power_domain": "24VDC", "normal_state": "energized",
      "location": {"x": 640, "y": 55, "zone": "top-center"},
      "part_number": null, "manufacturer": null,
      "aliases": ["24V power supply", "PSU"]
    },
    {
      "id": "CR-BP", "class": "relay",
      "description": "Bypass relay. When energized, its contacts bypass the start/stop circuit.",
      "ratings": {"coil_voltage": "24VDC"},
      "function": "Bypasses PB start/stop control when energized.",
      "power_domain": "24VDC", "normal_state": "de-energized",
      "location": {"x": 1180, "y": 690, "zone": "bottom-right"},
      "part_number": null, "manufacturer": null,
      "aliases": ["bypass relay"]
    },
    {
      "id": "PB1", "class": "push_button",
      "description": "Brown 22AWG start/stop switch #1 with start/stop cable.",
      "ratings": {},
      "function": "Operator start/stop control feeding relay CR1.",
      "power_domain": "control_signal", "normal_state": "open (not pressed)",
      "location": {"x": 180, "y": 250, "zone": "left"},
      "part_number": null, "manufacturer": null,
      "aliases": ["start/stop switch 1", "start stop button 1"]
    }
  ],

  "terminals": [
    {"id": "CB1:1",    "parent_component": "CB1",   "function": "line",   "net": "L1"},
    {"id": "CB1:2",    "parent_component": "CB1",   "function": "output", "net": "L1-A"},
    {"id": "CR-BP:A1", "parent_component": "CR-BP", "function": "coil",   "net": "NET-125"},
    {"id": "CR-BP:A2", "parent_component": "CR-BP", "function": "coil",   "net": "NET-0V"},
    {"id": "PB1:2",    "parent_component": "PB1",   "function": "output", "net": "NET-PB1-SP"}
  ],

  "nets": [
    {"id": "NET-24V", "signal_type": "power",  "nominal_voltage": "24VDC",
     "member_terminals": ["PS1:+24V", "CB2:1"]},
    {"id": "NET-0V",  "signal_type": "ground", "nominal_voltage": "0V",
     "member_terminals": ["PS1:0V", "CR-BP:A2"]},
    {"id": "NET-125", "signal_type": "control","nominal_voltage": "24VDC",
     "member_terminals": ["CR-BP:A1", "CB-BYPASS:2"]}
  ],

  "wires": [
    {"id": "W001", "color": "BLACK", "gauge": "10AWG",
     "from_terminal": "PLG1:L1", "to_terminal": "CB1:1", "cable": null, "net": "L1"},
    {"id": "W042", "color": "BLUE", "gauge": "18AWG",
     "from_terminal": "CB-BYPASS:2", "to_terminal": "CR-BP:A1", "cable": "24E-1", "net": "NET-125"}
  ],

  "cables": [
    {"id": "24E-1", "description": "Blue 18AWG control cable bundle", "member_wires": ["W042"]},
    {"id": "START-STOP-CABLE-1", "description": "PB1 start/stop cable (brown/white/blue 22AWG)", "member_wires": []}
  ],

  "subsystems": [
    {"id": "SUB-115VAC-INPUT", "description": "115VAC single-phase input power section",
     "member_components": ["PLG1", "CB1"]},
    {"id": "SUB-24VDC-SUPPLY", "description": "24VDC power supply section",
     "member_components": ["PS1", "CB2"]},
    {"id": "SUB-BYPASS", "description": "Bypass-relay circuit",
     "member_components": ["CR-BP", "CB-BYPASS"]}
  ],

  "relationships": [
    {"type": "HAS_TERMINAL", "src": "CR-BP", "tgt": "CR-BP:A1",
     "description": "Relay CR-BP has coil terminal A1.", "properties": {}},
    {"type": "CONNECTS_TO", "src": "CB-BYPASS:2", "tgt": "CR-BP:A1",
     "description": "Bypass breaker terminal 2 connects to CR-BP coil terminal A1 via a BLUE 18AWG wire on net 125.",
     "properties": {"wire_color": "BLUE", "wire_gauge": "18AWG", "net": "NET-125"}},
    {"type": "ON_NET", "src": "CR-BP:A1", "tgt": "NET-125",
     "description": "CR-BP coil terminal A1 is on control net 125 (24VDC).", "properties": {}},
    {"type": "PROTECTS", "src": "CB1", "tgt": "SUB-24VDC-SUPPLY",
     "description": "CB1 (20A) protects the 115VAC feed to the 24VDC supply.", "properties": {}},
    {"type": "POWERS", "src": "PS1", "tgt": "NET-24V",
     "description": "PS1 supplies 24VDC to net 24V.", "properties": {}},
    {"type": "ACTUATES", "src": "PB1", "tgt": "CR1",
     "description": "Push button PB1 actuates relay CR1's coil.", "properties": {}},
    {"type": "COIL_CONTROLS_CONTACT", "src": "CR1", "tgt": "CR1:11-14",
     "description": "Energizing CR1's coil closes its normally-open contacts 11-14, sending the RUN signal to the drive cards.",
     "properties": {}},
    {"type": "REFERENCES", "src": "PS20115MLM4-2", "tgt": "MXCS-P9",
     "description": "This drawing references external device connection drawing MXCS-P9.", "properties": {}}
  ]
}
```

---

## 6. Mapping to LightRAG `ainsert_custom_kg` (VERIFIED against source)

`ainsert_custom_kg(custom_kg: dict, full_doc_id: str = None)` is defined at
**`lightrag/lightrag.py:2342`**. It expects a dict with three keys. Verified field
requirements (from the implementation, ~lines 2350–2460):

**`chunks[]`**
- `content` — **required**.
- `source_id` — **required** (a string you choose; it is the join key entities/relationships
  reference).
- `file_path` — optional (default `"custom_kg"`).
- `chunk_order_index` — optional (default 0).
- Chunk IDs are content-hashed internally.

**`entities[]`**
- `entity_name` — **required** (this becomes the graph node id → use canonical designators).
- `entity_type` — optional (default `"UNKNOWN"`).
- `description` — optional (default `"No description provided"`).
- `source_id` — **must equal a chunk's `source_id`**; otherwise logs
  `"... has an UNKNOWN source_id"` and the entity is orphaned from its text.
- `file_path` — optional.

**`relationships[]`**
- `src_id`, `tgt_id` — **required**. If a referenced node doesn't exist yet, it is
  auto-created as an `UNKNOWN` node — so define entities first / for every id you reference.
- `description` — **required**.
- `keywords` — **required** (accessed with no default — omitting it raises `KeyError`).
- `weight` — optional (default 1.0).
- `source_id` — should match a chunk's `source_id` (same rule as entities).

**Transform rules (master §5 → custom-KG):**
- **entities** ← every `components[]`, `terminals[]`, `nets[]`, `wires[]`, `cables[]`,
  `subsystems[]`, plus `drawing`. `entity_name` = `id`; `entity_type` = `class`/type;
  `description` = the object's `description`.
- **relationships** ← every `relationships[]` entry. `src_id`/`tgt_id` = `src`/`tgt`;
  `keywords` = `type` + flattened `properties`; `weight` = 1.0.
- **chunks** ← one prose chunk per entity and per relationship (use its `description`), each
  with a unique `source_id`; set the entity/relationship `source_id` to that same string so
  the join works.
- Keep the rich §5 `circuit_logic.json` as the human-auditable master; **generate** the
  custom-KG JSON from it (don't hand-maintain both).

Adapt `_1_custom_index_01.py` (which already calls `ainsert_custom_kg`) as the indexing
script; point its `WORKING_DIR` at the mod_linx work dir and its embedding config at
`text-embedding-3-large` / dim 3072.

---

## 7. Files to review before writing code

All paths verified to exist on 2026-07-23 unless noted.

| Path | Why review it |
|---|---|
| `/home/js/LightRAG-Dev/CLAUDE.md` | Project-wide rules (init requirement, storage, style, query modes). |
| `/home/js/LightRAG-Dev/jrs/work/mod_linx/mod_linx_data/PS20115MLM4-2.pdf` | The target schematic (vector PDF). The concrete extraction target. |
| `/home/js/LightRAG-Dev/jrs/construction_skills/SKILL.md` | Template skill; hybrid extraction+vision workflow to adapt. |
| `/home/js/LightRAG-Dev/jrs/construction_skills/scripts/extract.py` | Reusable PyMuPDF vector-extraction backbone (text coords, lines, polylines). |
| `/home/js/LightRAG-Dev/jrs/construction_skills/references/tags_and_symbols.md` | Example of a tag/symbol reference doc; schematic version needs new content. |
| `/home/js/LightRAG-Dev/jrs/schematic_skills/` | **Empty** — destination for the new skill. |
| `/home/js/LightRAG-Dev/circuit_logic.json` | **Prior attempt #1** (~18 KB) from `_1_ra_index.py`; incomplete — study its gaps. |
| `/home/js/LightRAG-Dev/jrs/work/mod_linx/mod_linx_data/circuit_logic.json` | Prior connection-list sample (edges only, ~88 lines); shows the old flat format we are superseding. |
| `/home/js/LightRAG-Dev/jrs/_1_ra_index.py` | The AI-extraction indexing path that missed entities (what NOT to repeat). |
| `/home/js/LightRAG-Dev/jrs/_1_custom_index_01.py` | The `ainsert_custom_kg` path to adapt for indexing the schematic KG. |
| `/home/js/LightRAG-Dev/jrs/_2_ra_query_text.py` | Existing text-query script; reuse for testing answers (hybrid mode). |
| `/home/js/LightRAG-Dev/jrs/_2_ra_query_image.py` | Existing image-query script; reference. |
| `/home/js/LightRAG-Dev/jrs/_0_interesting_queries.md` | Sample queries; use to define acceptance tests for troubleshooting answers. |
| `/home/js/LightRAG-Dev/lightrag/lightrag.py` (line 2342) | Ground-truth `ainsert_custom_kg` schema (see §6). |
| `/home/js/LightRAG-Dev/jrs/work/mod_linx/mod_linx_data/__enqueued__/Troubleshooting Mod-Linx Conveyors.pdf` | Prose manual to index (normal AI extraction) into the same working_dir; entity names must match the schematic KG. |
| `/home/js/LightRAG-Dev/jrs/work/mod_linx/mod_linx_work_dir/` | Existing LightRAG working dir for mod_linx (JSON/GraphML storage). A `_Backup` copy also exists. |

---

## 8. Open Items / Next Steps

1. **Draft the extraction skill** in `schematic_skills/` targeting the §5 schema
   (netlist tracer + tiled vision verification). *Not started — awaiting go-ahead. This is
   the intended next action.*
2. Decide **junction-vs-crossover** handling (real junction vs. wires that cross but don't
   connect) — the main accuracy risk.
3. Handle **off-page / bus nets** (`MXCS-P9`/`MXCS-P11` external connections; ground/power
   rails) so the graph doesn't fragment.
4. Assess how accurate the **existing `circuit_logic.json`** files are (how much tracing is
   already solved) before rebuilding.
5. Build the **master→custom-KG transform** and the `ainsert_custom_kg` indexing script
   (adapt `_1_custom_index_01.py`).
6. **Ingest the troubleshooting manual** into the same working_dir with matching entity
   names; verify graph linkage.
7. Define **acceptance tests**: a set of troubleshooting questions (seed from
   `_0_interesting_queries.md`) with expected-answer criteria; query in `hybrid` mode.

### Accuracy risks to keep in mind
- **Junctions vs. crossovers** — wrong here = wrong netlist = confidently wrong answers.
- **Label→wire / label→terminal association** is a proximity heuristic; ambiguous on dense
  sheets.
- **Off-page / bus nets** fragment the graph if not handled explicitly.
- Vision verification mitigates but does not eliminate these — a QA pass belongs in the plan.

---

## 9. Glossary
- **Netlist** — the list of electrical connections: which component/terminal connects to
  which, and on what net.
- **Net** — a set of terminals that are electrically common (same node). The drawing labels
  many nets with numbers (110, 120, 125, 130…) and names (24V, 0V, RUN, DIR, SPD).
- **Custom KG** — a knowledge graph (`{chunks, entities, relationships}`) injected directly
  via `ainsert_custom_kg`, bypassing LLM extraction.
- **Contact / coil** — a relay has a *coil* (energize to actuate) and *contacts* (NO/NC)
  that switch other circuits. `COIL_CONTROLS_CONTACT` captures this dependency.
