# Email Address Analyzer

An [NLP++](https://github.com/VisualText/nlp-engine) analyzer that finds and
decomposes email addresses inside free-form text. It runs over a **whole
document** — HTML pages, Markdown, plain text, signatures, forum posts — not
one candidate per line, and it understands both ordinary `local@host`
addresses and the scraper-dodging `name at example dot com` style.

For every address it extracts the local part, any subdomain, the registrable
domain, the top-level domain, and (for country-code TLDs) the country, and
emits the result as JSON.

## What it extracts

| Field        | Meaning                                  | Example (`press@media.acme.co.uk`) |
|--------------|------------------------------------------|------------------------------------|
| `local`      | the part before the `@`                  | `press`                            |
| `subdomain`  | label(s) below the registrable domain    | `media`                            |
| `domainname` | the registrable domain label             | `acme`                             |
| `tld`        | top-level (or second-level) domain        | `co`                               |
| `cd`         | country code, for ccTLD addresses         | `uk`                               |
| `country`    | country name resolved from the ccTLD      | `UnitedKingdom`                    |

Example output (`info@acme.com` and `press@media.acme.co.uk`):

```json
{
  "Emailaddrs": {
    "email": [
      { "id": "1", "local": "info", "domainname": "acme", "tld": "com" },
      { "id": "2", "local": "press", "subdomain": "media",
        "domainname": "acme", "tld": "co", "cd": "uk",
        "country": "UnitedKingdom" }
    ]
  }
}
```

## How it works

The analyzer is **zone-based**. Instead of scanning the whole document with
the email-matching rules, it first carves out small `_EMAILZONE` regions
around the spots where an address can occur, then runs the decomposition rules
only inside those zones.

1. **Tokenize** (`dicttok`) and load the dictionaries (`kbinit`, `funcs`).
2. **Build zones around triggers.** Two passes mark `_EMAILZONE` regions:
   - `EmailZone` — anchored on a literal **`@`**. The zone is the maximal run
     of email-legal characters on either side of the `@`, so it captures the
     whole `local@host` and stops at whitespace or delimiters (it even strips a
     leading `mailto:`).
   - `EmailZoneAt` — anchored on a spelled-out **`at`** word. It grabs a
     same-line window around the trigger so an obfuscated address like
     `john at example dot com` is fully contained.

   Each trigger element is flagged with `[trig]` so the engine indexes the
   rule on the `@`/`at` token and only attempts it where a trigger actually
   occurs — fast matching instead of scanning every position.
3. **Decompose inside each zone.**
   - `email0` handles `local@host`: it walks the host label-by-label and splits
     it into subdomain / domain / tld / ccTLD, using the dictionary tags on
     each label. It handles any number of subdomains and multi-level public
     suffixes (`co.uk`, `edu.uk`, …).
   - `email0at` handles the `at`/`dot` obfuscated form, walking outward from
     the `at` to recover the local part and the domain.
   - `email1`–`email6` are the legacy fall-through rules for cases the front
     passes leave behind.
4. **Collect and emit** (`email7`, `output`) the addresses into the
   `Emailaddrs` concept and write `output.json`.

### Dictionaries (`kb/user/`)

- `charactders.dict` — tags every character that is legal in an email with
  `ec1` (local part) and `ec2` (domain part). The zone passes use these tags
  (`_xVAR("ec1")`) to decide what belongs in an address, so the rules don't
  hard-code punctuation.
- `country.dict` — ccTLD → country name (e.g. `uk` → United Kingdom).
- `domain.dict` — known generic TLDs (`com`, `org`, `app`, `io`, `travel`, …).

## Directory layout

```
emailaddress/
  spec/                 NLP++ passes + analyzer.seq (the pass order)
  kb/user/              dictionaries (chars, country, domain)
  input/
    email_texts/        30 sample pages (HTML / Markdown / plain text)
    email_variations.txt   line-by-line corpus of edge cases
    text.txt            small sample
  output/               build/debug logs
```

## Running the analyzer

```sh
nlp.exe -ANA <path>/emailaddress -WORK <nlp-engine-dir> <input-file>
```

- `-ANA` is this analyzer folder.
- `-WORK` is the NLP++ engine directory (needs its `data/` tree).

The result is written next to the input file in `<input-file>_log/`:

- `output.json` — the structured addresses.
- `final.tree` — the full parse tree (useful for debugging zones/tags).

To analyze your own text, drop a `.txt`, `.md`, or `.html` file anywhere
(e.g. in `input/`) and point the command at it.

## Test corpus

`input/email_texts/` contains 30 realistic pages — 10 HTML, 10 Markdown,
10 plain text — with emails embedded in `mailto:` links, tables, prose,
signatures, and `at`/`dot` obfuscation, plus deliberate decoys
(`plainaddress`, `@no-local.com`, `user@localhost`, social `@handles`) that
must **not** be matched. A full run extracts ~216 addresses and rejects the
decoys.

## Known limitations

- Addresses on the reserved `.example` TLD don't resolve, because `.example`
  is intentionally absent from `domain.dict`.
- A `mailto:` link plus its visible link text yields the same address twice
  (both are real occurrences in the page).
- Bracketed/parenthesized obfuscation (`john [at] example [dot] com`,
  `jane (at) example (dot) net`) is not yet decomposed; only the bare
  `at`/`dot` word form is.
- `.io` / `.co` / `.me` resolve as their literal ccTLDs (British Indian Ocean
  Territory / Colombia / Montenegro) per the country dictionary.
