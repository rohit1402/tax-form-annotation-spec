# Tax Form Field Annotation Spec

## 1. Scope

This spec defines a data structure for describing where a value goes on a tax form, how it should look once it's printed, and which value from a nested data set it should pull. It covers:

- What value goes where on a form
- How that value should be formatted
- Which value to pull from a nested data set
- How to handle repeating structures, like multiple W-2s or a dependents table

This spec does not do the rendering itself. A separate application reads the annotation file and a data set, then draws the values onto the form. This document only defines the file format.

---

## 2. Form Annotation Object

An annotation file describes one form. A form is just a named list of fields.

```json
{
    "schemaVersion": "1.0",
    "formId": "w2-2026",
    "formVersion": "2026",
    "pageDetails": {
        "width": 612,
        "height": 792,
        "unit": "pt",
        "origin": "top-left",
    },
    "fields": [
        // list of fields on the form
    ]
}
```

| Property | Type | Description |
|---|---|---|
| `schemaVersion` | string | Version of the schema used for particular form |
| `formId` | string | Unique name for the form template (e.g. `w2-2026`, `1040-2026`) |
| `formVersion` | string | Tax year or revision, since forms can change |
| `pageDetails` | object | The details about pages |
| `fields` | array | List of Field Annotation objects, see section 3 |


**`PageDetails` object:**

| Property | Type | Description |
|---|---|---|
| `width` | number | Page width, in `unit` |
| `height` | number | Page height, in `unit` |
| `unit` | enum | Units `pt`, `in`, `mm`, `px` |
| `origin` | enum | `top-left` (default) or `bottom-left`|

---

## 3. Field Annotation Object

Every box on a form is one Field Annotation object, made up of four parts: identity, position, format, and a data reference.

```json
{
    "fieldId": "w2-box1",
    "label": "Box 1 - Description about box 1",
    "type": "currency",
    "position": {
        "page": 1,
        "x": 412.5,
        "y": 88.0,
        "width": 90.0,
        "height": 14.0
    },
    "format": {
        "style": "currency",
        "decimalPlaces": 2,
        "thousandsSeparator": true,
        "negativeStyle": "parentheses",
        "align": "right",
        "fontSize": 9
    },
    "dataRef": "$.taxpayer.employment.w2s[0].box1"
}
```

| Property | Type | Description |
|---|---|---|
| `fieldId` | string | Unique key for this box within the form |
| `label` | string | A human-readable description. Not rendered, just for anyone reading the file |
| `type` | enum | `text`, `currency`, `number`, `date`, `checkbox`, `checkboxGroup` (3.1), `repeatingGroup` (3.2) |
| `position` | object | See section 4 |
| `format` | object | See section 5 |
| `dataRef` | string | A path into the taxpayer data set, see section 6 |

`text`, `currency`, `number`, `date`, and `checkbox` all use the shape above as-is. Two field types, `checkboxGroup` and `repeatingGroup`, add a few extra properties on top of it. Both are covered below.

### 3.1 Checkbox Groups (`type: "checkboxGroup"`)

Use this when a group of checkboxes are mutually exclusive, like filing status on a 1040. The group looks up one value and marks whichever option matches it:

```json
{
    "fieldId": "filing-status",
    "type": "checkboxGroup",
    "dataRef": "$.taxpayer.filingStatus",
    "options": [
        {
            "matchValue": "single",
            "position": {
                "page": 1,
                "x": 50,
                "y": 120,
                "width": 10,
                "height": 10
            }
        },
        {
            "matchValue": "mfj",
            "position": {
                "page": 1,
                "x": 50,
                "y": 135,
                "width": 10,
                "height": 10
            }
        },
        {
            "matchValue": "mfs",
            "position": {
                "page": 1,
                "x": 50,
                "y": 150,
                "width": 10,
                "height": 10
            }
        }
    ],
    "format": {
        "checkedSymbol": "X"
    }
}
```

The renderer resolves `dataRef`, checks which option's `matchValue` matches that value, and draws `checkedSymbol` at that option's position only. The other options stay blank.

A `checkboxGroup` field adds one property on top of the base shape:

