# Workday XSLT Namespaces — Full Reference

Namespaces are the #1 source of "my XSLT returns nothing" bugs in Workday. This file documents every namespace you'll encounter, when it appears, and how to declare and use it.

## The XSLT-side namespace declaration

Every Workday stylesheet starts roughly like this:

```xml
<xsl:stylesheet version="2.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:wd="urn:com.workday.report/RPT_INT0001_Worker_Outbound"
    xmlns:xtt="urn:com.workday/xtt"
    xmlns:etv="urn:com.workday/etv"
    exclude-result-prefixes="wd xtt etv">
```

Three things matter:
1. The **URI** on the right side of `=` must match what the input XML actually uses. Copy it from the input — don't type it from memory.
2. The **prefix** on the left side of `=` is arbitrary, but conventionally `wd`, `peci`, `xtt`, `etv`, `ps`. Don't fight convention; reviewers expect these.
3. `exclude-result-prefixes` lists prefixes that should NOT appear as `xmlns:*` declarations in the output. Without it, your output XML will be cluttered with namespace declarations even when those prefixes aren't used.

## The catalog

### `wd` — Workday business object / report data

Two distinct flavors of `wd:`:

**Custom Report output:**
```
xmlns:wd="urn:com.workday.report/RPT_INT0001_Worker_Outbound"
```
The URI ends with the report's reference ID. **Every report has its own URI.** If you copy an XSLT between reports, you must update this URI. This is a common bug when reusing templates.

**Web service (bsvc) output:**
```
xmlns:wd="urn:com.workday/bsvc"
```
Used when consuming WWS (Workday Web Services) responses like `Get_Workers`, `Get_Positions`, etc. Studio integrations that call SOAP services see this URI.

You can never have both `wd:` URIs in the same stylesheet. If your input mixes report data and bsvc data (rare but possible — e.g., Studio chains a report and a web service call), use two different prefixes:

```xml
xmlns:wdr="urn:com.workday.report/RPT_INT0001_Worker_Outbound"
xmlns:wd="urn:com.workday/bsvc"
```

### `peci` — Payroll Effective Change Interface

```
xmlns:peci="urn:com.workday/peci"
```

Wraps PECI-specific elements (`peci:Effective_Change`, `peci:Worker_Reference`, etc.) and processing attributes (`peci:isDeleted`). See `peci-patterns.md` for what's inside.

### `xtt` — XML To Text

```
xmlns:xtt="urn:com.workday/xtt"
```

Used to add formatting attributes that the Document Transformation step reads when converting XML to text output (CSV, fixed-width, custom delimiter). The XSLT *emits* these attributes; the DT step consumes them.

You only need `xmlns:xtt` declared if your output XSLT actually emits `xtt:*` attributes.

### `etv` — Element Transformation & Validation

```
xmlns:etv="urn:com.workday/etv"
```

Like `xtt`, but for validation and content-level transformations (truncate, required, format, enumerate). Same pattern: declare only if you emit `etv:*` attributes in your output.

### `ps` — PICOF (Position Synch Outbound File)

```
xmlns:ps="urn:com.workday/picof"
```

Less common than PECI. Appears in PICOF integrations. Treat similarly to `peci:` — Workday-specific element prefix.

### `xsl` — the XSLT namespace itself

```
xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
```

Mandatory in every stylesheet. Without it, `<xsl:template>`, `<xsl:for-each>`, etc. wouldn't be recognized.

### `xs` — XML Schema

```
xmlns:xs="http://www.w3.org/2001/XMLSchema"
```

Often appears in inputs (especially anything generated from Workday with type info) and leaks into outputs unless excluded. Include in `exclude-result-prefixes` if you don't need it.

## XPath rules with namespaces

The cardinal rule:

> **An XPath step matches only nodes that share both the local name AND the namespace URI.**

If the input has `<wd:Worker xmlns:wd="urn:com.workday/bsvc">`, then:

- `Worker` matches **nothing** (no namespace).
- `wd:Worker` matches **only if** `wd:` in the stylesheet is bound to `urn:com.workday/bsvc`. If your stylesheet binds `wd:` to the report URI instead, this fails silently.
- `*[local-name()='Worker']` matches by local name only, ignoring namespace. Useful as a debugging tool to confirm the node exists; **never ship code using this** — it's a code smell and hides namespace bugs.

## `exclude-result-prefixes` — keeping output clean

Input XML carries namespace declarations. By default XSLT propagates those into the output, even when no element in the output uses that prefix. The result:

```xml
<!-- output you didn't want -->
<Person xmlns:xs="http://www.w3.org/2001/XMLSchema"
        xmlns:peci="urn:com.workday/peci"
        xmlns:wd="urn:com.workday.report/RPT_INT0001">
  <Name>John</Name>
</Person>
```

Downstream vendors often choke on these declarations. The fix is on the stylesheet element:

```xml
<xsl:stylesheet version="2.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:wd="urn:com.workday.report/RPT_INT0001"
    xmlns:peci="urn:com.workday/peci"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    exclude-result-prefixes="wd peci xs">
```

List every prefix that should not appear as `xmlns:*` in the output.

## Common namespace mistakes

| Symptom | Likely cause |
|---------|--------------|
| Output is completely empty | XPath uses bare names (`Worker`) when source has namespaces (`wd:Worker`) |
| Output is empty after editing the URI | You changed the URI in `xmlns:wd=...` but the source XML still uses the old URI |
| `exclude-result-prefixes` doesn't help | A prefix you're "excluding" is actually used in the output XML body — XSLT won't drop a declaration that's referenced |
| Same XPath works in one report, not another | Each Custom Report has its own URI; the stylesheet was copied without updating `xmlns:wd` |
| `wd:Report_Entry` returns nothing in a WWS context | WWS uses `urn:com.workday/bsvc`, not the report URI. Different namespace, different node identity. |
