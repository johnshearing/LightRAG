# How To Use This Skill

Operator's guide to the `schematic-extraction` skill: what it produces, what you do yourself,
what needs Claude, and how to run it on a new drawing without destroying previous work.

`SKILL.md` is written for Claude. This file is written for you.

---

## 1. What the skill is for

LightRAG indexes prose well and schematics badly. Handing a D-size electrical drawing to a
vision model gets you a confident, partial answer — it will find maybe a third of the
components and quietly invent the rest.

This skill splits the job by what each method is actually good at:

- **Geometry is deterministic.** Which conductor touches which terminal is in the PDF's
  vector layer. A script recovers it completely, every time.
- **Interpretation needs eyes.** What a symbol *means*, whether a crossing circle is a
  junction or a crossover hop, what a stroked label actually says — those need a vision
  pass, on small tiles, never on the whole sheet.
- **Completeness is a property of the script, not the model.** Nothing is silently dropped.

The result is injected into LightRAG with `ainsert_custom_kg`, which **bypasses LLM entity
extraction entirely**. A schematic is a deterministic netlist; it should not be re-guessed
by a language model. That is the whole reason this skill exists.

---

## 2. The pipeline and its artifacts

Run in this order. The Mod-Linx run (`work/mod_linx/schematic_extraction/`) is the worked
example throughout.

| # | Artifact | Produced by | Who does it | Edit it? |
|---|---|---|---|---|
| 1 | `geometry.json` | `scripts/extract.py` | you or Claude | **No** — regenerate |
| 2 | `tiles/` | `scripts/render_tiles.py` | you or Claude | **No** — regenerate |
| 3 | `crops/` (optional) | `scripts/render_tiles.py crops` | you or Claude | **No** — regenerate |
| 4 | `author_circuit_logic.py` | Claude, by hand | **Claude only** | **Yes — this is the one you maintain** |
| 5 | `circuit_logic.json` | running #4 `author_circuit_logic.py` | either | **No** — regenerate from `author_circuit_logic.py` |
| 6 | `custom_kg.json` | `schematic_skills/scripts/build_kg.py circuit_logic.json -o custom_kg.json` | you or Claude | **No** — regenerate |
| 7 | `EXTRACTION_NOTES.md` | Claude, by hand | Claude | Yes, it's documentation |
| 8 | (LightRAG index) | `scripts/index_schematic.py` | you or Claude | n/a |

**The file you index into LightRAG is `custom_kg.json`.** Not the PDF, not
`circuit_logic.json`.

**The file you correct when an answer is wrong is `author_circuit_logic.py`.** Then re-run
it, rebuild, re-index. Never hand-edit the JSON — it will drift out of sync with its own
derived edges.

### The correction loop

```bash
# 1. fix the misreading in the tables at the top of author_circuit_logic.py
python author_circuit_logic.py                       # regenerates circuit_logic.json
python <skill>/scripts/build_kg.py circuit_logic.json -o custom_kg.json --pretty --validate --report
python <skill>/scripts/index_schematic.py custom_kg.json -w <work_dir>
```

---

## 3. Tunable parameters — what and where

### Where they live

The defaults are the `DEFAULTS` dict at **`scripts/extract.py:57`**. **Do not edit that
dict.** Override per-drawing on the command line instead, so the skill stays reusable and
the values you actually used are recorded in the `params` block of the resulting
`geometry.json`:

```bash
python extract.py drawing.pdf --params '{"wire_min_len": 4, "label_attach_dy": 6}' -o geometry.json
```

The one you'll reach for most is not in that dict at all:

```bash
python extract.py drawing.pdf --layers SCHEMATIC -o geometry.json --pretty
```

`--layers` restricts conductor detection to a named PDF layer (OCG). Most CAD exports put
the border, title block and revision notes on separate layers; without this flag the title
block's ruled lines become phantom conductors. Run `--stats-only` first and read the
`layers` block to see what the drawing actually has. Mod-Linx has
`{0: 183, FORMAT: 157, SCHEMATIC: 6166, REVNOTE: 92}` — `SCHEMATIC` is obviously the one.

### The parameters worth knowing

