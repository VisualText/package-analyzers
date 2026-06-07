# TELEPHONE ANALYZER

This is an NLP++ Analyzer that parses through text and identifies telephone
numbers, covering North American (NANP) and international formats. Sample
inputs are in `input/text.txt` and a broad set of variations is in
`input/phone_variations.txt`.

## Formats recognized

**North American (NANP)** — with any common separators (`-`, `.`, space, `/`),
optional parentheses around the area code, optional leading `1`/`+1`, and
international dialing prefixes:

- `2124567890`, `212-456-7890`, `212.456.7890`, `212 456 7890`
- `(212) 456-7890`, `(212)-456-7890`
- `1-212-456-7890`, `+1-212-456-7890`, `+12124567890`
- `001-212-456-7890`, `191-212-456-7890` (intl dialing prefix)
- `456-7890` (7-digit local, no area code)

**International** — `+<country code> <national number>`, both spaced/separated
and concatenated:

- `+44 20 1234 1234`, `+33 1 23 45 67 89`, `+370 601 12345`, `+212-456-7890`
- `+442012341234`, `+37060112345` (concatenated — split via the country-code KB)

## Information extracted

For each number identified, some or all of the following are extracted
(depending on the format):

| Field | Meaning |
|-------|---------|
| `text` | the matched text |
| `country_code` | numeric dialing code (e.g. `1`, `44`, `370`) |
| `country` | country name(s) for the dialing code (international numbers) |
| `area` | NANP area code |
| `prefix` | NANP exchange / prefix |
| `station` | NANP station / line number |
| `national_number` | the national portion of an international number |
| `type` | `mobile`, `landline`, or `international` |

Output is written to `<input-file>_log/output.json`.

## How it works

The analyzer is a cascade of passes (`spec/analyzer.seq`), ordered so that the
longest / most-specific patterns are matched first and shorter patterns pick up
the leftovers:

1. **Telep0** — country/intl-prefixed NANP (`1-`, `+1-`, `001-`, `191-` + area/prefix/station)
2. **TelepIntl** — international `+CC national` (spaced/separated)
3. **TelepIntlCat** — concatenated international (`+442012341234`), split using the country-code KB
4. **Telep1** — core NANP 3-group / leading-symbol / full-digit patterns
5. **Telep1b** — 7-digit local fallback (`456-7890`)
6. **Telep2** — collects each match into the `telephones` knowledge base concept

Country codes live in the knowledge base at `kb/user/tel-country-codes.kbb`
(concept `codes`), which is auto-loaded at startup. The concatenated-number
splitter (`IntlCountryCode` / `CountryName` in `spec/funcs.nlp`) does a
longest-prefix lookup against it.

## Running

Create a text file in the `input` folder with the text to parse, then run the
analyzer over it. Results land in `input/<your-file>.txt_log/output.json`.
