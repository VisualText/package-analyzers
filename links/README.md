# Hyperlinks (Links) Analyzer

An [NLP++](https://github.com/VisualText/nlp-engine) analyzer that finds and
decomposes web links / URLs inside free-form text. It runs over a **whole
document** — HTML pages, Markdown, plain text — not one candidate per line, so
several links on the same line (e.g. two Markdown links in a sentence) are each
extracted separately.

For every link it reports the scheme, host breakdown (subdomain / domain /
TLD, with country for ccTLDs), and the page path, and emits the result as JSON.

## What it extracts

| Field       | Meaning                                   | Example (`https://en.wikipedia.org/wiki/Machine_learning`) |
|-------------|-------------------------------------------|------------------------------------------------------------|
| `linktext`  | the full URL as found                     | `https://en.wikipedia.org/wiki/Machine_learning`           |
| `scheme`    | protocol (resolved name, from the dict)   | `SecureHypertextTransferProtocol`                          |
| `subdomain` | label(s) below the registrable domain     | `en`                                                       |
| `domain`    | the registrable domain label              | `wikipedia`                                                |
| `tld`       | top-level (or second-level) domain         | `org`                                                      |
| `country`   | country name, for ccTLD links              | (set for `.uk`, `.jp`, ...)                                |
| `pagepath`  | path / query after the host               | `wiki/Machine_learning`                                    |

Example output:

```json
{
  "Links": {
    "link": [
      {
        "id": "1",
        "linktext": "https://en.wikipedia.org/wiki/Machine_learning",
        "scheme": "SecureHypertextTransferProtocol",
        "tld": "org",
        "subdomain": "en",
        "domain": "wikipedia",
        "pagepath": "wiki/Machine_learning"
      },
      {
        "id": "2",
        "linktext": "ftp://ftp.example.com/files/file.zip",
        "scheme": "FileTransferProtocol",
        "tld": "com",
        "subdomain": "ftp",
        "domain": "example",
        "pagepath": "files/file.zip"
      }
    ]
  }
}
```

## How it works

The analyzer is **zone-based**. Instead of scanning the whole document with
the link-matching rules, it first carves out a small `_LINKZONE` region around
each URL, then decomposes only inside those zones.

1. **Tokenize** (`dicttok`) and load the dictionaries (`kbinit`, `funcs`).
2. **Build zones** (`LinkZone`). A URL is a contiguous run of URL characters
   with no single special anchor — but its top-level domain is always preceded
   by a `.`. So the rule **triggers on a TLD token** (a label tagged in
   `domain.dict` or `country.dict`) that immediately follows a `.`, then grows
   the zone **outward** over URL characters: left across the host and scheme,
   right across the path and query string. Whitespace and the wrapper
   delimiters that are not URL characters (`< > " ( ) [ ]`) bound the run, so
   Markdown `[text](url)` and HTML `href="url"` yield a clean URL.

   The TLD element carries the `[trig]` flag so the engine only attempts the
   rule where a TLD occurs (fast matching). Requiring the `.` right before the
   TLD is what keeps non-links out: file extensions (`.docx`, `.png`, `.gz`),
   abbreviations (`e.g.`, `i.e.`), IP addresses, version numbers, dates, and
   ratios never trigger a zone — only real TLDs (`com`, `org`, `io`, `co.uk`, …)
   do.
3. **Decompose inside each zone.**
   - `Links` finds the `.TLD` (`_domain`) sub-nodes and flags the zone.
   - `Link1` renames a flagged `_LINKZONE` to `_conlink`.
   - `Link2`–`Link7` split the URL into scheme, subdomain, domain, TLD/ccTLD,
     and page path, trim trailing punctuation, and collect each link into the
     `Links` knowledge-base concept.
4. **Emit** (`output`) the links as `output.json`.

### Dictionaries (`kb/user/`)

- `schemelist.dict` — scheme → human-readable name (`https` →
  SecureHypertextTransferProtocol, `ftp` → FileTransferProtocol, …).
- `domain.dict` — known generic TLDs (`com`, `org`, `app`, `io`, `aero`, …).
- `country.dict` — ccTLD → country name (`uk` → United Kingdom, `jp` → Japan, …).

## Directory layout

```
links/
  spec/                 NLP++ passes + analyzer.seq (the pass order)
  kb/user/              dictionaries (schemes, domains, countries)
  input/
    links-texts/        30 sample pages (HTML / Markdown / plain text)
    text.txt            small sample
  output/               build/debug logs
```

## Running the analyzer

```sh
nlp.exe -ANA <path>/links -WORK <nlp-engine-dir> <input-file>
```

- `-ANA` is this analyzer folder.
- `-WORK` is the NLP++ engine directory (needs its `data/` tree).

The result is written next to the input file in `<input-file>_log/`:

- `output.json` — the structured links.
- `final.tree` — the full parse tree (useful for debugging zones).

To analyze your own text, drop a `.txt`, `.md`, or `.html` file anywhere
(e.g. in `input/`) and point the command at it.

## Test corpus

`input/links-texts/` contains 30 realistic pages — 10 HTML, 10 Markdown,
10 plain text — with links in `href` attributes, Markdown `[text](url)` and
`<url>` forms, prose, tables, signatures, and bookmark lists, covering
`https`/`http`/`ftp`, `www`, bare domains (`bit.ly`), ccTLDs, query strings,
and fragments, plus deliberate non-link noise (file names, IPs, versions,
abbreviations) that must **not** be matched. A full run extracts ~247 links and
rejects the noise.

## Known limitations

- A bare `www.host.com` (no scheme) can drop the `www` from `linktext` in the
  host decomposition.
- A URL that legitimately contains parentheses (e.g. a Wikipedia
  `..._(disambiguation)` path) is truncated at the parenthesis, because `( )`
  are treated as zone boundaries so that Markdown `(url)` wrappers stay out of
  the link.
- `.io` / `.co` / `.ly` resolve as their literal ccTLDs (British Indian Ocean
  Territory / Colombia / Libya) per the country dictionary.
