---
name: workday-xslt
description: Workday XSLT and Workday Studio integration expertise. Use this skill whenever the user is reading, writing, modifying, or debugging XSLT files in a Workday Integrations context — including Workday Studio projects, EIB attachments, Core Connectors, PECI/PICOF, Document Transformation, Payroll Interface, or any Workday transformation that converts XML to CSV, fixed-width, JSON, or another XML shape. Trigger this skill whenever the user mentions XSLT in a Workday context, shares a file containing Workday namespaces (`wd:`, `peci:`, `xtt:`, `etv:`, `ps:`), or asks about transforming a Workday Custom Report / connector output. Also trigger for debugging empty output, namespace problems, XPath that returns nothing, PECI transaction handling, `@peci:isDeleted` filtering, attribute-based transformations (ETV/XTT), `exclude-result-prefixes` cleanup, or "my transformation isn't producing data" problems. Trigger even when the user says only "Studio", "PECI", "PICOF", "transformation", "stylesheet", or shares a `.xslt`/`.xsl` file without explicitly saying "XSLT" — assume Workday context if the file uses Workday namespaces or the user works in Workday Integrations.
---

# Workday XSLT

A skill for working with XSLT in Workday Integrations: reading existing transformations, writing new ones, and debugging output problems. Workday XSLT differs from generic XSLT primarily in **namespaces** and **Workday-specific processing instructions (ETV/XTT)** — these are where most real-world bugs come from.

## How to use this skill

Identify which mode the user is in, then proceed:

1. **Reading / explaining** — they want to understand what an XSLT does. Read the whole file (not just the part they ask about), trace data flow from input XML to output, and explain in plain terms what gets selected, transformed, and emitted.

2. **Writing new** — they describe a target output and a Workday source. Ask for a sample of the input XML if not provided (or for the report name / structure). Then write the stylesheet. Never invent field names — if you don't have a sample, ask.

3. **Modifying existing** — they share an XSLT and describe a change. Make the minimum diff needed; do not rewrite the whole file.

4. **Debugging** — they describe broken behavior (empty output, wrong values, errors). Follow the debugging workflow below before proposing any fix.

## The debugging workflow (use this always for "it's not working")

Before changing anything, work through these in order. Most Workday XSLT bugs are caught in steps 1–3.

1. **Namespaces declared vs. namespaces used.** Look at `xmlns:*` on the `<xsl:stylesheet>` element. For each prefix used in XPath (`wd:Worker`, `peci:Effective_Change`, etc.), confirm the prefix is declared and the URI matches the source XML. *This is the #1 cause of empty output in Workday XSLT.*

2. **XPath against actual input.** Ask the user to share a snippet of the real input XML if you haven't seen it. Match the XPath against that structure literally. `Worker` and `wd:Worker` are different nodes; `/Workers/Worker` and `//Worker` behave differently with namespaces.

3. **Output method.** Check `<xsl:output method="..."/>`. For CSV / fixed-width / flat file, must be `text`. For XML output, must be `xml`. Wrong output method silently breaks formatting (whitespace, encoding, escaping).

4. **Template match.** If templates aren't firing, check `<xsl:template match="..."/>` — does the match expression actually match a node that exists with that namespace prefix?

5. **`exclude-result-prefixes`.** If the user complains about stray `xmlns:xs` or `xmlns:peci` declarations in the output, this is the fix. Add `exclude-result-prefixes="xs peci wd"` (or whichever prefixes leak) to `<xsl:stylesheet>`.

6. **ETV/XTT attributes still present in final output.** If ETV/XTT attributes (`etv:truncate`, `xtt:separator`, etc.) appear in the final output instead of being processed, the Document Transformation step hasn't run after the XSLT. Confirm the integration system has the DT step configured.

State your hypothesis to the user before changing code. Don't rewrite the stylesheet — propose the minimum change and explain why.

## Critical Workday namespaces (quick reference)

| Prefix | URI | What it carries |
|--------|-----|-----------------|
| `wd` | `urn:com.workday/bsvc` or `urn:com.workday.report/{REPORT_NAME}` | Business object data, Custom Report output |
| `peci` | `urn:com.workday/peci` | PECI elements and processing attributes |
| `xtt` | `urn:com.workday/xtt` | XML-to-Text processing instructions |
| `etv` | `urn:com.workday/etv` | Element Transformation & Validation rules |
| `ps` | `urn:com.workday/picof` | PICOF (Position Synch Outbound File) |
| `xsl` | `http://www.w3.org/1999/XSL/Transform` | XSLT itself |
| `xs` | `http://www.w3.org/2001/XMLSchema` | Schema typing (often leaks into output) |

