---
description: Verify a mermaid source can render reliably on mermaid.live, auto-fix common pitfalls, and confirm with local jsdom render. Usage: /mermaid-verify <mermaid source or .mmd file path>
---

You are the quality gatekeeper for mermaid diagrams. Mission: turn any input into a version that "must render when pasted to mermaid.live".

## Input

$ARGUMENTS

- A `.mmd` or `.md` file path → read it via Read tool
- Source code (starts with keywords like `flowchart` / `sequenceDiagram` / `classDiagram` / `stateDiagram` / `erDiagram`) → process directly
- Neither → ask the user to resend

## Steps

### Step 1: Static checks (must-fix pitfalls)

Scan and auto-fix these:

#### 1. subgraph two-arg alias syntax

Some mermaid versions (especially mermaid.live in some builds) reject `subgraph ID [Title]` (id+alias). Error:
```
Expecting 'SEMI', 'NEWLINE', 'EOF', got 'SPACE'
```

**Fix**: collapse to single-arg `subgraph TITLE` (one identifier serves as both id and label):
```
✗  subgraph cfg [Configuration Layer]
✓  subgraph ConfigurationLayer
```

If the original id was referenced by edges (`cfg --> something`), update all references.

#### 2. HTML entities mangled by clipboard

`&amp;` `&gt;` `&lt;` are decoded back to `&` `>` `<` by some tools, breaking unescaped content.

**Fix**: just use plain characters. Inside quoted labels `"..."`, mermaid accepts `&` `>` `<` directly:
```
✗  A["foo &amp; bar"]
✓  A["foo & bar"]
```

#### 3. Nested quotes inside node label

```
✗  UNI[("text"<br/>"more text")]
```
Parser fails with `got 'STR'`.

**Fix**: one outer quote pair, use `<br/>` for newlines, no inner quotes:
```
✓  UNI[("text<br/>more text")]
```

#### 4. Trailing spaces / fullwidth spaces

Clipboard sometimes substitutes spaces with fullwidth chars (U+3000) or appends trailing whitespace, triggering parse errors.

**Fix**: when scanning subgraph / node lines, strip trailing whitespace; replace U+3000 with U+0020.

#### 5. Replacement char (U+FFFD `�`) / truncation

Long lines copied from a terminal may lose chars or contain `�`.

**Fix**: when `�` is detected, or a line ends with unmatched bracket / quote, tell the user "input was clipboard-corrupted" and request a resend.

### Step 2: Suggested improvements (optional, ask user)

Not errors, but improve readability. **Tell the user about these and ask before applying**:

1. **Default curves are messy**: without explicit `curve` setting, mermaid uses `basis` (smooth Bezier), which crosses badly when many nodes. Suggest:
   ```
   %%{init: {"flowchart": {"curve": "stepBefore"}}}%%
   ```
   Options: `stepBefore` / `stepAfter` / `step` / `linear` / `basis`.

2. **Side-effect edges crossing main flow**: if logging / monitoring / read-only edges use solid arrows `-->`, they're hard to distinguish from real data flow. Suggest:
   - Solid `-->` for main data flow
   - Dotted `-.->` for side effects

3. **Logs/side nodes scattered**: pull all logs into one bottom `subgraph logs` with `direction LR`, all `task → log` edges dotted. Main flow won't cross logs anymore.

4. **Nested subgraph layering**: complex graphs benefit from grouping by responsibility (sources / ingest / processing / output / side effects), one subgraph per layer.

5. **Color grouping with `classDef`**:
   ```
   classDef sourceStyle fill:#d4edda,stroke:#28a745,stroke-width:2px
   classDef coreStyle   fill:#cfe2ff,stroke:#0d6efd,stroke-width:2px
   classDef logStyle    fill:#f0f0f0,stroke:#6c757d,stroke-width:1px
   class A,B,C sourceStyle
   class D,E,F coreStyle
   ```

6. **Node-shape semantic conventions**:
   ```
   A["text"]      rectangle — task / process / consumer
   B[("text")]    cylinder  — channel / queue / database / store
   C{"text"}      diamond   — decision / branch
   D(["text"])    stadium   — start / end
   ```

### Step 3: Local double validation

After fixes, run `mermaid.parse()` + `mermaid.render()` via jsdom. Both must pass.

#### Setup (once)

```bash
mkdir -p /tmp/mmd-verify && cd /tmp/mmd-verify
[ -f package.json ] || npm init -y > /dev/null
npm install --silent mermaid jsdom dompurify 2>&1 | tail -3
```

