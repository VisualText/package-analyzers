# Address Parser

An [NLP++](https://github.com/VisualText/nlp-engine) analyzer that finds and
decomposes postal addresses (USPS-style and simple international forms) inside
free-form text. It runs over a **whole document** — HTML pages, Markdown, plain
text — and handles addresses embedded in prose, in tables, and across multiple
lines (street on one line, `CITY ST ZIP` on the next).

For each address it reports the street number, street name/type/suffix, city,
state, postal code, country, and the address type, and emits the result as JSON.
Rural Route (RR), Highway Contract (HC), and PO Box addresses are recognized too.

## What it extracts

| Field        | Meaning                                       | Example |
|--------------|-----------------------------------------------|---------|
| `streetnum`  | house / street number                          | `123`   |
| `streetname` | street name                                    | `Main`  |
| `streettype` | street designator word                         | `Street`|
| `streetsuff` | normalized suffix code                          | `st`    |
| `state`      | state (resolved from abbreviation)             | `iowa`  |
| `pincode`    | postal / ZIP code                              | `50441-1902` |
| `country`    | country name (for international addresses)      | `Canada`|
| `type`       | `individual`, `HighwayContract`, `RuralRoute`, `post` | `individual` |
| `boxnum` / `hcnum` / `routenum` | box and route numbers for PO/HC/RR forms | `293 A` |

Example output:

```json
{
  "addresses": {
    "address": [
      {
        "id": "1",
        "streetnum": "123",
        "streetname": "Main",
        "streettype": "Street",
        "streetsuff": "st",
        "state": "Wyoming",
        "pincode": "64775",
        "type": "individual"
      },
      {
        "id": "2",
        "type": "HighwayContract",
        "hcnum": "72",
        "boxnum": "293 A",
        "state": "minnesota",
        "pincode": "55811-9702"
      }
    ]
  }
}
```

## How it works

The analyzer works on the **whole text** rather than treating each line as a
separate record. Unlike an email or a URL, an address has no single trigger
character, so detection is done in stages: the text is normalized, the postal
code and the address start are marked, and then the address span is grouped and
decomposed.

1. **Tokenize** (`dicttok`) and load the dictionaries (`KBFuncs`, `funcs`,
   `kbinit`).
2. **Build zones and normalize.** `Lines` wraps the text into `_ADDRESSZONE`
   regions, and within each zone the preprocessing passes run:
   - `RemoveWhiteSpace` collapses whitespace,
   - `pincode` marks the postal code (`_pincode`),
   - `RemoveSpecialChars` drops stray punctuation,
   - `PrecedingWords` marks where an address **starts** — a street number, a
     `PO`/`HC`/`RR` keyword, or a number following an address word
     (`address`, `located at`, `resides`, …; see `address-synonym.dict`).
3. **Group into addresses.** `removelines` flattens the zones and `Grouping`
   spans each address from its start marker through to its postal code, so a
   single record can cross what were originally several lines (the standard US
   `street ⏎ CITY ST ZIP` layout).
4. **Decompose** (`information1`–`information3`, `countryname`). Each `_address`
   is split into street parts, city (`FindCity`), state, postal code, and
   country; PO Box / Highway Contract / Rural Route forms get their box and
   route numbers.
5. **Collect and emit** (`kbmake`, `output`) the addresses as `output.json`.

> Note on the zone: `_ADDRESSZONE` is the preprocessing container (it replaced
> the old `_LINE`). Because the grouping step deliberately erases line
> boundaries to catch multi-line addresses, the real per-address unit is the
> `_address` node produced by `Grouping`.

### Dictionaries (`kb/user/`)

- `en-usa-states.dict` — state abbreviations → names (`IA` → iowa).
- `en-usa-streetsuff.dict` — street suffixes (`St`, `Ave`, `Rd`, `Lane`, …).
- `designator.dict` — unit designators (`apt`, `ste`, `bldg`, `fl`, …).
- `address-synonym.dict` — words that introduce an address (`located`,
  `resides`, `address`, …).
- `directions.dict` — `N`/`S`/`E`/`W`/`NW`/… direction words.
- `Country.dict` — country names. `military-address.dict` — APO/FPO forms.

## Directory layout

```
address-parser/
  spec/                 NLP++ passes + analyzer.seq (the pass order)
  kb/user/              dictionaries (states, suffixes, designators, ...)
  input/
    address-texts/      30 sample pages (HTML / Markdown / plain text)
    text.txt            small sample
  output/               build/debug logs
```

## Running the analyzer

```sh
nlp.exe -ANA <path>/address-parser -WORK <nlp-engine-dir> <input-file>
```

- `-ANA` is this analyzer folder.
- `-WORK` is the NLP++ engine directory (needs its `data/` tree).

The result is written next to the input file in `<input-file>_log/`:

- `output.json` — the structured addresses.
- `final.tree` — the full parse tree (useful for debugging).

To analyze your own text, drop a `.txt`, `.md`, or `.html` file anywhere
(e.g. in `input/`) and point the command at it.

## Test corpus

`input/address-texts/` contains 30 realistic pages — 10 HTML, 10 Markdown,
10 plain text — with addresses in contact pages, store locators, directories,
shipping labels, letters, and résumés, covering US `street, city, ST, ZIP+4`,
international `street, city, ZIP country`, PO Box, Highway Contract, and
multi-line layouts, plus pages with numeric noise.

## Known limitations

- The postal code is the main anchor, and it is matched loosely: any
  number-dash-number (e.g. a phone fragment `212-555` or a score `21-17`) and
  any 5–7-digit number (e.g. a SKU `778899`) can be taken as a `pincode`, so
  noise-heavy text can yield a few false addresses. Tightening `pincode` to a
  ZIP/ZIP+4 shape would remove these.
- City detection depends on `FindCity` and the surrounding tokens; unusual city
  formats may be missed or misattributed.
- Coverage targets USPS-style and simple `…, City, ZIP Country` forms; richly
  formatted international addresses are only partially parsed.
