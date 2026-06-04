# FunnelStory PPTX Deck Builder

## Overview

Producing a repeatable PPTX deck is a two-step process. Always do both steps in order.

**Step 1 — Design and iterate (Claude chat, no FunnelStory).**  
Use the [Anthropic PPTX skill](https://github.com/anthropics/skills/blob/main/skills/pptx/SKILL.md) with Python tools (`pptxgenjs`, `markitdown`, `soffice`, `pdftoppm`) to build, render, and visually verify the deck directly in the chat. No flows, no FunnelStory. The output is a validated slide structure the user has approved.

**Step 2 — Encode as a FunnelStory flow (repeatable, per-account).**  
Once the user confirms the output from Step 1, translate the confirmed design into a deterministic `shape_data → build_slides → pptx.create` flow that can run on demand for any account. The XMLs and layout logic from Step 1 feed directly into `build_slides`.

When the user says the Step 1 output looks right, ask whether they want to create the flow. If yes, move to Step 2. Use the slide XML and coordinate logic you already validated in Step 1 — don't redesign from scratch.

---

## Step 1 — Iterate in Claude Chat

Follow the [Anthropic PPTX skill](https://github.com/anthropics/skills/blob/main/skills/pptx/SKILL.md) for the full procedure. Key tools and their roles:

| Tool | Role |
|---|---|
| `pptxgenjs` (via Python) | Generate `.pptx` from slide definitions |
| `markitdown` | Parse an existing reference deck to extract text/structure |
| `soffice --headless --convert-to pdf` | Convert `.pptx` to PDF for rendering |
| `pdftoppm -png -r 100` | Render PDF pages to PNG for visual inspection |
| Subagent visual QA | Inspect rendered PNGs and flag layout issues |

### What to produce before Step 2

Before asking the user to confirm:

1. At least one rendered PNG the user has visually approved.
2. The raw `<p:sld>` XML for each slide (either as the `slides` array, or as a `files` map of path → XML if you need full zip control).
3. The complete list of shapes per slide with their bounding boxes (`x`, `y`, `cx`, `cy` in EMU) — you'll need these to wire up the `build_slides` step.

Once the user confirms, proceed to Step 2.

---

## Step 2 — FunnelStory Flow

### Prerequisites

- **FunnelStory MCP** connected for the target workspace.
- `read_resource("file://flow/guide.md")` — flow argument shapes and step wiring.
- `read_resource("file://semantic/schema.sql")` and `read_resource("file://semantic/usage.md")` — write correct semantic queries.
- `query_semantic_db` — validate semantic SQL before embedding in a flow.
- `configure_flow` / `run_flow` — save and test the flow.

---

### Phase 0 — Read before doing anything

```
read_resource("file://flow/guide.md")
read_resource("file://semantic/schema.sql")
read_resource("file://semantic/usage.md")
```

Then **inspect the actual records** for the data you'll be working with. Don't infer field meanings from field names. For any source/dataset table you'll be touching, fetch 2–3 sample rows and look at the actual JSON. Field semantics (which field holds the title, which is the parent's name, what status values appear) are not portable across sources, customers, or even across record types within one source.

---

### Phase 1 — Requirements gathering

Ask **exactly once** (batch all questions together):

1. **Target identifier** — which account/entity/record to use for the first test run, and what input field the flow takes (code, name, UUID, etc.).
2. **Slide content** — what should each slide show? Walk through a list. The user's domain determines the specifics; common slide *roles* include a cover/title slide, an executive summary or KPI snapshot, a recent-activity or "what changed" slide, a work-in-progress or status-by-item slide, an exceptions/issues slide, and a look-ahead slide. Ask what each slide is *for*, not just what data it shows.
3. **Live data or fixed deck?** — Does the deck need to reflect live data each time it runs (fetch from semantic DB or connections on every trigger), or is the content fixed (same slides every time, with only minor personalization like account name)? This determines whether fetch steps are needed at all.
4. **Data sources** — semantic DB only, or external connections? Confirm dataset/table names. (Only needed if live data.)
5. **Visual style** — dark/light cover, brand colors (hex), reference deck.

Defaults are fine for (3)/(4) if the user is unsure. If the user already went through Step 1, most of this is already answered — confirm only what's still open.

---

### Phase 2 — Validate queries before building the flow

For every data requirement, run the query directly with `query_semantic_db` (or `preview_data_connection` for external connections) and confirm rows come back. **Inspect the actual JSON of one or two records.**

Three traps to watch for in any source data:

- A field named `name` might mean something other than the record's own name (in many systems it inherits the parent's name).
- A "title" field might be empty for some record types and populated for others — different types of records in the same table can have different conventions.
- Status fields may have a canonical machine value (e.g. `status_type`) and a display value (e.g. `status`) that drift apart. Bucket on the canonical one if present.