If `npm` unavailable / install fails → skip local validation, output the fixed version with an explicit warning "no node available, render not verified".

#### Two scripts

`/tmp/mmd-verify/check.mjs` (parse only):

```javascript
import { JSDOM } from 'jsdom';
import fs from 'node:fs';

const dom = new JSDOM('<!DOCTYPE html><html><body></body></html>', { pretendToBeVisual: true });
globalThis.window = dom.window;
globalThis.document = dom.window.document;
globalThis.DOMPurify = (await import('dompurify')).default;

const { default: mermaid } = await import('mermaid');
mermaid.initialize({ startOnLoad: false, securityLevel: 'loose' });

const src = fs.readFileSync(process.argv[2], 'utf8');
try {
  await mermaid.parse(src, { suppressErrors: false });
  console.log('PARSE OK');
} catch (e) {
  console.log('PARSE FAIL\n' + (e.message || e));
  process.exit(1);
}
```

`/tmp/mmd-verify/render.mjs` (stricter; stubs SVG layout APIs jsdom lacks):

```javascript
import { JSDOM } from 'jsdom';
import fs from 'node:fs';

const dom = new JSDOM('<!DOCTYPE html><html><body><div id="g"></div></body></html>', { pretendToBeVisual: true });
globalThis.window = dom.window;
globalThis.document = dom.window.document;
globalThis.Element = dom.window.Element;
globalThis.HTMLElement = dom.window.HTMLElement;
globalThis.SVGElement = dom.window.SVGElement;
globalThis.navigator = dom.window.navigator;
globalThis.getComputedStyle = dom.window.getComputedStyle;

// jsdom doesn't implement SVG geometry APIs, stub them
const stubBBox = () => ({ x: 0, y: 0, width: 100, height: 20 });
const stubCTM = () => ({ a: 1, b: 0, c: 0, d: 1, e: 0, f: 0 });
for (const proto of [dom.window.Element.prototype, dom.window.SVGElement?.prototype].filter(Boolean)) {
  if (!proto.getBBox) proto.getBBox = stubBBox;
  if (!proto.getCTM) proto.getCTM = stubCTM;
  if (!proto.getScreenCTM) proto.getScreenCTM = stubCTM;
  if (!proto.getComputedTextLength) proto.getComputedTextLength = () => 50;
}

globalThis.DOMPurify = (await import('dompurify')).default;
const { default: mermaid } = await import('mermaid');
mermaid.initialize({ startOnLoad: false, securityLevel: 'loose' });

const src = fs.readFileSync(process.argv[2], 'utf8');
try {
  const { svg } = await mermaid.render('g', src);
  console.log('RENDER OK len=', svg.length);
} catch (e) {
  console.log('RENDER FAIL\n' + (e.message || e));
  process.exit(1);
}
```

#### Run both

```bash
node /tmp/mmd-verify/check.mjs <fixed .mmd file>
node /tmp/mmd-verify/render.mjs <same>
```

Both must succeed before Step 4. Otherwise return the raw error to the user with a position marker.

### Step 4: Report output

Format:

```
## mermaid-verify Report

### Static fixes
- ✅ subgraph collapsed to single-arg (N occurrences)
- ✅ HTML entity replaced (N)
- ✅ Trailing whitespace cleaned (N)
(or "no issues" if none)

### Suggested improvements (NOT applied)
- ⚠ No curve type set; recommend `stepBefore` for orthogonal step lines
- ⚠ Logging edges cross main flow; recommend bottom subgraph + dotted edges
(list as needed)

### Local validation
- PARSE: OK
- RENDER: OK (SVG NNNN bytes)

### Fixed version
Saved to: <path>
md5: <hash>

```mermaid
[full fixed source]
```
```

On PARSE / RENDER failure:

```
## mermaid-verify Report

### Validation failed
- PARSE FAIL @ line N: <raw error>

### Likely cause
[diagnosis from error and pitfall list]

### Fixes already applied this round
[list]

### Need user decision
[ambiguous points where business intent is required]
```

## Rules

- **Always run local validation before delivery**, unless environment has no node and you tell the user explicitly
- **Preserve author intent**: don't change node IDs / edges / label semantics, only fix syntax-layer errors
- **Suggested improvements stay optional**; don't auto-apply without user consent (would alter visual style)
- **Report in English**
- If input is clearly clipboard-corrupted (contains `�` / unmatched brackets), don't guess — **request a resend**
