# Plan: Indexing Electrical Schematics with LightRAG for Troubleshooting

**Date:** 2026-07-23
**Status:** Planning only — no code to be written yet.
**Goal:** Turn electrical schematics (starting with the Mod-Linx power supply drawing
`PS20115MLM4-2`) into a LightRAG knowledge base that answers troubleshooting questions
accurately about the *actual circuit* the schematic represents.

---

## 1. Context & Problem

- `LightRAG-Dev/jrs` builds RAG-based AI troubleshooting experts by indexing technical
  manuals and querying the index.
- Electrical schematics in PDF do **not** index well with the standard approach.
- Sample drawing: `/home/js/LightRAG-Dev/jrs/work/mod_linx/mod_linx_data/PS20115MLM4-2.pdf`
  (a true **vector** PDF, 1 page — not a scan; all text labels and wire lines are in the
  vector layer and can be extracted losslessly).

### Prior attempts
- `_1_ra_index.py` — uses **RAGAnything** (vision reads the whole PDF, then LightRAG's LLM
  extracts entities/relations). **Failure mode observed:** only a small fraction of the
  entities were found, and entity properties/relationships were incomplete. This is
  inherent: one global vision pass cannot enumerate ~50 components and ~150 wires on a
  dense D-size sheet, and LLM extraction runs on lossy caption text rather than the vector
  data.
- `_1_custom_index_01.py` — uses `rag.ainsert_custom_kg(docs, ...)`, which injects a
  knowledge graph (entities + relationships + chunks) **you build yourself** and **skips
  the LLM extraction step**.
- `circuit_logic.json` (an edges-only connection list) was a first attempt produced via
  `_1_ra_index.py` — incomplete, and lacks node properties, nets, and control logic.

---

## 2. Key Decisions

### 2.1 Candidate methods evaluated
Three methods were considered for producing indexable text/JSON from schematics:

1. **OpenCV feature enhancement** — enhances an *image*; emits no text. Only relevant later
   for **scanned** schematics. Not a standalone method. (Scanned inputs are out of scope
   for now.)
2. **YOLO symbol detection** — gives *nodes* (symbol locations) but **not edges**
   (connectivity), needs an expensive hand-labeled training set, and symbols vary by
   vendor/standard. At best a *component inside* Method 3, not a method on its own.
3. **Adapt the `construction_skills` into `schematic_skills`** (vector extraction + AI
   vision verification) → **CHOSEN**. The PDF is vector-based, so PyMuPDF can extract every
   text label (component tags, terminal numbers, wire color+gauge) at exact coordinates,
   plus every line/polyline (the wires). This is the raw material for a netlist. Output is
   text/JSON — natively indexable by LightRAG.

**Decision: pursue Method 3.** Treat OpenCV/YOLO only as optional assists *inside* Method 3
(OpenCV for future scanned drawings; YOLO only if text-less symbols become a real gap).

### 2.2 Reuse boundary from `construction_skills`
- **Reusable:** the `extract.py` vector-extraction backbone (text-with-coordinates, lines,
  polylines) and the hybrid extraction + vision-verification workflow.
- **NOT reusable:** the tag dictionary (`L1`, `GPO`, `WC`…) and all scale/area/volume math.
  Those are for *quantity takeoff*, a different task.
- **New work:** the interpretation layer must become a **netlist tracer** (associate wire
  labels to segments, follow segments through junctions, bind endpoints to the nearest
  terminal, infer nets), not a counter.
- New skills live in `/home/js/LightRAG-Dev/jrs/schematic_skills`.

### 2.3 Indexing method — DECISIVE
- **Use `ainsert_custom_kg` for the schematic's structure.** Do **not** let the LLM extract
  the graph from the drawing. A schematic is a deterministic netlist; completeness must be a
  property of our extraction script, not of the LLM's attention. We control and can audit
  every node/edge.
- **Keep the AI upstream, not in-loop:** use vision on **cropped tiles** of the drawing
  (not the whole sheet) to classify symbols and **verify** the script-built netlist.
- **Use normal AI extraction for the prose troubleshooting manual**
  (`Troubleshooting Mod-Linx Conveyors.pdf`) into the **same** working_dir, so semantic
  knowledge merges into the same graph.
- ⚠️ **Critical:** entity names must be **identical** across both sources (schematic KG and
  manual). Use the drawing's exact designators (e.g. `CR-BP`) as canonical names and add
  the plain-language phrase ("bypass relay") as an alias/in the description so the manual's
  text links to the same node. Mismatched names create disconnected graphs and queries
  won't traverse symptom → component → wiring.

### 2.4 What makes queries *accurate*
- Attach a **prose chunk** (one descriptive sentence) to every entity and relationship;
  LightRAG retrieves chunks + graph together, and terse JSON alone gives weak vector
  matches.