| Parameter | Default | Change it when |
|---|---|---|
| `label_attach_dy` | 8.5 | **Most common tune.** Vertical search distance for the net label above / spec label below a conductor run. Must stay *below* the pitch between adjacent parallel runs, or a wire steals its neighbour's label. Symptom: `conductors_with_net_label` far below `conductors`, or labels attached to the wrong wire. Tighter drawing → lower it (6); airier drawing → raise it (11). |
| `wire_min_len` | 6.0 | A conductor segment must be at least this long. Lower it (3–4) on a B-size sheet or a dense drawing where short jumpers between adjacent terminals are being dropped. Raise it if glyph fragments are being counted as conductors. |
| `glyph_max_len` | 8.0 | Strokes shorter than this are treated as parts of letters, not wires. Raise it if the drawing uses a large text height and letter strokes are showing up as conductors; lower it if genuine short wires are vanishing. |
| `terminal_dot_max_dia` | 5.0 | Circles at or below this diameter are terminal points, larger ones are devices. Raise it if small relay coils are being classed as terminals; lower it if terminal dots are being reported as device circles. Check `symbols_terminal_points` vs `symbols_device_circles` in the stats. |
| `endpoint_bind_radius` | 7.0 | How close a conductor endpoint must be to a symbol or terminal-number label to bind to it. Raise slightly if wires aren't landing on terminals; raise too far and a wire binds to the wrong adjacent terminal. |
| `node_snap` | 0.35 | Endpoints closer than this are the same node. Raise to ~1.0 for a sloppy export where wires don't quite meet. Symptom: `nets` ≈ `conductors`, i.e. nothing is merging. Raise it too far and separate nets fuse — dangerous. |
| `text_max_height` | 12.0 | Clusters taller than this are graphics, not text. Raise for a drawing with a large title-block font. |
| `ortho_tol` | 0.25 | Deviation allowed when calling a segment horizontal or vertical. Raise for a hand-drawn or rotated-export drawing with slightly skewed runs. |
| `word_gap_ratio` | 1.45 | Horizontal gap (as a multiple of glyph height) at which two glyphs stop being one word. Lower it if `BLUE 18AWG` merges with the label next to it; raise it if single words are splitting. |
| `line_band` | 2.5 | Vertical tolerance for "same text line". Raise for a drawing with wobbly baselines. |
| `glyph_cell` | 1.2 | Union-find cell size when clustering strokes into glyphs. Rarely needs touching. |
| `ocr_dpi` / `ocr_pad` | 600 / 2.5 | Raster DPI and padding for OCR crops. Raise `ocr_dpi` to 800 for very small text. Irrelevant if the PDF has real embedded text or you pass `--no-ocr`. |

`render_tiles.py` has its own flags rather than a params dict: `--rows`, `--cols`
(default 4×4 — go to 5×5 or 6×6 for an E-size sheet), `--overlap` (default 25 pt), `--dpi`
(default 400), and `--annotate geometry.json`.

### How to know a parameter needs changing

Run this first, always:

```bash
python extract.py drawing.pdf --stats-only
```

Then read the stats block **before trusting anything downstream**:

| Symptom | Likely cause | Action |
|---|---|---|
| `has_embedded_text: false` | CAD stroke font — no real text in the PDF | Expected on plotted drawings. Every label came from OCR; plan on a thorough vision pass. |
| `labels_low_confidence` a large share of `text_labels` | Stroke font, small text | Normal. Work the contact sheets (`render_tiles.py crops`). |
| `conductors_with_net_label` ≪ `conductors` | Labels not attaching | Tune `label_attach_dy`. |
| `junctions` implausibly high | Crossovers read as junctions | Inspect visually. A false junction merges two nets and produces confidently wrong answers. |
| `nets` ≈ `conductors` | Nets not merging | Net labels unread, or `node_snap` too tight. |
| Conductor count wildly high | Border/title block included | Use `--layers`. |

Mod-Linx reference stats, for calibration:
`text_labels: 502, labels_read: 431, labels_low_confidence: 159, conductors: 149,
conductors_with_net_label: 70, junctions: 2, nets: 111, review_items: 278`.

Note `nets: 111` from geometry versus **26** in the finished `circuit_logic.json` — the
vision pass merged them. That gap is normal and is exactly the value the human step adds.

---

## 4. Can you do this yourself, or do you need Claude?

**Both.** The pipeline is deliberately split.

**You can run alone** — pure scripts, no model:

- Step 1 `extract.py` — geometry
- Step 2 `render_tiles.py` — tiles, zooms, crops
- Step 5 running `author_circuit_logic.py` (once it exists)
- Step 6 `build_kg.py` — validation and KG build
- Step 7 `index_schematic.py` — indexing
- Any correction to `author_circuit_logic.py` where you already know the right answer

