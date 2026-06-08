# PECI Patterns

PECI (Payroll Effective Change Interface) is Workday's stream-of-changes integration for payroll. Unlike a full snapshot like EIB, PECI delivers **effective changes in sequence** — each worker change with its effective date, sorted chronologically, with enough context for the downstream payroll system to apply changes correctly.

This file covers PECI-specific XSLT patterns. Read it when:
- Input XML has `xmlns:peci="urn:com.workday/peci"`
- The user mentions PECI, payroll changes, effective dating, top-of-stack, change detection
- You see `peci:Effective_Change`, `peci:isDeleted`, `peci:Transaction_*`

## The PECI XML shape (typical)

```xml
<peci:PECI xmlns:peci="urn:com.workday/peci">
  <peci:Effective_Change>
    <peci:Worker_Reference>
      <peci:ID type="Employee_ID">12345</peci:ID>
    </peci:Worker_Reference>
    <peci:Effective_Date>2026-01-15</peci:Effective_Date>
    <peci:Entry_DateTime>2026-01-20T09:32:00</peci:Entry_DateTime>
    <peci:Transaction_Reference>
      <peci:ID type="Transaction_ID">TXN-998877</peci:ID>
    </peci:Transaction_Reference>
    <peci:Payroll_Data>
      <peci:Annual_Salary peci:isDeleted="false">75000</peci:Annual_Salary>
      ...
    </peci:Payroll_Data>
  </peci:Effective_Change>
  
  <peci:Effective_Change>
    ...
  </peci:Effective_Change>
</peci:PECI>
```

Each `<peci:Effective_Change>` represents one delta. A worker can have many in a single PECI run.

## The `@peci:isDeleted` attribute

This is the attribute people most often get wrong. Its meaning depends on context:

- On a **data element** (e.g., `<peci:Annual_Salary peci:isDeleted="true">`), it signals that the field was *cleared* in Workday — the downstream system should treat the field as removed/null, not retain the previous value.
- On an `<peci:Effective_Change>` itself, it signals the change was *retracted* (rescinded business process).

Filtering decisions you'll commonly need to make:

**Skip retracted changes:**
```xml
<xsl:for-each select="peci:Effective_Change[not(@peci:isDeleted='true')]">
  ...
</xsl:for-each>
```

**Emit deletions explicitly (vendor needs to know):**
```xml
<xsl:for-each select="peci:Effective_Change">
  <Record>
    <Action>
      <xsl:choose>
        <xsl:when test="@peci:isDeleted='true'">DELETE</xsl:when>
        <xsl:otherwise>UPDATE</xsl:otherwise>
      </xsl:choose>
    </Action>
    ...
  </Record>
</xsl:for-each>
```

Always confirm with the user which interpretation the downstream payroll vendor expects — getting `isDeleted` wrong silently corrupts payroll.

## Effective change sequencing — do not sort

PECI emits changes in the order Workday expects them to be applied. **Do not sort** by effective date, entry date, or worker ID unless the vendor specifically requires it. Re-sorting can break:

- Multiple same-day changes (order is determined by entry time, not just effective date)
- Retractions that follow the change they retract
- Compensation changes that depend on a position change emitted just before

If a vendor *does* require sorting (e.g., group by worker, then by date), document it loudly in the XSLT comments. Future you will thank present you.

## Top-of-stack logic

"Top of stack" means: for a given worker on a given effective date, the most recent transaction (by entry time) is the one that wins.

PECI delivers all transactions, including superseded ones. If the vendor only wants the final state, you need to filter:

```xml
<!-- For each (worker, effective_date), keep only the row with the latest Entry_DateTime -->
<xsl:for-each-group select="peci:Effective_Change"
    group-by="concat(peci:Worker_Reference/peci:ID[@type='Employee_ID'], '|',
                      peci:Effective_Date)">
  <xsl:for-each select="current-group()">
    <xsl:sort select="peci:Entry_DateTime" order="descending"/>
    <xsl:if test="position() = 1">
      <!-- emit this row -->
    </xsl:if>
  </xsl:for-each>
</xsl:for-each-group>
```

(`for-each-group` requires XSLT 2.0, which Workday Studio supports.)

## Full extract vs. change detection

PECI runs in two modes, controlled by a launch parameter:

- **Change detection (default for scheduled runs):** only emits changes since the last successful run. The XSLT sees only deltas.
- **Full extract:** emits everything regardless of last-run state. Often used for vendor initialization or re-sync.

You generally don't need to handle this differently in XSLT — Workday gives you what it gives you, and your XSLT transforms the input it sees. The thing to know: if a customer says "I'm only seeing 3 records, where's the rest?", they may be on a change-detection run, not a full extract. That's an integration configuration issue, not an XSLT bug.

## Effective Date vs. Entry DateTime

These are different and both matter:

- `peci:Effective_Date` — when the change takes effect in the business. ("Salary increase effective Feb 1.")
- `peci:Entry_DateTime` — when the change was entered in Workday. ("HR clicked submit on Jan 25 at 14:32.")

Most vendors care about Effective_Date for processing logic and Entry_DateTime only for tie-breaking same-day changes.

## Worker_Reference — pick the right ID type

```xml
<peci:Worker_Reference>
  <peci:ID type="WID">5b1a2c3d...</peci:ID>
  <peci:ID type="Employee_ID">12345</peci:ID>
  <peci:ID type="Pay_Group_ID">PG-US-EAST</peci:ID>
</peci:Worker_Reference>
```

A worker reference contains multiple IDs. To pick a specific one:

```xml
<xsl:value-of select="peci:Worker_Reference/peci:ID[@type='Employee_ID']"/>
```

WID is internal Workday — most vendors want Employee_ID or a custom external identifier.

## Common PECI XSLT bugs

| Symptom | Likely cause |
|---------|--------------|
| Some workers missing from output | XPath filter is too aggressive — likely `[not(@peci:isDeleted='true')]` applied at wrong level |
| Records appear out of order | Stylesheet is sorting; remove the sort or align it with vendor spec |
| Same worker appears multiple times when they shouldn't | Need top-of-stack logic — currently emitting all changes, not the final state |
| `isDeleted` records reach vendor as live updates | Filter not applied, or applied at wrong nesting level |
| Pay group missing | Looking at wrong `peci:ID[@type=...]` — verify with input XML |
| Effective_Date shows wrong format | Workday emits ISO `YYYY-MM-DD`; vendor may need reformatting via `etv:dateFormat` |

## Performance note

PECI files for large tenants can be huge (hundreds of MB). If the user is running into Studio memory issues:

- Consider streaming (STX) for early filtering before XSLT
- Split by pay group at the integration level, run separate XSLT per group
- Avoid `//` axis steps in PECI XSLT — full-tree scans on large DOMs are slow