- Two relationship types turn a static netlist into a *troubleshooting* graph:
  - **`ON_NET`** — so a fault propagates across every point on a net, not just the two ends
    of one wire.
  - **`COIL_CONTROLS_CONTACT`** — so control logic ("PB1 pressed → CR1 coil energizes →
    contacts 11-14 close → RUN signal to cards") is explicit, not inferred.

---

## 3. Pipeline (target architecture)

```
PDF (vector)
  │
  ├─ (a) PyMuPDF vector extraction  → text labels + coords, lines/polylines   [deterministic]
  │
  ├─ (b) Netlist tracer             → components, terminals, wires, NETS       [deterministic]
  │
  ├─ (c) Tiled AI-vision verify     → classify symbols, confirm/repair edges   [AI, upstream]
  │
  ├─ (d) Enriched circuit_logic.json (master, human-auditable)                 [see schema §4]
  │
  ├─ (e) Transform → LightRAG custom-KG JSON {chunks, entities, relationships}
  │
  ├─ (f) ainsert_custom_kg()        → schematic graph (no LLM extraction)
  │
  └─ (g) normal ainsert() of troubleshooting manual into SAME working_dir      [AI extraction]
        → merged graph; query with hybrid mode
```

*Note: per project memory, always query this knowledge base in `hybrid` mode.*

---

## 4. Enriched `circuit_logic.json` — Full Schema

This is the **master, human-auditable** artifact the extraction skill must produce. It is a
*superset* of LightRAG's custom-KG format; a small transform flattens it into
`{chunks, entities, relationships}` for `ainsert_custom_kg`.

### 4.1 Top-level structure

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

### 4.2 Field definitions

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

### 4.3 Worked example (Mod-Linx `PS20115MLM4-2`, partial)

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
      "id": "CB1",
      "class": "circuit_breaker",
      "description": "20 A main circuit breaker on the 115VAC input line side.",
      "ratings": {"current": "20A", "poles": 1},
      "function": "Protects the 115VAC input feed to the power supply.",
      "power_domain": "115VAC",
      "normal_state": "closed (ON)",
      "location": {"x": 512, "y": 60, "zone": "top-center"},
      "part_number": null, "manufacturer": null,
      "aliases": ["20 amp breaker", "main breaker"]
    },
    {
      "id": "PS1",
      "class": "power_supply",
      "description": "24VDC 20A power supply converting 115VAC to 24VDC.",
      "ratings": {"input": "115VAC", "output": "24VDC", "current": "20A"},
      "function": "Supplies 24VDC to control circuits and drive cards.",
      "power_domain": "24VDC",
      "normal_state": "energized",
      "location": {"x": 640, "y": 55, "zone": "top-center"},
      "part_number": null, "manufacturer": null,
      "aliases": ["24V power supply", "PSU"]
    },
    {
      "id": "CR-BP",
      "class": "relay",
      "description": "Bypass relay. When energized, its contacts bypass the start/stop circuit.",
      "ratings": {"coil_voltage": "24VDC"},
      "function": "Bypasses PB start/stop control when energized.",
      "power_domain": "24VDC",
      "normal_state": "de-energized",
      "location": {"x": 1180, "y": 690, "zone": "bottom-right"},
      "part_number": null, "manufacturer": null,
      "aliases": ["bypass relay"]
    },
    {
      "id": "PB1",
      "class": "push_button",
      "description": "Brown 22AWG start/stop switch #1 with start/stop cable.",
      "ratings": {},
      "function": "Operator start/stop control feeding relay CR1.",
      "power_domain": "control_signal",
      "normal_state": "open (not pressed)",
      "location": {"x": 180, "y": 250, "zone": "left"},
      "part_number": null, "manufacturer": null,
      "aliases": ["start/stop switch 1", "start stop button 1"]
    }
  ],

  "terminals": [
    {"id": "CB1:1",   "parent_component": "CB1",   "function": "line",    "net": "L1"},
    {"id": "CB1:2",   "parent_component": "CB1",   "function": "output",  "net": "L1-A"},
    {"id": "CR-BP:A1","parent_component": "CR-BP", "function": "coil",    "net": "NET-125"},
    {"id": "CR-BP:A2","parent_component": "CR-BP", "function": "coil",    "net": "NET-0V"},
    {"id": "PB1:2",   "parent_component": "PB1",   "function": "output",  "net": "NET-PB1-SP"}
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
     "from_terminal": "CB-BYPASS:2", "to_terminal": "CR-BP:A1", "cable": null, "net": "NET-125"}
  ],

  "cables": [
    {"id": "24E-1", "description": "Blue 18AWG control cable bundle",
     "member_wires": ["W042"]},
    {"id": "START-STOP-CABLE-1", "description": "PB1 start/stop cable (brown/white/blue 22AWG)",
     "member_wires": []}
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

### 4.4 Mapping to LightRAG custom-KG (`ainsert_custom_kg`)
Transform the master file into `{chunks, entities, relationships}`:
- **entities** ← every `components[]`, `terminals[]`, `nets[]`, `wires[]`, `cables[]`,
  `subsystems[]`, plus the `drawing`. Fields: `entity_name` (= `id`), `entity_type`
  (= `class`/type), `description`, `source_id`.
- **relationships** ← every `relationships[]` entry. Fields: `src_id`, `tgt_id`,
  `description`, `keywords` (from `type` + properties), `weight`, `source_id`.
- **chunks** ← one prose chunk per entity and per relationship (its `description`), each
  with a unique `source_id` referenced by the entity/relationship above.

---

## 5. Open Items / Next Steps
1. **Draft the extraction skill** in `schematic_skills/` targeting the §4 schema (netlist
   tracer + tiled vision verification). *Not started — awaiting go-ahead.*
2. Decide junction-vs-crossover handling (real junction vs. wires that cross but don't
   connect) — the main accuracy risk.
3. Handle off-page / bus nets (`MXCS-P9/P11` external device connections, ground/power
   rails).
4. Confirm how accurate the current `circuit_logic.json` is (how much tracing is already
   solved).
5. Build the master→custom-KG transform and the `ainsert_custom_kg` indexing script (adapt
   `_1_custom_index_01.py`).
6. Ingest the troubleshooting manual into the same working_dir with matching entity names.

### Accuracy risks to keep in mind
- **Junctions vs. crossovers** — wrong here = wrong netlist = confidently wrong answers.
- **Label→wire and label→terminal association** is a proximity heuristic; ambiguous on dense
  sheets.
- **Off-page / bus nets** fragment the graph if not handled explicitly.
- Vision verification mitigates but does not eliminate these — a QA pass belongs in the plan.
