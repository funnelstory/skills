# FunnelStory PPTX Deck Builder

## Purpose

Build a FunnelStory flow that generates a `.pptx` file per account and iterate on layout until the output matches expectations. Covers requirements gathering, query validation, flow authoring (`semantic.query` / `data_connection.query` → `script.js` → `pptx.create`), test runs, preview links, reading XML for feedback, and design iteration.

Use this when the user wants to **automate** deck generation (scheduled or on-demand flows), not a one-off QBR deliverable — for that, use `qbr-internal/`, `qbr-external/`, or `qbr-deck/`.

## Prerequisites

- **FunnelStory MCP** connected for the target workspace.
- **`read_resource("file://flow_guide.md")`** — flow argument shapes and step wiring.
- **`read_resource("file://semantic/schema.sql")`** and **`read_resource("file://semantic/usage.md")`** — write correct semantic queries.
- **`query_semantic_db`** — validate semantic SQL before embedding in a flow.
- **`get_data_connection_schema`** / **`preview_data_connection`** — confirm external connection schema.
- **`configure_flow`** / **`run_flow`** — save and test the flow.
- **`read_presentation`** — inspect slide XML when iterating on visual feedback.

---

## Phase 0 — Read before doing anything

Call these resources before writing a single line of JSON:

```
read_resource("file://flow_guide.md")
read_resource("file://semantic/schema.sql")
read_resource("file://semantic/usage.md")
```

You need the schema to write correct queries and the flow guide for exact argument shapes.

---

## Phase 1 — Requirements gathering

Ask **exactly once** (batch all questions together):

1. **Account to target** — which account name or ID to use for the first test run.
2. **Slide content** — what should each slide show? Walk through a list like:
   - Title / cover slide (account name, date, overall status)
   - Key metrics (which ones? from semantic DB or external connection?)
   - Activity / events summary (last N days?)
   - Action items / tasks
   - Any other sections they have in mind
3. **Data sources** — is all data in the semantic DB, or does it come from an external connection (Salesforce, HubSpot, ClickUp, Slack, etc.)? Get connection IDs if needed.
4. **Visual style** — dark or light background? Brand colours (hex)? Any reference deck they can share?

Do not proceed until you have answers to (1) and (2). (3) and (4) can default if they don't know yet.

---

## Phase 2 — Validate queries before building the flow

For each data requirement identified in Phase 1:

- **Semantic DB data** (accounts, activities, tasks, dataset records): write the SQL and call `query_semantic_db` with it. Confirm you get rows back for the target account before embedding it in the flow.
- **External connection data**: call `preview_data_connection` or `get_data_connection_schema` to confirm the table/column names, then note that you cannot pre-validate — it will be tested via `run_flow` in Phase 3.

Do **not** embed a query in the flow that you have not validated (semantic) or at minimum confirmed the schema for (external).

---

## Phase 3 — Build the flow

### Flow structure

```
[data query step(s)]  →  [script.js — build slide XMLs]  →  [pptx.create]
```

Each query step outputs into a named variable (e.g. `@.account_data`, `@.tasks`). The `script.js` step receives all of them via `input` and returns an array of slide XML strings. `pptx.create` receives that array.

### `script.js` contract

- `input` receives whatever you pass in `args.input` — pass a single object containing all fetched data.
- `result` must be an array of raw `<p:sld>` XML strings, one per slide.
- The script must be **synchronous** — no `await`, `Promise`, or network calls.
- 5 s wall-clock limit, 64 KiB source limit.
- Use `ctx.now` (RFC3339 string) for the report date if needed.

### Minimal slide XML shape

Every slide XML must be a self-contained `<p:sld>` document with the correct namespaces:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<p:sld xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main"
       xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships"
       xmlns:p="http://schemas.openxmlformats.org/presentationml/2006/main">
  <p:cSld>
    <p:bg>
      <p:bgPr>
        <a:solidFill><a:srgbClr val="0F172A"/></a:solidFill>
      </p:bgPr>
    </p:bg>
    <p:spTree>
      <p:nvGrpSpPr>
        <p:cNvPr id="1" name=""/><p:cNvGrpSpPr/><p:nvPr/>
      </p:nvGrpSpPr>
      <p:grpSpPr>
        <a:xfrm><a:off x="0" y="0"/><a:ext cx="0" cy="0"/>
                <a:chOff x="0" y="0"/><a:chExt cx="0" cy="0"/></a:xfrm>
      </p:grpSpPr>
      <!-- shapes go here -->
    </p:spTree>
  </p:cSld>
  <p:clrMapOvr><a:masterClrMapping/></p:clrMapOvr>
</p:sld>
```

Slide canvas is **12 192 000 × 6 858 000 EMUs** (widescreen 16:9). All positions and sizes in EMUs (1 inch = 914 400 EMUs).

### Text shape template

```xml
<p:sp>
  <p:nvSpPr><p:cNvPr id="2" name="title"/><p:cNvSpPr/><p:nvPr/></p:nvSpPr>
  <p:spPr>
    <a:xfrm><a:off x="640080" y="2194560"/><a:ext cx="10972800" cy="960120"/></a:xfrm>
    <a:prstGeom prst="rect"><a:avLst/></a:prstGeom>
    <a:noFill/>
  </p:spPr>
  <p:txBody>
    <a:bodyPr wrap="square" lIns="0" tIns="0" rIns="0" bIns="0" anchor="ctr"/>
    <a:lstStyle/>
    <a:p>
      <a:pPr><a:buNone/></a:pPr>
      <a:r>
        <a:rPr lang="en-US" sz="4800" b="1" dirty="0">
          <a:solidFill><a:srgbClr val="FFFFFF"/></a:solidFill>
          <a:latin typeface="Calibri"/>
        </a:rPr>
        <a:t>TITLE TEXT HERE</a:t>
      </a:r>
    </a:p>
  </p:txBody>