When in doubt: write a small SQL query that surfaces all distinct values of the field you care about, before you write the rendering logic.

---

### Phase 3 — Flow architecture (two-step, no LLM)

**Strongly prefer this graph** over a single big script or any AGENT/LLM step:

```
[fetch step(s)]   semantic.query / data_connection.query     →  named vars   (omit if fixed deck)
shape_data        script.js, deterministic JS                →  @.shaped
build_slides      script.js, slide adjustments               →  @.slide_files
build_title       script.js, filename builder                →  @.deck_title
create_deck       pptx.create                                →  @.deck
set_response      script.js, response message (optional)     →  @.response
```

`set_response` is optional and its content depends on what the flow does. If the flow emails the deck, the response might confirm delivery ("Sent your Q3 deck to pulkit@example.com"). If it does something else, build accordingly. Omit the step entirely if the flow doesn't need a structured response.

#### Why two JS steps (shape + render), not one and not an agent

- **Determinism.** Same input → identical output. Cheaper, faster, debuggable. No model hallucinating fields or silently dropping rows.
- **Separation of concerns.** `shape_data` knows source-data semantics (field aliases, status buckets, sorting, severity labels, date parsing). `build_slides` knows only the shaped schema and pixel coordinates. Either can be replaced independently.
- **Smaller individual scripts** — each fits comfortably under the 64 KiB script.js limit.
- **Easier iteration.** Visual feedback only touches `build_slides`. Data-meaning feedback only touches `shape_data`.

Use an LLM (`AGENT`) step **only** when you genuinely need natural-language summarization or judgment over unstructured text — never for bucketing/sorting/formatting, which is deterministic JS.

#### Updating individual steps

After the flow exists, use `configure_flow` with `flow_id` + `step_id` + `step` to update one step at a time. Passing a full `config` **replaces the entire graph** and drops any step not in the new payload.

---

### Phase 4 — `shape_data` step (the data step)

The shape step turns raw query results into a clean JSON object the renderer can consume blindly. The renderer should never have to look up parent fields, parse dates, or apply business rules — those all live here.

#### Contract

- **Input**: whatever you fetched (named vars from the query steps), plus `ctx.now` for the current timestamp if date display is needed.
- **Output**: `result = JSON.stringify(shaped)` — a single JSON string with a stable schema you control.
- Synchronous JS, no `await`, no network calls. 5 s wall clock, 64 KiB source limit.

#### Parse query results correctly

`semantic.query` and many `data_connection.query` results come back as `{results: [{record: "<json string>"}, ...]}`. The `record` value is often a **stringified JSON**, not an object. Always normalize:

```javascript
function parseResults(qr) {
  if (!qr || !qr.results) return [];
  var out = [];
  for (var i = 0; i < qr.results.length; i++) {
    var rec = qr.results[i].record;
    if (typeof rec === 'string') {
      try { out.push(JSON.parse(rec)); } catch (e) {}
    } else if (rec) {
      out.push(rec);
    }
  }
  return out;
}
```

#### Pick a canonical "best title" helper, per record type

Different record types in the same table often use different fields for their human-facing title. Build a small helper that knows your data:

```javascript
function bestTitle(x) {
  // Edit this per source: which field is THIS record type's real title?
  if (x.record_type === 'A') return x.name || '';
  if (x.record_type === 'B') return x.what || x.name || '';
  return x.title || x.name || '';
}
```