**Claude must do** — needs vision and judgement:

- Viewing all 16+ tiles and deciding what each symbol is (coil vs lamp vs meter)
- Distinguishing a junction from a crossover hop arc
- Transcribing low-confidence labels
- **Writing `author_circuit_logic.py`** — the component/terminal/net/wire tables
- Synthesising `ON_NET` and `COIL_CONTROLS_CONTACT` edges
- Writing the prose descriptions that LightRAG actually embeds and matches against

So the realistic workflow for a new drawing is: **ask Claude Code**, and let it drive the
scripts. You take over for re-runs, corrections and re-indexing. There is no way to skip the
Claude step and get a usable graph — the vision pass *is* the skill.

### How a new session knows to write `author_circuit_logic.py`

Because **`SKILL.md` Step 5 now tells it to.** This is worth knowing about, because it was
briefly not true. The Mod-Linx session invented the generator-script pattern on the fly and
never wrote it back into the skill — Step 5 said only "write `circuit_logic.json`". A fresh
session reading that would have hand-written 200 KB of JSON directly, losing the derived-edge
guarantee and the provenance trail, and the correction loop in §2 would not have existed.

Step 5 was amended on 2026-07-27 to mandate the pattern, specify the five-part structure
(docstring → literal tables → derived edges → authored edges → write), and point at the
Mod-Linx script as the worked example. Steps 8 (correction loop) and the output-layout
section were added at the same time.

`_claude_notes/schematic_indexing_plan.md` does **not** need to be in the skill directory.
It mentions the filename once, in a past-tense record of completed work — it documents, it
doesn't instruct. The skill has to be self-contained; anything a future session must *do*
belongs in `SKILL.md` or `references/`.

**The general lesson:** when a session invents a technique that turns out to matter, it has
to be written back into `SKILL.md` or it is lost. If you notice a good practice from one run
missing from the skill, say so — that's a skill bug, not a documentation nicety.

### Installing the skill so Claude can find it

As of this writing the skill lives at `jrs/schematic_skills/` and is **not installed** as a
Claude Code skill — there is no `.claude/skills/` entry for it, so Claude will not
auto-discover it by description. Two options:

**A. Point at it explicitly** (works today, no setup):

> Read /home/js/LightRAG-Dev/jrs/schematic_skills/SKILL.md and follow it to extract
> the netlist from <pdf>.

**B. Install it properly** so it triggers on its own:

```bash
mkdir -p /home/js/LightRAG-Dev/.claude/skills
ln -s /home/js/LightRAG-Dev/jrs/schematic_skills \
      /home/js/LightRAG-Dev/.claude/skills/schematic-extraction
```

The directory name must match the `name:` field in `SKILL.md`'s frontmatter
(`schematic-extraction`). Use `/home/js/.claude/skills/` instead if you want it available
across all projects.

---

## 5. Output locations — nothing is hardcoded

**Every script writes wherever you tell it to.** All output paths are explicit `-o` /
`--output` / `-w` arguments. There is no built-in destination, and nothing forces output to
`work/mod_linx/schematic_extraction/` — that path was simply the choice made during the
Mod-Linx session.

The **one** exception: `author_circuit_logic.py` contains

```python
OUT = Path(__file__).parent / "circuit_logic.json"
```

which is relative to *its own location*. Copy it into a new directory and it writes there.
That is the behaviour you want, but it means a copied script writes next to itself, not next
to where you ran it from.

### Do you need to rename the existing directory?

**No — use a new directory instead.** Renaming works, but it is the fragile option: you'd
have to remember to do it every time, and forgetting silently overwrites a day's work.

Recommended convention, one directory per drawing:

```
jrs/work/<machine>/schematic_extraction/<DRAWING_NUMBER>/
    geometry.json
    tiles/
    crops/
    author_circuit_logic.py
    circuit_logic.json
    custom_kg.json
    EXTRACTION_NOTES.md
```

If you'd rather keep the current flat layout for Mod-Linx, at minimum rename it to
`schematic_extraction_PS20115MLM4-2/` before starting the next drawing, so the drawing
number is in the path.

**When you ask Claude to process a new drawing, state the output directory in your prompt.**
Left unspecified it may well reuse the Mod-Linx path, because that's the example it can see.

### Everything that would be clobbered