</p:sp>
```

`sz` is in hundredths of a point (4800 = 48 pt). Escape XML special chars in text: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`.

### Colored badge / status indicator

```xml
<!-- Background rounded rect -->
<p:sp>
  <p:nvSpPr><p:cNvPr id="5" name="badge_bg"/><p:cNvSpPr/><p:nvPr/></p:nvSpPr>
  <p:spPr>
    <a:xfrm><a:off x="640080" y="4709160"/><a:ext cx="2743200" cy="502920"/></a:xfrm>
    <a:prstGeom prst="roundRect">
      <a:avLst><a:gd name="adj" fmla="val 14545"/></a:avLst>
    </a:prstGeom>
    <a:solidFill><a:srgbClr val="DC2626"/></a:solidFill>
  </p:spPr>
</p:sp>
```

### `pptx.create` call step

```json
{
  "id": "create_deck",
  "op": "CALL",
  "next": "",
  "out": { "set": "@.deck" },
  "call": {
    "function_id": "pptx.create",
    "args": {
      "title": "{{ $.account_name }} – Weekly Update",
      "slides": "{{ @.slide_xmls }}"
    }
  }
}
```

Returns `{ "file_id": "...", "url": "https://drive.google.com/file/d/.../view" }`. The URL is **publicly accessible** to anyone with the link — `ShareAnyoneReader` is called automatically.

### `script.js` step wiring

```json
{
  "id": "build_slides",
  "op": "CALL",
  "next": "create_deck",
  "out": { "set": "@.slide_xmls" },
  "call": {
    "function_id": "script.js",
    "args": {
      "source": "/* your JS here — result = [slide1XML, slide2XML, ...] */",
      "input": {
        "account": "{{ @.account_data }}",
        "tasks":   "{{ @.tasks }}",
        "date":    "{{ $.ctx.now }}"
      }
    }
  }
}
```

### Writing the `script.js` source

Write a self-contained JS program inside the `source` string. Keep it readable — functions, variables, loops are all fine. Typical structure:

```javascript
function escapeXml(s) {
  return String(s)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}

function statusColor(status) {
  var s = (status || '').toLowerCase();
  if (s.indexOf('high risk') !== -1) return 'DC2626';
  if (s.indexOf('at risk')   !== -1) return 'D97706';
  return '16A34A';
}

function titleSlide(account, date) {
  // ... return full <p:sld> XML string
}

function metricsSlide(metrics) {
  // ... return full <p:sld> XML string
}

var slides = [];
slides.push(titleSlide(input.account, input.date));

if (input.tasks && input.tasks.results && input.tasks.results.length > 0) {
  slides.push(tasksSlide(input.tasks.results));
}

result = slides;
```

Use `if` guards and `for` loops for dynamic content — the number of slides can vary per account.

---

## Phase 4 — Save and test run

1. Call `configure_flow` to save the flow.
2. Call `run_flow` with the target account's trigger data and `options.test.enabled = true`.
3. Inspect the output for `deck.url` and `deck.file_id`.
4. Share the URL with the user: *"Here's the deck: [link]. Open it to check the layout and tell me what to change."*

If the test run fails on a step, read the event trace from the response and fix the issue before asking the user for visual feedback.

---

## Phase 5 — Interpret feedback and iterate

When the user provides visual feedback ("the title is too small", "status badge should be on the right", "missing the ARR metric"):

1. Call `read_presentation` with `file_id` from the last run. This returns `{ slides: [{ index, xml }] }`.
2. Parse the XML mentally to map the current state:
   - Identify shape positions (`<a:off x=... y=...>`) and sizes (`<a:ext cx=... cy=...>`).
   - Find text values (`<a:t>...</a:t>`) and font sizes (`sz=`).
   - Find fill colors (`<a:srgbClr val=...>`).
3. Describe the current layout back to yourself (internal reasoning, not shown to user) so you understand exactly what needs to change.
4. Update the `script.js` `source` in the flow to apply the change — adjust positions, sizes, colors, add/remove shapes, add/remove slides.
5. Call `configure_flow` to save the updated flow.
6. Call `run_flow` again (test mode) for the same account.
7. Call `read_presentation` on the new `file_id` to confirm the change took effect in the XML.
8. Only then tell the user: *"Updated — here's the new deck: [link]. The title is now 64pt and the badge moved to the bottom-right."*

Repeat until the user approves.

---

## Phase 6 — Confirm and close

Once the user is happy, confirm:

- The flow is saved and active.
- The trigger is set up correctly (scheduled, account-based, or on-demand).
- The output link will be regenerated fresh each run.
- If email delivery was requested, the `email.send` step is wired after `pptx.create` using `{{ @.deck.url }}` in the body.

---

## Rules

- **Never guess XML namespace prefixes** — always use the full declarations shown in the templates above.
- **Always escape user-facing strings** in XML using `escapeXml()` in the JS. Account names, task titles, and metric values can contain `&`, `<`, `>`.
- **EMU arithmetic**: 1 pt = 12 700 EMU. 1 inch = 914 400 EMU. Widescreen slide = 12 192 000 × 6 858 000 EMU.
- **Do not call `read_presentation` speculatively** — only call it when you have a `file_id` and the user has given feedback that requires interpreting the current XML.
- **One iteration at a time** — make all the user's requested changes in a single `configure_flow` + `run_flow` cycle, not one change per cycle.
- **Test mode** (`options.test.enabled = true`) on every run during iteration. Only remove it for the final production-ready run if the user explicitly asks to send/deliver the deck.
- **Validate queries** (`query_semantic_db`) for any semantic SQL you add or change during iteration, before saving.