The renderer should call `bestTitle(x)` once and trust the result.

#### Filter out records that aren't your target

If a single source table mixes parent records with child records (a "container" with its "items"), guard against the container showing up in your item list:

```javascript
for (var i = 0; i < allItems.length; i++) {
  var it = allItems[i];
  if (it.record_type === 'container') continue;  // skip the parent
  // bucket as item …
}
```

Without a guard, the parent often appears as the first row of your item table.

#### Bucket on canonical status, fall back to display status

Many systems have both `status_type` (canonical: `open`, `closed`, `done`, `in progress`) and `status` (the user-facing label, often customized per workspace). Bucket on the canonical one first:

```javascript
var st = (m.status_type || '').toLowerCase();
var ms = (m.status || '').toLowerCase();
if (st === 'done' || st === 'closed' || ms === 'completed') doneList.push(m);
else if (/* past-due check */) overdueList.push(m);
else if (ms === 'in progress' || st === 'in progress') wipList.push(m);
else upcomingList.push(m);
```

For numeric fields like severity, **verify direction by sampling** (lower = worse, or higher = worse? both conventions exist).

#### Don't duplicate a "spotlight" item in a list next to it

A common slide pattern is: big callout of the most important item, plus a smaller list of "what's next". If the renderer pulls the spotlight from index 0 and the list also starts at index 0, the first list item duplicates the spotlight. Shape the data so the renderer can't make that mistake:

```javascript
shaped.spotlight = sortedItems[0] ? formatSpotlight(sortedItems[0]) : null;
shaped.list      = sortedItems.slice(1, 5).map(formatListItem);
```

The renderer just renders both fields. The shape step owns the no-duplication invariant.

#### Provide pre-ordered arrays for table-style slides

If a slide is going to show "all items ordered by priority/status", build that single ordered array here, not in the renderer:

```javascript
shaped.all_items = [].concat(
  overdueList.map(toRow),
  wipList.map(toRow),
  upcomingList.map(toRow),
  doneList.map(toRow)
);
shaped.total_items = overdueList.length + wipList.length + upcomingList.length + doneList.length;
```

The renderer iterates blindly. If the user wants a different order, change the shape step in one place.

#### Derive everything the renderer might need to caption

If the renderer needs to print "X% complete · Y of Z done · N overdue", build that string here once and put it in the shaped data. Don't make the renderer do arithmetic or string concatenation over the data — that splits business logic across two steps.

#### Customer/entity name derivation: use targeted prefix logic, not greedy splits

Source records often store names with structured prefixes (codes, type tags). To extract a clean display name:

```javascript
// Specific, anchored, predictable:
var match = rawName.match(/^PREFIX[\s\-:]+(.+)$/i);
var display = match ? match[1].trim() : rawName;
```

Avoid `rawName.split('-')[0]` style. For a name like "Type-Customer", split-on-`-` yields "Type", not "Customer". Anchor on the known prefix shape.

#### Compute status deterministically

```javascript
var status = 'On Track';
if (severeIssueCount >= 2) status = 'High Risk';
else if (severeIssueCount >= 1) status = 'At Risk';
if (progress >= 100) status = 'Completed';
```

Then build a rationale string with the actual numbers, so the user can audit the verdict:

```javascript
shaped.status_rationale = progress + '% complete · ' + done + ' of ' + total + ' done · ' + overdue + ' overdue';
```

This is more useful than any LLM-generated paragraph and is reproducible.

---

### Phase 5 — `build_slides` step (the rendering step)

#### The core principle: adjust, don't rebuild

The `build_slides` script's job is to **adjust** the validated slide structure from Step 1 — adding or removing slides based on data, substituting text values, and adapting layout for variable row counts. It does **not** rebuild OOXML from scratch.

OOXML is complex: namespaces, relationship IDs, master/layout references, theme wiring — getting all of this right by hand in a JS string template is fragile and hard to debug. Step 1 produces correctly-formed XML using real tools (`pptxgenjs`, or raw unzipping of a reference deck). Bake that validated structure into the flow as the base, then let the script only touch what varies per-account.

