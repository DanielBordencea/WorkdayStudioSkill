# Workday XSLT Debugging Guide

Approach: form a hypothesis, then test it. Don't change code before you know what's wrong. Most "broken XSLT" bugs are one of about ten things — work through this catalog in roughly the order shown.

## Step 0: get the inputs

Before debugging, you need three things from the user. If you don't have them, ask:

1. The XSLT (full file, not a snippet — namespace declarations live at the top)
2. A sample of the input XML the XSLT is processing
3. The actual output they're getting + what they expected

Skipping this step and guessing wastes everyone's time.

## Top causes, in rough order of frequency

### 1. Namespace mismatch

**Symptom:** Output is empty, or specific elements are missing, even though XPath looks correct.

**Diagnosis:** Compare every `xmlns:*` declaration on the stylesheet against the actual URIs in the input XML.

Common variants:
- Stylesheet declares `xmlns:wd="urn:com.workday.report/RPT_INT0001"` but input uses `RPT_INT0002` (template copied between reports).
- Stylesheet uses bare names (`Worker`) but input has `wd:Worker`.
- Stylesheet declares `xmlns:wd="urn:com.workday/bsvc"` but input is a custom report (different URI).

**Quick check using local-name (debugging only — do not ship):**

```xml
<xsl:value-of select="count(//*[local-name()='Worker'])"/>
```

If this returns >0 but `count(//wd:Worker)` returns 0, you have a namespace bug.

### 2. Empty output / no nodes selected

After ruling out namespaces:

- Did you start XPath at the right level? `Report_Entry` vs. `Report_Data/Report_Entry` vs. `/wd:Report_Data/wd:Report_Entry` — the slash and the prefix change everything.
- Are you using `select` when you meant `match`? In `<xsl:template match="...">`, the pattern is a *match pattern* (subset of XPath). In `<xsl:value-of select="...">`, it's a full XPath expression.
- Is your template firing? Add a temporary `<xsl:message>FIRING template X</xsl:message>` to confirm.

### 3. Wrong output method

**Symptom:** CSV has weird whitespace, XML output has no declaration, special characters appear as entities when you wanted literals.

```xml
<xsl:output method="text"/>     <!-- for CSV / fixed-width / flat -->
<xsl:output method="xml" indent="yes"/>  <!-- for XML output -->
<xsl:output method="html"/>     <!-- rare in Workday -->
```

`method="xml"` is the default. If you forget to set `method="text"` for a CSV stylesheet, you'll see XML escaping (`&amp;`, `&lt;`) in places you don't want them.

### 4. Namespace declarations leaking into output

**Symptom:** Output XML has `xmlns:xs`, `xmlns:peci`, `xmlns:wd` attributes you don't want.

**Fix:** Add `exclude-result-prefixes` to `<xsl:stylesheet>`:

```xml
<xsl:stylesheet version="2.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:wd="urn:com.workday.report/RPT_INT0001"
    xmlns:peci="urn:com.workday/peci"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    exclude-result-prefixes="wd peci xs">
```

If a prefix is *used* in the output body (e.g., you're emitting `<wd:Something>`), `exclude-result-prefixes` won't drop it. Either rename the output element to a non-namespaced version, or accept the declaration.

### 5. ETV/XTT attributes appearing in final output unchanged

**Symptom:** Output looks like:
```xml
<City etv:truncate="true" etv:maxLength="30">Llanfair...</City>
```

**Cause:** The Document Transformation step isn't running after your XSLT. The XSLT emits the ETV/XTT markup correctly, but nothing consumes it.

**Fix:** This isn't an XSLT problem — it's an integration system configuration problem. The integration needs a Document Transformation step that processes ETV/XTT *after* the XSLT step. Tell the user to check the integration system's transformation stack.

### 6. Encoding issues

**Symptom:** Non-ASCII characters (ă, ș, ț, ü, ñ) appear as `?`, `Ã©`, or garbage.

Check:
- `<xsl:output method="..." encoding="UTF-8"/>` — explicitly set UTF-8
- The input file's encoding (Workday emits UTF-8 by default)
- The downstream vendor's expected encoding (some legacy systems need Windows-1252 or ISO-8859-1)

For Workday Studio, set encoding in the route step too — XSLT encoding alone won't override a downstream component that re-encodes.

### 7. Whitespace and newlines

**Symptom:** Output has extra spaces, missing newlines, or text wrapped weirdly.

XSLT preserves whitespace from the stylesheet by default. Common patterns:

```xml
<!-- explicit newline -->
<xsl:text>&#xA;</xsl:text>

<!-- strip whitespace from input -->
<xsl:strip-space elements="*"/>

<!-- preserve whitespace selectively -->
<xsl:preserve-space elements="Comment Description"/>
```

For CSV output, always use `<xsl:text>` for separators and newlines — don't rely on stylesheet formatting:

```xml
<xsl:value-of select="wd:ID"/>
<xsl:text>,</xsl:text>
<xsl:value-of select="wd:Name"/>
<xsl:text>&#xA;</xsl:text>
```

### 8. XPath gotchas

- `//Element` does a full-tree scan. Avoid in large PECI files — performance dies.
- `Element[@attr]` checks for attribute presence, not value. Use `Element[@attr='value']` for value comparison.
- `position()` is 1-indexed.
- `name()` returns the prefixed name (`wd:Worker`); `local-name()` returns just `Worker`.

### 9. XSLT 1.0 vs. 2.0 confusion

Workday Studio supports XSLT 2.0. If you're working from an older example that says `version="1.0"`, you may be missing useful 2.0 features:

- `xsl:for-each-group` (grouping)
- `xsl:function` (user-defined functions)
- Sequences, regular expressions
- `xs:date`, `xs:dateTime` typing

Upgrading to `version="2.0"` rarely breaks anything but enables a lot.

### 10. Performance — large DOMs

Workday Studio loads documents into memory as DOM. For large inputs:

- Avoid `//` (descendant-or-self axis) — full-tree scan
- Avoid deeply nested predicates evaluated in loops — cache with `<xsl:variable>`
- For PECI > ~100k records, consider split-transform-aggregate

## Tools

For local testing outside Studio, Oxygen XML Editor is the most common Workday consultant tool. Saxon HE (free) runs XSLT 2.0/3.0 from the command line and is fine for sanity-checking transformations.

Workday Community has the official documentation behind login. The public references that consistently come up: ZaranTech, Pixentia blog, sglmr.com (good namespace-exclusion writeup), and the Workday-Pro-Integrations exam study guides (which lean on real ETV examples).