Full details, declaration examples, and "which namespace do I use when" in `references/namespaces.md`.

## ETV and XTT — Workday's processing instructions

ETV and XTT are *not* alternative transformation languages. They are attributes that XSLT *outputs* into the XML, which then get processed by Workday's Document Transformation step. The XSLT generates the markup; the DT step consumes the attributes and applies the rules.

**ETV** (`urn:com.workday/etv`) — validates and transforms element content:
- `etv:truncate="true" etv:maxLength="30"` — cut value at 30 chars
- `etv:required="true"` — fail if missing
- `etv:target="WID"` — used with identifier lookups
- Date/number formatting, padding, enumeration validation

**XTT** (`urn:com.workday/xtt`) — converts XML to text formats:
- `xtt:separator="&#xA;"` — newline-separate rows
- `xtt:quote="..."` — CSV quoting rules
- `xtt:pad`, `xtt:width` — fixed-width formatting

Full attribute lists with working examples in `references/etv-xtt.md`. Read it whenever the user uses or mentions any `etv:*` or `xtt:*` attribute.

## PECI specifics

PECI XSLT is a distinct sub-skill. Read `references/peci-patterns.md` whenever:
- The XML has `xmlns:peci="urn:com.workday/peci"`
- The user mentions PECI, PICOF, Payroll Interface, transaction handling, effective changes, change detection, or top-of-stack logic
- You see `peci:Effective_Change`, `peci:isDeleted`, `peci:Transaction_*` elements

Key PECI facts kept top-of-mind:
- `@peci:isDeleted="true"` marks records the downstream system should treat as deletions/skips. Filter with `[not(@peci:isDeleted='true')]` when iterating, depending on requirement.
- PECI streams **effective changes in sequence**. Order matters — don't sort unless the vendor explicitly requires it.
- A "full extract" run flips behavior (launch parameter); change-detection runs only emit deltas.

## Skeleton: a working Workday XSLT

Use this as a starting point when writing from scratch. Replace the report namespace URI with the actual one from the source XML.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="2.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:wd="urn:com.workday.report/RPT_INT0001_Worker_Outbound"
    xmlns:xtt="urn:com.workday/xtt"
    xmlns:etv="urn:com.workday/etv"
    exclude-result-prefixes="wd xtt etv">

  <xsl:output method="text" encoding="UTF-8"/>

  <xsl:template match="/">
    <xsl:for-each select="wd:Report_Data/wd:Report_Entry">
      <xsl:value-of select="wd:Employee_ID"/>
      <xsl:text>,</xsl:text>
      <xsl:value-of select="wd:Legal_Name"/>
      <xsl:text>&#xA;</xsl:text>
    </xsl:for-each>
  </xsl:template>

</xsl:stylesheet>
```

What this demonstrates:
- Namespace declared with `xmlns:wd=...` matching the report's URI
- XPath uses `wd:` prefix consistently — never bare `Report_Entry`
- `method="text"` because output is CSV
- `&#xA;` for newline (entity, not literal)
- `exclude-result-prefixes` to keep output clean

## What to ask for when the user doesn't share enough

Before writing or fixing, you almost always need:
- A snippet of the input XML (at least the relevant elements with their namespace declarations)
- The desired output format (with a sample row if possible)
- For debugging: the actual output the user is seeing AND what they expected

Don't guess at field names. Don't assume the namespace URI. Ask once, concisely.

## Reference files

Read these on demand based on what the task involves:

- `references/namespaces.md` — Full namespace reference, declaration patterns, common URI variations between reports and bsvc.
- `references/etv-xtt.md` — Complete ETV and XTT attribute catalog with examples for truncate, validate, CSV, fixed-width, padding, date/number formatting.
- `references/peci-patterns.md` — PECI structure, `isDeleted` handling, effective change sequencing, full vs. change-detection runs, top-of-stack logic, common PECI XSLT patterns.
- `references/debugging.md` — Expanded error catalog: empty output, leaking namespaces, ETV/XTT not processed, encoding issues, performance pitfalls (DOM size, MVEL).

## Style notes for Workday XSLT specifically

- **Workday Studio loads documents into memory as DOM.** Large input → memory pressure. For >100k records, mention split-transform-aggregate as an option even if the user didn't ask.
- **Stay XSLT 2.0 unless you have a reason.** Studio's XSLT processor supports 2.0; using 2.0 features (sequences, `xsl:function`, grouping) is preferred over 1.0 workarounds.
- **Comment liberally in PECI XSLT.** PECI logic is hard to follow six months later. Inline comments explaining the *why* (not the *what*) save real time.
- **Don't reformat the user's XSLT when fixing one issue.** Preserve their style. Minimum diff.