| Property | Type | Description |
|---|---|---|
| `options` | array of `CheckboxOption` | The list of mutually exclusive choices, see the shape below |

**`CheckboxOption` object:**

| Property | Type | Description |
|---|---|---|
| `matchValue` | string | Compared against the value `dataRef` resolves to. If they match, this option gets marked |
| `position` | object | See section 4. Each option has its own position, separate from the others |

A `checkboxGroup` field uses one `dataRef` and one `format`, shared across all options, since only one option is ever marked at a time. That's different from `repeatingGroup` below, where each column needs its own formatting.

### 3.2 Repeating Group fields (`type: "repeatingGroup"`)

Some boxes are really a template that repeats once per item in an array, like W-2 boxes 12a to 12d, or the dependents table on a 1040. Instead of writing one annotation per possible row, a `repeatingGroup` field defines the row once:

```json
{
    "fieldId": "w2-box12-codes",
    "type": "repeatingGroup",
    "dataRef": "$.taxpayer.employment.w2s[0].box12[*]",
    "maxInstances": 4,
    "rowHeight": 14.0,
    "position": {
        "page": 1,
        "x": 412.5,
        "y": 300.0,
        "width": 90.0,
        "height": 14.0
    },
    "template": [
        {
            "fieldId": "code",
            "dataRef": "code",
            "format": {
                "style": "plain"
            },
            "offsetX": 0,
            "offsetY": 0,
        },
        {
            "fieldId": "amount",
            "dataRef": "amount",
            "format": {
                "style": "currency"
            },
            "offsetX": 20,
            "offsetY": 0,
        }
    ]
}
```

The renderer resolves `dataRef` to an array. For each item, up to `maxInstances`, it draws the `template` fields at `position.y + (index * rowHeight)`, shifting each field horizontally by its `offsetX`. Inside `template`, `dataRef` is relative to the current array item, not the whole document.

A `repeatingGroup` field adds these properties on top of the base shape:

| Property | Type | Description |
|---|---|---|
| `maxInstances` | integer | The most rows the renderer will draw, even if the array has more items than that. Keeps things from overflowing the page |
| `rowHeight` | number | Vertical space between rows, added to `position.y` for each one, in the same `unit` as `pageDetails` |
| `template` | array of `TemplateField` | The fields drawn once per array item, see the shape below |

**`TemplateField` object:**

| Property | Type | Description |
|---|---|---|
| `fieldId` | string | Unique key for this field within the template |
| `dataRef` | string | Path resolved against the current array item, not the document root |
| `format` | object | See section 5 |
| `offsetX` | number | Horizontal shift from `position.x`, in the same `unit` as `pageDetails` |
| `offsetY` | number | Horizontal shift from `position.x`, in the same `unit` as `pageDetails` |

A `repeatingGroup` field doesn't use a top-level `format`. Each column inside `template` gets its own, since a code column and an amount column usually need different formatting.

---

## 4. Positioning

A box's position just needs a rectangle, a page number, and a coordinate convention.

| Property | Type | Description |
|---|---|---|
| `page` | integer | Page number the box is on, starting at 1 |
| `x`, `y` | number | Coordinates of the box's top-left corner |
| `width`, `height` | number | Box size |

---

## 5. Formatting

Formatting turns a raw value into display text before it gets drawn.

| Property | Type | Applies to | Description |
|---|---|---|---|
| `style` | enum | all | `plain`, `currency`, `percent`, `date`, `ssn`, `ein`, `checkmark` |
| `decimalPlaces` | integer | currency, number, percent | Digits after the decimal point |
| `thousandsSeparator` | boolean | currency, number | Adds commas |
| `negativeStyle` | enum | currency, number | `minus` (`-500.00`) or `parentheses` (`(500.00)`). Tax forms usually use parentheses |
| `dateFormat` | string | date | e.g. `MM/DD/YYYY` |
| `align` | enum | all | `left`, `right`, `center`. Numbers are usually right-aligned |
| `fontSize` | number | all | Point size. Small boxes often need a smaller default |
| `maxLines` | integer | text | For multi-line boxes like an employer's address. The renderer wraps or truncates as needed |
| `checkedSymbol` | string | checkbox | What gets drawn when a checkbox is true, e.g. `"X"` |

