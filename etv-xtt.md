# ETV and XTT — Attribute Reference

ETV (Element Transformation & Validation) and XTT (XML To Text) are Workday's processing-instruction systems for the Document Transformation step. They are **not** alternative transformation languages — they are attributes you emit from XSLT, which the DT step then consumes to transform or format the data.

The flow:

```
Input XML
    ↓
Your XSLT  (emits ETV/XTT attributes on output elements)
    ↓
Intermediate XML with etv:* / xtt:* attributes
    ↓
Document Transformation step (consumes attributes, removes them, applies rules)
    ↓
Final output (XML, CSV, fixed-width, etc.)
```

So in your XSLT you write something like:

```xml
<City etv:truncate="true" etv:maxLength="30">
  <xsl:value-of select="wd:City"/>
</City>
```

And the DT step produces:

```xml
<City>Llanfairpwllgwyngyllgogerychwy</City>
```

(value truncated, attribute stripped).

## ETV — content-level transformations and validations

Declare: `xmlns:etv="urn:com.workday/etv"`

### Truncation

```xml
<City etv:truncate="true" etv:maxLength="30">...</City>
```

`etv:truncate="true"` allows truncation when length exceeds `etv:maxLength`. Without `etv:truncate="true"`, a value over `maxLength` causes the DT step to log an error (or fail, depending on validation setup).

To **require** truncation reporting (vendor wants to know which records were cut):

```xml
<City etv:truncate="true" etv:maxLength="30" etv:reportTruncation="true">...</City>
```

### Required fields

```xml
<SSN etv:required="true"><xsl:value-of select="wd:SSN"/></SSN>
```

If the element is empty or missing after XSLT runs, DT raises a validation error. Used heavily in payroll integrations (vendor requires SSN, Date of Birth, etc.).

### Padding

```xml
<EmployeeID etv:pad="left" etv:padCharacter="0" etv:width="10">1001</EmployeeID>
```

Output after DT: `0000001001`

`etv:pad` can be `"left"` or `"right"`. Common for legacy payroll vendors expecting fixed-format identifiers.

### Date formatting

```xml
<HireDate etv:dateFormat="MM/dd/yyyy"><xsl:value-of select="wd:Hire_Date"/></HireDate>
```

Reformats from Workday's ISO date (`2025-06-08`) to whatever the vendor wants. Patterns follow Java's `SimpleDateFormat`.

### Number formatting

```xml
<Salary etv:numberFormat="#,##0.00"><xsl:value-of select="wd:Annual_Salary"/></Salary>
```

Same — Java `DecimalFormat` patterns.

### Identifier targeting (WID, etc.)

```xml
<WorkerLink etv:target="WID"><xsl:value-of select="wd:Worker_WID"/></WorkerLink>
```

Used when the DT step needs to know an identifier is a WID (vs. Employee_ID, etc.). Typically appears with messages and hyperlinks in the integration UI.

### Validation against an Integration Map

```xml
<Country etv:enumeration="Country_Map"><xsl:value-of select="wd:Country_Code"/></Country>
```

Validates that the value exists in the named Integration System Map. Fails the row if not. Used heavily for code translation (Workday → vendor codes).

## XTT — converting XML to text

Declare: `xmlns:xtt="urn:com.workday/xtt"`

XTT attributes tell the DT step how to flatten XML into text.

### Row separator

```xml
<File xtt:separator="&#xA;">
  <Row>...</Row>
  <Row>...</Row>
</File>
```

Each child of `<File>` becomes a row, separated by newline. Use `&#xA;` (LF) or `&#xD;&#xA;` (CRLF) depending on vendor.

### Field separator

```xml
<Row xtt:separator=",">
  <Field>1001</Field>
  <Field>John</Field>
  <Field>Smith</Field>
</Row>
```

Children of `<Row>` joined by `,` → produces `1001,John,Smith`.

### CSV quoting

```xml
<Row xtt:separator="," xtt:quote="&quot;" xtt:quoteIfNecessary="true">
  ...
</Row>
```

`xtt:quoteIfNecessary="true"` quotes only fields that contain the separator, quote char, or newline. Standard CSV behavior.

### Fixed-width fields

```xml
<Field xtt:width="10" xtt:pad="right" xtt:padCharacter=" ">JOHN</Field>
```

Output: `JOHN      ` (4 chars + 6 spaces).

Use `xtt:pad="left"` to right-align (numeric fields):

```xml
<Field xtt:width="10" xtt:pad="left" xtt:padCharacter="0">1001</Field>
```

Output: `0000001001`

### Suppressing the element itself

```xml
<Header xtt:omitElement="true">
  <Field>EMP_ID</Field>
  <Field>NAME</Field>
</Header>
```

The `<Header>` wrapper disappears; only its content's text representation survives. Useful when you don't want a structural element to print itself.

### Combining XTT and ETV

You'll routinely use both in the same stylesheet. Common pattern: ETV truncates/pads at content level, XTT controls record/field formatting at structural level.

```xml
<Row xtt:separator=",">
  <EmpID etv:pad="left" etv:padCharacter="0" etv:width="10">
    <xsl:value-of select="wd:Employee_ID"/>
  </EmpID>
  <Name etv:truncate="true" etv:maxLength="30">
    <xsl:value-of select="wd:Legal_Name"/>
  </Name>
</Row>
```

## Gotchas

- **ETV/XTT attributes appear in your output unchanged.** That means the DT step isn't running. Check the integration system has a Document Transformation step *after* the XSLT.
- **Whitespace in `xtt:separator` matters.** `xtt:separator=" "` (space) and `xtt:separator=""` (empty) and `xtt:separator="&#x20;"` (entity-encoded space) all behave differently in practice — be explicit.
- **`etv:required="true"` blocks the row even if the rest is fine.** Use sparingly; for "warn but pass" you generally want validation rules outside ETV.
- **Don't put ETV/XTT attributes in the wrong namespace.** `etv:truncate` is correct; `truncate` alone (no prefix) does nothing — it's just a normal attribute, ignored by DT.
- **ETV operates on element *content*, not on attribute values.** You can't `etv:truncate` an attribute. Wrap the value in an element if you need ETV on it.