Concretely:
- Slide shape XML, bounding boxes, and style attributes come from Step 1 — embed them as JS string constants.
- The script substitutes data (account name, KPI values, row lists) into those constants.
- Slide-level conditionals add or remove whole slides based on what data exists.
- Layout math (row heights, overflow caps) adjusts based on row counts — but the coordinate anchors come from Step 1, not invented fresh.

#### Output format: always `files`

`build_slides` always outputs a `files` map — an object of zip entry path → XML string. Pass it directly as the `files` arg to `pptx.create`. Never use the `slides` array.

```javascript
result = JSON.stringify({
  "ppt/slides/slide1.xml": slide1XML,
  "ppt/slides/slide2.xml": slide2XML,
  // add or omit slides based on data
});
```

Any zip entry not present in `files` falls back to the generated default. The defaults include the slide master, layout, relationships, and content types. You only need to include entries that have data-driven content or differ from the 16:9 widescreen boilerplate.

**Default slide canvas**: 16:9 widescreen — 12 192 000 × 6 858 000 EMU. This is the default for any run that doesn't override `ppt/presentation.xml`. If the reference deck uses a different size, add it explicitly:

```javascript
files["ppt/presentation.xml"] = '<p:presentation xmlns:p="..." ...>'
  + '<p:sldSz cx="9144000" cy="6858000"/>'  // 4:3 example
  + '...</p:presentation>';
```

Common canvas sizes (EMU): 16:9 = 12 192 000 × 6 858 000 · 4:3 = 9 144 000 × 6 858 000 · 16:10 = 11 430 000 × 7 144 875.

#### Contract

- **Input**: `shaped` (the JSON string from `shape_data`), plus `ctx.now` if date display is needed.
- **Output**: `result = JSON.stringify({ "ppt/slides/slideN.xml": xml, ... })` — a files object.
- First line parses the shaped data safely:

```javascript
var raw = input.shaped;
var d = typeof raw === 'string' ? JSON.parse(raw) : raw;
```