**How `percent`, `ssn`, and `ein` work:**

- **`percent`**: assumes the raw value is already a ratio (`0.075`, not `7.5`). The renderer multiplies by 100 and adds a `%` sign. `decimalPlaces` controls how many digits show up, so `decimalPlaces: 1` turns `0.075` into `7.5%`. Default is 0 decimal places.
- **`ssn`**: assumes the raw value is a plain 9-digit string or number, no dashes. The renderer adds them in the standard `XXX-XX-XXXX` pattern.
- **`ein`**: same idea, but with the `XX-XXXXXXX` pattern EINs use.

---

## 6. Referencing Nested Data

`dataRef` uses JSONPath to find a value inside the taxpayer's data. This spec sticks to standard JSONPath instead of inventing a new syntax, so any existing JSONPath library can read it.

### 6.1 Simple lookup

```
$.taxpayer.ssn
```

Resolves to a single value.

### 6.2 Indexed array lookup

```
$.taxpayer.employment.w2s[0].box1
```

Points at the first W-2's box 1. Use this when a field maps to one specific item in an array.

`checkboxGroup` and `repeatingGroup` fields also use `dataRef`, but they use it differently: a `checkboxGroup` resolves one scalar to compare against a list of options, and a `repeatingGroup` resolves an array to repeat a template over. Both are covered in sections 3.1 and 3.2, since the interesting part is how the field type uses the resolved value, not the path syntax itself.

### 6.3 Aggregation

Some boxes aren't a single stored value, they're a total computed across several items, like 1040 line 1a, which sums up box 1 from every W-2. For that, `dataRef` supports wrapping a wildcard path in an aggregate function:

```
sum($.taxpayer.employment.w2s[*].box1)
```

Supported functions are `sum()`, `count()`, and `first()`. That's kept intentionally small. Anything more complicated, like conditional logic or numbers that depend on other forms, should be computed upstream and stored as a plain field, then referenced normally as in 6.1.

---

## 7. Worked Examples

### Example A: W-2 Box 1 (currency, simple lookup)

```json
{
    "fieldId": "w2-box1-wages",
    "label": "Box 1 - Wages, tips, other compensation",
    "type": "currency",
    "position": {
        "page": 1,
        "x": 412.5,
        "y": 88.0,
        "width": 90.0,
        "height": 14.0
    },
    "format": {
        "style": "currency",
        "decimalPlaces": 2,
        "thousandsSeparator": true,
        "negativeStyle": "parentheses",
        "align": "right",
        "fontSize": 9
    },
    "dataRef": "$.taxpayer.employment.w2s[0].box1"
}
```

### Example B: W-2 Box 13 (checkbox, not a group)

Box 13 actually has three independent yes/no checkboxes (statutory employee, retirement plan, third-party sick pay). They're not mutually exclusive, so each one is its own `checkbox` field rather than a `checkboxGroup`:

```json
{
    "fieldId": "w2-box13-retirement-plan",
    "label": "Box 13 - Retirement plan",
    "type": "checkbox",
    "position": {
        "page": 1,
        "x": 300,
        "y": 250,
        "width": 10,
        "height": 10
    },
    "format": {
        "style": "checkmark",
        "checkedSymbol": "X"
    },
    "dataRef": "$.taxpayer.employment.w2s[0].retirementPlan"
}
```

### Example C: 1040 Line 1a (aggregation)

```json
{
    "fieldId": "1040-line1a-wages",
    "label": "Line 1a - Total wages from Form(s) W-2, box 1",
    "type": "currency",
    "position": {
        "page": 1,
        "x": 480,
        "y": 400,
        "width": 90,
        "height": 14
    },
    "format": {
        "style": "currency",
        "decimalPlaces": 2,
        "thousandsSeparator": true,
        "negativeStyle": "parentheses",
        "align": "right",
        "fontSize": 9
    },
    "dataRef": "sum($.taxpayer.employment.w2s[*].box1)"
}
```