`geometry.json`, `tiles/*`, `crops/*`, `author_circuit_logic.py`, `circuit_logic.json`,
`custom_kg.json`, `EXTRACTION_NOTES.md` — all of them. `author_circuit_logic.py` is the
painful one: it's the only place the human readings live, and it cannot be regenerated by a
script. **Commit it to git before starting another drawing.**

---

## 6. Running a new drawing, start to finish

**This is a reference listing, not a shell script to run.** You can't run it end to end —
step 5 in the middle is the vision pass, and no shell can do it. It's here so you can see the
whole pipeline at once, and so you can run any individual step yourself when you need to.

**In practice you prompt Claude**, giving it three things: where the skill is, which PDF, and
where the output goes. Claude then drives the scripts itself.

> Read `/home/js/LightRAG-Dev/jrs/schematic_skills/SKILL.md` and follow it to extract the
> complete netlist from `<path/to/drawing.pdf>`. Put all output in `<output_dir>`. Review
> every tile visually and transcribe the low-confidence labels before writing
> author_circuit_logic.py. I want every net to list all of its member terminals.

(If you install the skill per §4, drop the first sentence — Claude will find it by
description.)

### Activate the virtual environment first

`ModuleNotFoundError: No module named 'lightrag'` means you're on the system Python. Only
`index_schematic.py` imports lightrag — `extract.py`, `render_tiles.py` and `build_kg.py`
don't — so steps 1–6 work outside the venv and step 7 suddenly doesn't. Activate it anyway:

```bash
source /home/js/LightRAG-Dev/.venv/bin/activate
```

The listing below is what Claude will do on your behalf:

```bash
SKILL=/home/js/LightRAG-Dev/jrs/schematic_skills
OUT=/home/js/LightRAG-Dev/jrs/work/<machine>/schematic_extraction/<DRAWING_NO>
mkdir -p "$OUT"

# 1. sanity check — read the stats and the layers block
python $SKILL/scripts/extract.py drawing.pdf --stats-only

# 2. full geometry, restricted to the schematic layer
python $SKILL/scripts/extract.py drawing.pdf --layers SCHEMATIC -o "$OUT/geometry.json" --pretty

# 3. tiles for the vision pass
python $SKILL/scripts/render_tiles.py tiles drawing.pdf -o "$OUT/tiles/" \
    --annotate "$OUT/geometry.json" --rows 4 --cols 4 --dpi 400

# 4. contact sheets of the low-confidence labels
python $SKILL/scripts/render_tiles.py crops drawing.pdf \
    --geometry "$OUT/geometry.json" -o "$OUT/crops/"

# --- 5. CLAUDE: view every tile and every crop sheet, then write
#        $OUT/author_circuit_logic.py and run it ---

python "$OUT/author_circuit_logic.py"

# 6. build and validate the KG
python $SKILL/scripts/build_kg.py "$OUT/circuit_logic.json" -o "$OUT/custom_kg.json" \
    --pretty --validate --report

# 7. index — dry run first
python $SKILL/scripts/index_schematic.py "$OUT/custom_kg.json" -w <work_dir> --dry-run
python $SKILL/scripts/index_schematic.py "$OUT/custom_kg.json" -w <work_dir>
```

---

## 6b. Ready-to-paste prompts

Start the session in `/home/js/LightRAG-Dev/jrs`.

### Full extraction of a new drawing

The general form. Substitute the PDF path and the output directory:

> Read `/home/js/LightRAG-Dev/jrs/schematic_skills/SKILL.md` and
> `references/HowToUseThisSkill.md`, then follow the skill to extract the complete netlist
> from `<path/to/drawing.pdf>`.
>
> Put all output in `/home/js/LightRAG-Dev/jrs/work/<machine>/schematic_extraction/<DRAWING_NO>/`.
>
> Run `--stats-only` first and show me the stats and layers block before going further. Then
> review every tile visually and transcribe the low-confidence labels before writing
> `author_circuit_logic.py`. Pay particular attention to junction-vs-crossover — on
> `PS20115MLM4-2` all 88 small circles were classified as terminal points and most were
> crossover hops. I want every net to list all of its member terminals.
>
> Stop after `build_kg.py --validate --report` and show me the report.
> I want to inspect the output files and do the indexing myself after the inspection.

Worked example, for the second Mod-Linx drawing:

> Read `/home/js/LightRAG-Dev/jrs/schematic_skills/SKILL.md` and
> `references/HowToUseThisSkill.md`, then follow the skill to extract the complete netlist
> from `/home/js/LightRAG-Dev/jrs/work/mod_linx/mod_linx_data/PS10115MLC2-2.pdf`.
>
> Put all output for this drawing in
> `/home/js/LightRAG-Dev/jrs/work/mod_linx/schematic_extraction/PS10115MLC2-2/`.
>
> Run `--stats-only` first and show me the stats and layers block before going further. Then
> review every tile visually and transcribe the low-confidence labels before writing
> `author_circuit_logic.py`. Pay particular attention to junction-vs-crossover: on
> `PS20115MLM4-2` all 88 small circles were classified as terminal points and most were
> crossover hops. I want every net to list all of its member terminals.
>
> Stop after `build_kg.py --validate --report` and show me the report. Don't index into LightRAG.
> I want to inspect the output files and do the indexing myself after the inspection.

Why each clause earns its place:

| Clause | What it prevents |
|---|---|
| Naming `SKILL.md` by path | Works whether or not the skill is installed as a symlink (§4). |
| Move the old output first | Protects `author_circuit_logic.py`, the one irreplaceable file. |
| `--stats-only` checkpoint | Catches a wrong `--layers` before an hour of tile reading is wasted. |
| The junction-vs-crossover note | Carries the hardest-won lesson from the first run into the new session. |
| "every net lists all its member terminals" | Forces the `ON_NET` edges; without them the graph is a bare wire list. |
| Stop before indexing | Keeps the working-directory decision (§7) yours, and it's hard to undo. |

If the drawing turns out to be another sheet of the *same* panel, replace the last paragraph
with: *"Then index into `work/mod_linx/mod_linx_work_dir` and query in hybrid mode to check
the netlist survived."*

### Sanity check only, before committing to a full run

> Run `schematic_skills/scripts/extract.py --stats-only` on `<pdf>` and tell me whether this
> drawing is a good candidate for the skill — check for a vector layer, which OCG holds the
> schematic, and whether any parameters need tuning off the defaults.

### Correcting a wrong answer

> Querying `<work_dir>` in hybrid mode gave me `<wrong answer>`; the drawing actually shows
> `<what you know is true>`. Go back to the tiles for
> `work/<machine>/schematic_extraction/<DRAWING_NO>/`, confirm what's on the sheet, then fix
> the reading in `author_circuit_logic.py`, re-run it, rebuild the KG and re-index. Record
> the correction in `EXTRACTION_NOTES.md`.

### Indexing an already-extracted drawing

> Index `work/<machine>/schematic_extraction/<DRAWING_NO>/custom_kg.json` into
> `<work_dir>` using `schematic_skills/scripts/index_schematic.py`. Dry-run first. Then query
> in hybrid mode to confirm the netlist survived — ask what net 110 connects to and check it
> lists every member terminal.

---

## 7. LightRAG indexing notes

- **Index `custom_kg.json`**, via `index_schematic.py`, which uses `ainsert_custom_kg`. The
  graph goes in exactly as built; LightRAG runs no entity extraction over it.
- **Embedding model must match the working directory.** `text-embedding-3-large`, dim 3072
  for this project. Overridable via `EMBEDDING_MODEL` / `EMBEDDING_DIM` env vars, but
  changing it means clearing vector storage and re-indexing *everything* in that directory.
- **Index the machine's troubleshooting manual into the SAME working_dir**, through the
  ordinary `ainsert` path. The two graphs merge on shared entity names — which is the entire
  reason the schematic must use the drawing's exact designators (`CR-BP`) as entity names,
  with plain-language names in `aliases`.
- **Query in `hybrid` mode.**
- `--dry-run` validates without writing. Always run it first.
- `--doc-id` defaults to the KG filename. Give distinct drawings distinct doc-ids if you're
  putting several into one working directory.

### `index_schematic.py` vs `jrs/_1_custom_index_01.py`