**Do NOT add a markdown-fence-stripping regex** (`/^```/` etc.) "just in case". The shape step always emits raw JSON via `JSON.stringify`, so fences never appear. And regexes containing `\n` get mangled when the source is JSON-encoded into the flow config (see "Encoding gotchas" below).

#### Helper functions: keep them tiny

A small vocabulary of helpers makes every slide compose cheaply:

- `sp(id, name, x, y, cx, cy, fill, text, sz, bold, color, italic, anchor)` — text shape.
- `bx(id, x, y, cx, cy, fill)` — filled rectangle (no text).
- `ln(id, x, y, cx, cy, color)` — thin colored line (a rectangle with very small `cy`).
- `sld(bg, shapes)` — slide wrapper with background fill + namespace declarations.
- `esc(s)` — XML escape.

Slide-level helpers like `hdr(...)` and `ftr(...)` keep top crumbs and footers consistent across slides.

#### XML escaping — escape **once**, in the helper, never in source strings

```javascript
function esc(s) {
  return String(s == null ? '' : s)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}
```

When building shape text, pass **raw special characters** to the helper:

```javascript
// CORRECT — raw ampersand, esc() handles it once:
sp(5, 'pt', ..., 'Quarterly Review & Outlook', ...);

// WRONG — already-escaped string gets escaped again, producing &amp;amp;
sp(5, 'pt', ..., 'Quarterly Review &amp; Outlook', ...);
```

The double-escape symptom is literal `&amp;` text in the rendered slide, instead of `&`.

#### Coordinate system

| | Value |
|---|---|
| Slide canvas (16:9 widescreen, default) | 12 192 000 × 6 858 000 EMUs |
| 1 inch | 914 400 EMUs |
| 1 pt | 12 700 EMUs |
| Font size attribute | hundredths of a point (`sz="1200"` = 12 pt) |

Standard reference points for 16:9: left/right margins ≈ 457 200 (0.5"), top crumb at y ≈ 150 000, slide title around y ≈ 550 000, footer at y ≈ 6 630 000.

**Custom dimensions**: pass `files["ppt/presentation.xml"]` with a `<p:sldSz cx="..." cy="..."/>` set to your target size. All coordinate math in the `build_slides` step must use the same canvas dimensions. Common sizes (EMU): 4:3 = 9 144 000 × 6 858 000; 16:10 = 11 430 000 × 7 144 875.

#### **PPTX has no automatic inter-shape layout. This is the single biggest rule.**

Read it twice. Internalize it.

- Every `<p:sp>` is positioned absolutely. If two shapes' bounding boxes overlap, they render on top of each other in z-order. The **next shape does not get pushed down** when the previous shape's text wraps.
- `normAutofit` and `spAutoFit` only handle **intra-shape** overflow (shrinking text or growing one shape to fit its own text). They cannot resolve **inter-shape** collisions. They are a safety net for individual shapes, not a layout system.
- The one PPTX construct that does real reflow-style layout is `<a:tbl>` (tables): row heights grow to fit cell content, pushing later rows down. **For variable-length tabular content, use a real table.** Stacking individual `<p:sp>` rectangles to look like a table is fragile.

Everywhere else, you do the math: compute positions from content counts, lengths, and presence/absence of optional sections.

#### The recurring overlap patterns (write these into muscle memory)

#### Pattern 1: Title wraps → element below at fixed Y collides

Two separate shapes, one above the other. The top shape has variable-length text that wraps to 1 or 2 lines. The bottom shape sits at `top_y + fixed_offset`. When the title wraps, the bottom shape collides with line 2.

**Fix:** make them a single shape. Concatenate title + detail/date with a separator, set the shape `cy` tall enough for 2 lines at the chosen font size, and let the wrap happen *inside* one shape.

```javascript
// Two-shape version (fragile):
sp(12, 'title', x, y,         w, 320000, null, item.title,  1700, true, ...);
sp(13, 'meta',  x, y+560000,  w, 270000, null, item.meta,   1100, false, ...);

// One-shape version (robust):
sp(12, 'titleMeta', x, y, w, 540000, null, item.title + ' — ' + item.meta, 1300, true, ...);
```

This converts an inter-shape layout problem into an intra-shape font/sizing problem, which is bounded and predictable.

#### Pattern 2: Fixed-row table → variable-width cell text wraps into the next row

Rows at fixed Y intervals; one row's text wraps to 2 lines and visually merges with the next row.

**Fix options, in order of preference:**

1. Use a real `<a:tbl>` (rows grow automatically).
2. Truncate in the shape step (`text.substring(0, N).trim() + '…'`). Bounds the input.
3. Compute row height from `rowsShown` and cap min/max:

   ```javascript
   var rowH = Math.floor(availableHeight / rowsShown);
   if (rowH < 280000) rowH = 280000;
   if (rowH > 460000) rowH = 460000;
   ```

   Then make each row's `cy` slightly less than `rowH` so dividers don't ride into the next row.

#### Pattern 3: Overflow indicator collides with footer

A list shows N rows; if there's more data, a "+ M more …" line is appended. The footer sits at a fixed Y at the bottom of the slide. Without care, the "+ M more" overlaps the footer.

**Fix:** cap the list length so the tail indicator + bottom margin clears the footer. The cap depends on row height; if footer is at y ≈ 6 630 000 and you want 60 000 EMU of breathing room, the last row + indicator must end by y ≈ 6 570 000. Either:

- Hard-cap `maxRows` (e.g. 8 or 9 instead of 10 at standard row heights), or
- Position the indicator dynamically from the actual last-row end-Y, and assert it doesn't exceed the footer threshold.

#### Pattern 4: "Spotlight" duplicated in the list rendered next to it

A slide pulls "the most important item" out for a big callout (the spotlight), and also renders a smaller list of upcoming/recent items in the same panel. If both are built from the same array starting at index 0, the spotlight item is also the first list item.

**Fix:** in the shape step, build `spotlight = items[0]` and `list = items.slice(1, N)`. Never let the renderer pick the spotlight — that's a data-shaping decision.

#### Pattern 5: Bottom panel of a slide overlaps the footer

A colored panel anchored at the bottom of the content area extends down further than expected (because its content is taller), and visually crashes into the footer text.

**Fix:** the panel's `y + cy` must be ≤ `footer_y − small_gap`. Compute the panel height from its content, not from a fixed value, and assert the bound.

#### `normAutofit` as a safety net

Add it to individual shapes that hold variable-length text — titles, status pills, card values, single-row labels. It won't fix overlaps, but it prevents text from spilling outside the shape's own box:

```xml
<a:bodyPr wrap="square" anchor="ctr">
  <a:normAutofit fontScale="100000" lnSpcReduction="0"/>
</a:bodyPr>
```

PowerPoint will reduce `fontScale` automatically if the text doesn't fit. Costs ~30 chars per shape, no downside.

#### When you have variable tabular data, just use `<a:tbl>`

If the slide is *fundamentally* a table (rows of items with the same columns, content length unknown), build a real `<a:graphicFrame>` containing `<a:tbl>` instead of stacking `<p:sp>` shapes. The XML is more verbose and custom per-cell styling (colored pills, multi-color text within a cell) is harder, but row growth is automatic — so you trade layout effort for styling effort. For dashboards with predictable-length content, custom shapes win; for raw data tables with arbitrary content, real tables win.

#### Slide-level conditional rendering

Every slide should be wrapped in a guard:

```javascript
if (data.exists_for_this_slide) {
  var s = [];
  hdr(s, ...); // crumb + status badge
  // ... build shapes
  ftr(s, ...);
  slides.push(sld('FFFFFF', s));
}
```

Don't ship empty slides with "No data" placeholders unless the user explicitly wants them. Better to skip the slide and produce a 3-slide deck than a 5-slide deck with 2 empty ones.

---

### Phase 6 — Test and visual verification

#### Always run in test mode

**Every `run_flow` call must include `options.test.enabled = true`.** No exceptions during development or iteration. Test mode runs the flow end-to-end but skips persistent write side effects — no emails are sent, no external systems are touched.

```javascript
run_flow({
  flow_id,
  input: { /* exact keys from input_schema */ },
  options: { test: { enabled: true } }
})
```

Production (non-test) runs happen on the user's schedule/trigger or when the user explicitly asks to send a real email. The skill's job during build and iteration is to verify the flow works, not to trigger real side effects.

#### Input defaults are a footgun

If a field's `value` in the input schema is set to a real value, runs with no input (or with a wrong key name) silently use the default. To turn off the default, set `value: ""`. Always pass the input with the exact key from the schema; mismatched key names produce silent default substitution, not an error.

#### What you can verify in test mode (without ever opening a file)

- **The flow doesn't error.** Inspect the event trace in the response — every step should reach `Step executed`. Failures show up here with the offending step's error message.
- **Data shape is correct.** The event trace contains the intermediate variables after each step. Read the `shape_data` output and confirm the schema you intended is what you produced.
- **Slide XML is well-formed.** The `build_slides` step output in the event trace contains the slide XMLs before they are zipped. Read them, search for specific strings, count shapes per slide, and confirm escaping looks right (raw `&amp;`, not `&amp;amp;`).
- **Coordinate math.** Read the `<a:off>` and `<a:ext>` values from the `build_slides` output and assert that things like footer-Y > last-content-Y + gap. This catches the recurring overlap patterns programmatically before any human looks at a render.

This covers ~80% of bugs the skill is likely to introduce. The remaining ~20% — text wrapping behavior, actual rendered font sizes, visual whitespace judgment — needs a human eye.

#### Visual verification: extract bytes from test mode output

When you need a real render, **do not flip test mode off just to get the PPTX bytes**. Test mode already produces the full PPTX — `pptx.create` runs and its output (`@.deck.data`, base64-encoded bytes) is available in the event trace. Extract and render locally:

```bash
# @.deck.data from the event trace is base64 — decode and render
echo "<base64-from-@.deck.data>" | base64 -d > /tmp/deck.pptx
libreoffice --headless --convert-to pdf /tmp/deck.pptx
pdftoppm -png -r 100 /tmp/deck.pdf /tmp/slide
```

Then inspect each `/tmp/slide-N.png`.

If you need the user to see the real render, ask them to trigger the flow from the UI (or disable test mode on their initiative) so it sends the email. That keeps the side effect intentional.

#### What to scan for (whether in PNG or in XML)

- **Overlaps**: shapes whose bounding boxes intersect.
- **Footer collisions**: trailing content (overflow indicators, table rows) ending below the footer's start-Y.
- **Wrong-shape wrapping**: a title's line 2 appearing where a subtitle/detail line should be.
- **Double-escaped entities**: literal `&amp;`/`&lt;` instead of `&`/`<` (grep for `&amp;amp;` in the XML).
- **Empty shapes**: text shapes with empty `<a:t/>` — usually a field-name mismatch between renderer and shaped JSON.
- **Wrong record showing up**: a parent/container record as the first row of a child-records table.

---

### Phase 7 — Iteration loop

When the user reports a visual issue:

1. Locate the affected slide and shape — by reading the slide XML from the `create_deck` event trace, by rendering the XML locally with `soffice` + `pdftoppm`, or both.
2. Decide whether the fix belongs in `shape_data` (data shape, semantics, ordering, deduping) or `build_slides` (positions, sizes, colors, conditional rendering).
3. Edit only the relevant step. Push it with `configure_flow` using `flow_id` + `step_id` + `step`.
4. Re-run **in test mode** to confirm no error and that the slide XML in the event trace reflects the fix.
5. Ask the user to do a real run and share back if visual confirmation is needed.

Bundle related changes into a single iteration round — don't push one fix at a time.

---

## Sending the deck as an email attachment

When the flow should email the `.pptx` as an attachment (not just a link), follow this exact pattern. Every field below is required — omitting or mis-naming any one of them produces a hard error.

### 1. Wire up `pptx.create`

`pptx.create` always returns `{ "data": "<base64>" }`. The bytes live in **`$.deck.data`**.

Always pass the output of `build_slides` as `files`:

```json
"create_deck": {
  "op": "CALL",
  "call": {
    "function_id": "pptx.create",
    "args": {
      "files": "$.slide_files"
    }
  },
  "out": { "set": "@.deck" }
}
```

Any zip entry not present in `files` falls back to the generated 16:9 widescreen default (slide master, layout, relationships, content types). Only include entries that carry data-driven content or differ from the default.

### 2. Wire the attachment in `email.send`

All four fields on every attachment object are required:

| Field | Value | Notes |
|---|---|---|
| `filename` | `"{{ $.deck_title }}.pptx"` | |
| `data` | `"{{ $.deck.data }}"` | base64 from `pptx.create`; already the right format |
| `content_type` | `"application/vnd.openxmlformats-officedocument.presentationml.presentation"` | |
| `disposition` | `"attachment"` | must be explicit |
| `content_id` | any non-empty string, e.g. `"deck"` | **required as a workaround — see note below** |

```json
"send_email": {
  "op": "CALL",
  "call": {
    "function_id": "email.send",
    "args": {
      "to": ["{{ $.ctx.triggered_by.email }}"],
      "subject": "...",
      "body": "...",
      "attachments": [
        {
          "filename": "{{ $.deck_title }}.pptx",
          "data": "{{ $.deck.data }}",
          "content_type": "application/vnd.openxmlformats-officedocument.presentationml.presentation",
          "disposition": "attachment",
          "content_id": "deck"
        }
      ]
    }
  }
}
```

### `content_id` is required on all attachments

**Every attachment must include a non-empty `content_id`**, regardless of disposition. SES rejects the request if the field is absent or an empty string (`Member must have length greater than or equal to 1`). The value is arbitrary — `"deck"`, `"attachment-1"`, etc. — it just must not be empty.

### Common errors and their fixes

| Error | Cause | Fix |
|---|---|---|
| `validate: data is required` | Used `content` instead of `data` on the attachment | Rename to `data` |
| `$.deck.data` is empty / missing | `pptx.create` step failed — check the event trace for the error | Fix the step; `data` is always returned on success |
| SES `contentId` must have length ≥ 1 | Missing or empty `content_id` on attachment | Add `"content_id": "deck"` (or any non-empty string) |
| SES `contentId` error on `attachments.1.member` (index 1, not 0) | Backend injects a second attachment with empty `contentId` | Same fix — ensure your attachment has a non-empty `content_id` |

---

## Encoding & deployment gotchas

### `\n` inside a regex inside a JSON-encoded source string

When script source is delivered as a `source` string in `configure_flow`'s JSON payload, `\n` written in your source becomes a real newline at JSON-parse time, which breaks JS regex parsing ("Invalid regular expression: missing /").

**Fix:** write `\\n` in source string literals (JSON decodes that to `\n`, then JS regex parses it as the newline metachar). Or avoid `\n` in regexes entirely — there's almost always a substitute. If you find yourself reaching for a markdown-fence stripper or a trailing-newline regex, ask whether the input is actually that messy or whether your producer step can output cleaner data.

### Step source size — use `step_id` updates

For non-trivial renderers, the source can be 20–30 KB. Don't pass the full `config` on every update — it forces you to ship every other step too, and risks accidentally dropping one. Use the targeted form:

```javascript
configure_flow({ flow_id, step_id: 'build_slides', step: { /* just this step */ } })
```

This updates one step in place. Unchanged steps are untouched.

### Don't pass `config` for iteration

`configure_flow({ flow_id, config: {...} })` **replaces the entire graph** with what's in `config`. Any step not in the new config is dropped. Use `config` only for initial creation or intentional full rewrites.

---

## Quick reference: recurring overlap fixes

| Pattern | Fix |
|---|---|
| Title wraps, element below at fixed Y collides | Concatenate into a single shape with separator |
| Table row wraps, next row overlaps | Use `<a:tbl>`, OR truncate, OR compute row height with min/max |
| Overflow indicator overlaps footer | Cap `maxRows`, OR position indicator from actual last-row end-Y with footer threshold |
| Spotlight duplicated in list beside it | `.slice(1, N)` in shape step, never in renderer |
| Bottom panel overlaps footer | Compute panel height from content, assert `y + cy ≤ footer_y − gap` |

---

## Rules (don't break these)

- **Step 1 before Step 2.** Iterate visually in Claude chat using the Anthropic PPTX skill. Only create the flow once the user has confirmed the output.
- **`build_slides` always outputs a `files` map.** Never use the `slides` array. Pass `files` directly to `pptx.create`.
- **Seed `build_slides` from validated Step 1 XMLs.** Don't re-derive coordinate math; use what was already approved.
- **Inspect actual records before relying on field names.** Field semantics aren't portable across sources or record types.
- **Two JS steps**: `shape_data` owns data semantics, `build_slides` owns layout. No business logic in the renderer.
- **No LLM step for structured transformations.** Buckets, sorts, formatting, deduping — deterministic JS.
- **Always parse `record` from query results** — it's typically a string, not an object.
- **Skip parent/container records** when iterating item lists.
- **`.slice(1, N)` for "next" lists** rendered next to a spotlight.
- **One escape, in the helper.** Pass raw special chars to `esc()`. Never write `&amp;` in a source string literal.
- **Single shape for variable-length title + meta.** Concatenate with a separator; avoid the two-shape fixed-offset pattern.
- **Real `<a:tbl>` for variable-content tables.** Stacked `<p:sp>` rows only when content length is bounded and predictable.
- **`normAutofit` on individual variable-text shapes.** Safety net, not a layout system.
- **Use `step_id` updates** for iteration; reserve full `config` for initial creation.
- **Every `run_flow` uses `options.test.enabled = true`.** No exceptions during build or iteration. Real artifacts are produced by the user, not as a side effect of the skill.
- **Don't flip test mode off just to get the PPTX bytes.** Test mode gives you the bytes in the event trace — extract them and render locally with `soffice` + `pdftoppm`. Only disable test mode when the user intentionally wants to send a real email.
- **Pass input keys exactly as the schema defines them.** Mismatched key names silently fall back to defaults.
