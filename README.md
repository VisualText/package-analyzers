# Package Analyzers

A collection of [NLP++](https://github.com/VisualText/nlp-engine) analyzers that
extract structured information from free-form text. Each analyzer scans a
document — HTML, Markdown, or plain text — and emits JSON describing what it
found (emails, phone numbers, links, addresses, …).

These are the analyzers packaged for and dispatched to
[VisualText/py-package-nlpengine](https://github.com/VisualText/py-package-nlpengine).

## Analyzers

| Analyzer | Extracts | Output (key fields) |
|----------|----------|---------------------|
| [emailaddress](emailaddress/README.md) | email addresses (incl. `at`/`dot` obfuscation) | `local`, `subdomain`, `domainname`, `tld`, `cd`, `country` |
| [telephone](telephone/README.md) | phone numbers (NANP + international) | `area`, `prefix`, `station`, `country_code`, `national_number`, `country` |
| [links](links/README.md) | web links / URLs | `linktext`, `scheme`, `subdomain`, `domain`, `tld`, `pagepath` |
| [address-parser](address-parser/README.md) | postal addresses (USPS + simple international) | `streetnum`, `streetname`, `streettype`, `state`, `pincode`, `country`, `type` |
| [parse-en-us](https://github.com/VisualText/parse-en-us) | full USA-English syntactic parse | (git submodule) |

## Zone-based architecture

The `emailaddress`, `telephone`, `links`, and `address-parser` analyzers work on
the **whole document** rather than one candidate per line. An early pass carves
out a small *zone* around each candidate — `_EMAILZONE`, `_TELEPHONEZONE`,
`_LINKZONE`, `_ADDRESSZONE` — and the extraction rules run inside those zones.
This lets them handle candidates embedded in prose, tables, `mailto:`/`tel:`
markup, Markdown links, and multi-line addresses.

Where a candidate has a natural anchor, the zone rule keys off it with the
`[trig]` element flag for fast matching:

- **emailaddress** — the `@` symbol, and the spelled-out `at` word.
- **telephone** — a digit group.
- **links** — a top-level domain following a `.`.
- **address-parser** — no single-character anchor; the address span is grouped
  from a start marker through to its postal code.

> NLP++ note: zone node names have no internal underscore (`_EMAILZONE`, not
> `_EMAIL_ZONE`) because the rule-reduce parser splits an identifier at an
> internal `_`.

## Layout of an analyzer

```
<analyzer>/
  spec/                 NLP++ passes + analyzer.seq (the pass order)
  kb/user/              dictionaries and knowledge-base files
  input/
    <name>-texts/       sample pages (HTML / Markdown / plain text)
    text.txt            small sample
  output/               build/debug logs
  README.md
```

Each analyzer ships a 30-page test corpus under `input/<name>-texts/`
(10 HTML + 10 Markdown + 10 plain text), including deliberate decoys.

## Running an analyzer

```sh
nlp.exe -ANA <repo>/<analyzer> -WORK <nlp-engine-dir> <input-file>
```

- `-ANA` — the analyzer folder (e.g. `emailaddress`).
- `-WORK` — the NLP++ engine directory (needs its `data/` tree). On Windows the
  engine bundled with the VS Code NLP++ extension works well.

The result is written next to the input file in `<input-file>_log/`:

- `output.json` — the structured result.
- `final.tree` — the full parse tree (useful for debugging).

To analyze your own text, drop a `.txt`, `.md`, or `.html` file anywhere and
point the command at it. See each analyzer's README for its fields and
limitations.

## Submodules

`parse-en-us` is a git submodule. After cloning:

```sh
git submodule update --init --recursive
```

## Automation

GitHub Actions in `.github/workflows/` tag/release on pushes to `main` and
dispatch analyzer updates to the downstream package. Open changes as a pull
request rather than pushing analyzer edits directly to `main`.