Either works. Both load the JSON and call `rag.ainsert_custom_kg(kg, full_doc_id=...)`;
neither has a special claim on the format. Their embedding paths differ in code
(`_1_custom_index_01.py` uses llama-index's `OpenAIEmbedding`, the skill script uses
lightrag's `openai_embed`) but resolve to the same `text-embedding-3-large` / 3072, so the
vectors are compatible and you can point both at the same working directory.

| | `index_schematic.py` | `_1_custom_index_01.py` |
|---|---|---|
| Configuration | CLI args — no file edit per drawing | `WORKING_DIR` and `files_2b_indexed` hardcoded |
| `--dry-run` | yes | no |
| Pre-flight validation | required keys, and **every relationship has `keywords`** | none |
| Logging | stdout | rotating file → `lightrag_index.log` |
| Skip already-indexed | no | yes, via `kv_store_doc_status.json` |

The pre-flight `keywords` check is the one that earns its keep: `ainsert_custom_kg` reads
that key without a default and raises if it's absent, so without the check you find out
partway through a paid indexing run.

Treating them as complementary works well — free preflight, then index with whichever you
prefer:

```bash
source /home/js/LightRAG-Dev/.venv/bin/activate
python schematic_skills/scripts/index_schematic.py <OUT>/custom_kg.json -w <work_dir> --dry-run
python _1_custom_index_01.py     # or drop --dry-run from the line above
```

Note `-w` is a required flag, not positional. And one caveat on `_1_custom_index_01.py`'s
skip logic: it matches `doc["file_path"]` against the full path in `files_2b_indexed`, but
`ainsert_custom_kg` populates `file_path` from the entries inside the KG (which say
`circuit_logic.json`). Unverified — but if it re-indexes something you expected it to skip,
that's where to look.

### One working_dir per machine — decide before you index a second drawing

Merge-on-name is the skill's central mechanism, and it cuts both ways:

- **Between a schematic and its manual — you want it.** The manual's prose about `CR-BP`
  lands on the same node as the schematic's netlist facts about `CR-BP`. That's the payoff.
- **Between two sheets of the same panel — you want it.** Net `110` continuing onto sheet 2
  should be one net, not two.
- **Between two different panels — it silently corrupts the graph.** Every schematic has a
  `CB1`, a `PS1`, a `CR-BP`. Index two different units into one working directory and those
  become single nodes carrying contradictory facts from both machines. Nothing errors; the
  answers just quietly blend two drawings.

So before indexing a second drawing, establish whether it is *another sheet of the same
panel* or *a different panel*. The Mod-Linx folder holds `PS20115MLM4-2` (20 A, master, 4
drive cards) and `PS10115MLC2-2` — from the numbering, a sibling unit rather than a second
sheet. If that's right, they need separate working directories.

Three ways to handle a different panel, best first:

1. **Separate working directory per panel** — `mod_linx_work_dir_PS10115MLC2-2`. Clean, and
   each panel's manual can go in alongside it.
2. **Qualify the entity names** with the drawing number (`PS10115MLC2-2:CB1`) in
   `author_circuit_logic.py`, with the bare designator in `aliases`. Keeps one graph, but
   weakens manual merging, since the manual says "CB1".
3. **Accept the merge** — only defensible if the two panels really are electrically
   identical, and even then the wire colours and terminal counts usually differ.

---

## 8. Accuracy risks, ranked by damage

1. **A junction that is really a crossover.** Merges two nets; every downstream answer about
   either one is then confidently wrong. On the Mod-Linx sheet, `extract.py` classified all
   **88** small circles as terminal points when most are semicircular hop arcs meaning *no
   connection*. Genuine terminal points appear only inside terminal-block rectangles. This
   is the single most important thing the vision pass has to catch, and it is drawing-
   specific — check it on every new sheet.
2. **A misread net designator** (`10` for `110`) silently rewires the graph. Numeric labels
   carry net identity and deserve the closest attention.
3. **Label-to-conductor mis-association.** Parallel runs sit ~10 pt apart, so the spec below
   one run and the designator above the next occupy nearly the same band. Tune
   `label_attach_dy`.
4. **Missing `ON_NET` edges** turn a troubleshooting graph back into a bare wire list. A
   fault propagates across a whole net, not just one wire's two ends — so every terminal on
   a net needs an `ON_NET` edge. Without them, "what is wire 110 connected to" returns two
   terminals instead of five. `build_kg.py --report` warns if they're missing entirely.
5. **Fragmented off-page nets** produce invented destinations instead of honest boundaries.
   Off-page designators need a boundary entity and a `REFERENCES` edge, so the answer is
   "this continues on MXCS-P9" rather than a confident fabrication.

---

## 9. Limitations — when this skill won't work

- **Scanned schematics.** No vector layer means no geometry to trace. This skill does not
  handle them; that needs raster line-following (OpenCV) or a trained symbol detector first.
  Check with `--stats-only`: near-zero `conductor_segments` on a drawing that obviously has
  wires means it's a scan.
- **Symbol identity is always a vision decision.** The scripts report a circle's size and
  position, never what it is. Coil vs lamp vs meter cannot be scripted.
- **N.O. vs N.C. contacts** differ by a single short bar. Read the terminal numbers as the
  cross-check: `11/14` = N.O., `11/12` = N.C.
- **Multi-sheet drawings** extract page by page. Joining them across page boundaries via
  off-page designators is manual.
- **Domain facts not on the drawing.** A breaker wired as a manual switch looks exactly like
  a breaker used for protection. On the Mod-Linx sheet the REVERSE and BYPASS 5A breakers
  are switches — only CB1 and CB2 are true over-current devices. Nothing in the PDF says so;
  it took your domain knowledge. Expect one or two of these per drawing and expect to have
  to supply them yourself.
- **OCR is optional infrastructure.** `extract.py` needs `pymupdf`; label OCR additionally
  needs the `tesseract` binary and `pillow`. Without them you still get full geometry, with
  every label marked unread.

---

## 10. Practical habits

- **Run `--stats-only` first, every time.** Thirty seconds that tells you whether the rest of
  the run is worth doing.
- **Use `--annotate`** when rendering tiles. The Mod-Linx tiles were rendered clean
  (`"annotated": false` in `tiles.json`), which made them easier to read but left no visual
  record tying a conductor ID in `geometry.json` to a run on the drawing. That's an audit gap
  — you can't re-verify a specific ID later without re-rendering.
- **`tiles.json` is the map back to the drawing.** It records each tile's PDF rectangle in
  points, so "the error is in `tile_r3c2`" converts directly into a high-magnification
  re-render:
  ```bash
  python $SKILL/scripts/render_tiles.py region drawing.pdf --rect 700,430,1000,530 \
      -o zoom.png --annotate geometry.json --dpi 500
  ```
- **Write an `EXTRACTION_NOTES.md` for every drawing.** Corrections to earlier assumptions,
  flagged inferences, open uncertainties. Six months from now it's the only thing that tells
  you which parts of the graph to distrust. The Mod-Linx one is the template.
- **Keep a ground-truth Q&A list** — a dozen questions you already know the answers to — and
  check the finished graph against it. The Mod-Linx notes verify against exactly that and it
  caught real errors.
- **Commit `author_circuit_logic.py` and `EXTRACTION_NOTES.md` to git.** They encode human
  work that no script can reproduce. The other artifacts are all regenerable.
- **Prefer prose over designators in descriptions.** Terse designators embed poorly; the
  prose description is what a query actually matches against.

---

## 11. Quick reference

```bash
# extract.py
python extract.py drawing.pdf --stats-only
python extract.py drawing.pdf --layers SCHEMATIC -o geo.json --pretty
python extract.py drawing.pdf --no-ocr -o geo.json
python extract.py drawing.pdf --params '{"label_attach_dy": 6}' -o geo.json
python extract.py drawing.pdf --page 2 -o geo_p2.json

# render_tiles.py
python render_tiles.py tiles drawing.pdf -o tiles/ --annotate geo.json --rows 4 --cols 4 --dpi 400
python render_tiles.py region drawing.pdf --rect 700,430,1000,530 -o zoom.png --dpi 500
python render_tiles.py crops drawing.pdf --geometry geo.json -o crops/
python render_tiles.py crops drawing.pdf --geometry geo.json --ids T0316,T0327 -o crops/

# build_kg.py
python build_kg.py circuit_logic.json -o custom_kg.json --pretty --validate --report

# index_schematic.py
python index_schematic.py custom_kg.json -w /path/to/work_dir --dry-run
python index_schematic.py custom_kg.json -w /path/to/work_dir --doc-id PS20115MLM4-2
```

Key file locations:

- Skill: `/home/js/LightRAG-Dev/jrs/schematic_skills/`
- Parameter defaults: `scripts/extract.py:57` (`DEFAULTS`) — override with `--params`, don't edit
- Worked example: `/home/js/LightRAG-Dev/jrs/work/mod_linx/schematic_extraction/`
- Field spec for `circuit_logic.json`: `references/circuit_logic_schema.md`
- Symbol and convention notes: `references/schematic_conventions.md`
</content>
</invoke>
