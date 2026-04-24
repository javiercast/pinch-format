# The `.pinch` Format Specification

**Version:** 1.0  
**Status:** Draft  
**Author:** TokenPinch (tokenpinch.com)  
**Created:** April 2025  
**License:** MIT

---

## What is `.pinch`?

`.pinch` is a token-efficient text format for transmitting structured data to Large Language Models (LLMs). It reduces token consumption by 40–75% compared to raw CSV, JSON, XML, Word, or PDF content — without modifying or losing any data.

The key insight: when you paste a file into an AI chat, most tokens go to formatting overhead — repeated column names, verbose XML namespaces, excessive whitespace, and structural syntax that the AI doesn't need. `.pinch` removes that overhead while preserving all the actual information.

---

## Design Principles

1. **Self-describing** — every `.pinch` file contains a header that tells any LLM how to decode it. No system prompt changes or native platform support required.
2. **Lossless** — no data is modified, truncated, or removed. Only structural overhead is compressed.
3. **Human-readable** — `.pinch` files are plain text and can be opened in any text editor.
4. **LLM-agnostic** — works with Claude, ChatGPT, Gemini, Copilot, and any other LLM that accepts text input.
5. **Format-aware** — different source formats use different compression strategies, chosen automatically based on file type and structure.

---

## File Format

### MIME Type

```
application/x-tokenpinch
```

*(Pending IANA registration as `application/vnd.tokenpinch`)*

### File Extension

`.pinch`

### Encoding

UTF-8 plain text.

---

## Structure

Every `.pinch` file has the following structure:

```
[AIX-FORMAT v1.0]
<metadata block>
[/AIX-FORMAT]

[ALIASES]          ← optional, only present when aliasing was applied
<alias map>
[/ALIASES]

[DATA]
<compressed content>
[/DATA]
```

---

## The `[AIX-FORMAT]` Header

The header is mandatory and must appear at the top of the file. It tells the LLM what the file contains and how to decode it.

### Syntax

```
[AIX-FORMAT v1.0]
Source: <filename> | <optional metadata fields>
Decode: <human-readable decode instruction>
Generated-by: TokenPinch (tokenpinch.com)
[/AIX-FORMAT]
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `Source` | Yes | Original filename. May include `Sheet:`, `Rows:`, `Strategy:` |
| `Decode` | Yes | Plain English instruction for the LLM |
| `Generated-by` | Yes | Tool that generated the file |

### Examples

**Tabular file (CSV/Excel/JSON):**
```
[AIX-FORMAT v1.0]
Source: sales_data.csv | Rows: 120 | Strategy: TOON tabular + header aliasing
Decode: expand [ALIASES] to reconstruct column names, then parse [DATA] rows in header order.
Generated-by: TokenPinch (tokenpinch.com)
[/AIX-FORMAT]
```

**XML file:**
```
[AIX-FORMAT v1.0]
Source: api_response.xml | Strategy: XML namespace stripping
Removed: 8 namespace declarations | Original tags: ~47
Decode: standard XML — all namespace prefixes removed, content and attributes preserved.
Generated-by: TokenPinch (tokenpinch.com)
[/AIX-FORMAT]
```

**Text file (Word/PDF):**
```
[AIX-FORMAT v1.0]
Source: report.pdf | Strategy: PDF text normalization
Decode: plain text — whitespace normalized, content fully preserved.
Generated-by: TokenPinch (tokenpinch.com)
[/AIX-FORMAT]
```

---

## Compression Strategies

### Strategy 1: TOON Tabular + Header Aliasing

Used for: CSV, Excel (.xlsx/.xls), JSON arrays

**How it works:**

Column headers are declared once at the top with short aliases. Data rows use aliases instead of full names. This eliminates repetition of verbose header names across every row.

**Alias generation rules (priority order):**
1. Initials of all words: `transaction_date` → `td`, `customer_full_name` → `cfn`
2. First 2 chars of word 1 + first char of word 2: `product_name` → `prn`
3. First 2 characters: `status` → `st`
4. Fallback — first letter + incrementing number: `a1`, `a2`

Aliases are guaranteed unique within a file.

**`[ALIASES]` block syntax:**
```
[ALIASES]
full_header_name:alias | full_header_name:alias | ...
[/ALIASES]
```

**`[DATA]` block syntax:**
```
[DATA]
<label>[<row_count>]{<alias1>,<alias2>,...}:
value1,value2,...
value1,value2,...
[/DATA]
```

Where `<label>` is the filename (without extension) or sheet name for multi-sheet workbooks.

**Full example:**

Source CSV:
```csv
transaction_date,customer_id,product_name,unit_price,quantity
2024-01-15,CUST-001,Laptop Pro,999.99,2
2024-01-16,CUST-002,Wireless Mouse,29.99,5
```

Compressed `.pinch`:
```
[AIX-FORMAT v1.0]
Source: sales.csv | Rows: 2 | Strategy: TOON tabular + header aliasing
Decode: expand [ALIASES] to reconstruct column names, then parse [DATA] rows in header order.
Generated-by: TokenPinch (tokenpinch.com)
[/AIX-FORMAT]

[ALIASES]
transaction_date:td | customer_id:ci | product_name:pn | unit_price:up | quantity:q
[/ALIASES]

[DATA]
sales[2]{td,ci,pn,up,q}:
2024-01-15,CUST-001,Laptop Pro,999.99,2
2024-01-16,CUST-002,Wireless Mouse,29.99,5
[/DATA]
```

**Multi-sheet workbooks:**

Each sheet produces its own `[DATA]` block, separated by `---`:
```
[DATA]
Sheet1[100]{...}:
...rows...
[/DATA]

---

[DATA]
Sheet2[50]{...}:
...rows...
[/DATA]
```

---

### Strategy 2: XML Namespace Stripping

Used for: XML files, SOAP responses, enterprise API payloads

**How it works:**

XML namespace declarations (`xmlns:ns0="..."`) and namespace prefixes (`ns0:ElementName`) are removed. The XML structure, attributes, and content are fully preserved. Empty elements and redundant whitespace are also removed.

**Transformations applied:**
1. Remove `<?xml ... ?>` declaration
2. Remove all `xmlns:*` and `xmlns` attribute declarations
3. Remove namespace prefixes from element names: `<ns0:Response>` → `<Response>`
4. Remove namespace prefixes from attributes: `ns1:ID="X"` → `ID="X"`
5. Remove empty elements: `<Element/>` and `<Element></Element>`
6. Collapse 3+ consecutive blank lines to 2
7. Remove lines containing only whitespace

**Example:**

Source XML (47 tokens for namespace declarations alone):
```xml
<ns0:Response xmlns:ns0="http://schemas.example.com/data"
              xmlns:ns1="http://schemas.example.com/common">
  <ns1:Status>confirmed</ns1:Status>
  <ns0:Record ns1:ID="REC-001">
    <ns1:Name>John Doe</ns1:Name>
  </ns0:Record>
</ns0:Response>
```

Compressed `.pinch`:
```
[AIX-FORMAT v1.0]
Source: response.xml | Strategy: XML namespace stripping
Removed: 2 namespace declarations | Original tags: ~6
Decode: standard XML — all namespace prefixes removed, content and attributes preserved.
Generated-by: TokenPinch (tokenpinch.com)
[/AIX-FORMAT]

[DATA]
<Response>
  <Status>confirmed</Status>
  <Record ID="REC-001">
    <Name>John Doe</Name>
  </Record>
</Response>
[/DATA]
```

---

### Strategy 3: Text Normalization

Used for: Word documents (.docx), PDF files

**How it works:**

Plain text is extracted from binary formats and whitespace is normalized. No content is removed or modified.

**Transformations applied:**
1. Normalize line endings to `\n`
2. Remove trailing spaces and tabs from each line
3. Collapse 3+ consecutive blank lines to 2
4. Collapse 2+ consecutive spaces/tabs to a single space

**PDF-specific:** Text is extracted page by page. Each page is prefixed with `[Page N]`.

**Word-specific:** Text is extracted using mammoth.js. Images are ignored; text content is fully preserved.

---

## No-Compression Rule

If the compressed output would consume more tokens than the original file, the `.pinch` conversion is skipped. The original file should be used directly.

Token estimation formula used by TokenPinch: `tokens ≈ characters / 3.8`

This is an approximation. Actual token counts vary by model and tokenizer.

---

## Decoder Instructions for LLMs

When an LLM receives a `.pinch` file, it should:

1. Read the `[AIX-FORMAT]` header to understand the strategy used
2. If `[ALIASES]` is present, build a map of `alias → full_name`
3. Parse the `[DATA]` block according to the strategy:
   - **TOON tabular:** read the header row `label[N]{aliases}:`, then parse N comma-separated rows using the alias map to reconstruct full column names
   - **XML:** parse as standard XML
   - **Text:** read as plain text, treating `[Page N]` markers as page separators if present
4. Respond as if the original uncompressed file had been provided

---

## Implementing `.pinch` in Other Languages

To implement a `.pinch` encoder in any language, you need to:

1. **Detect** the source format (CSV, JSON array, XML, text)
2. **Apply** the appropriate strategy
3. **Generate** the `[AIX-FORMAT]` header with correct metadata
4. **Output** UTF-8 plain text

The reference implementation is available at [tokenpinch.com](https://tokenpinch.com) (client-side JavaScript, no server required).

### Minimal Python example (TOON tabular)

```python
import csv
import io

def build_aliases(headers):
    used = set()
    aliases = []
    for h in headers:
        words = h.lower().replace('_', ' ').replace('-', ' ').split()
        alias = None
        # Try initials
        initials = ''.join(w[0] for w in words if w)[:3]
        if initials and initials not in used:
            alias = initials
        # Fallback
        if not alias:
            base = h.lower().replace('_', '')[:1] or 'c'
            n = 1
            while f'{base}{n}' in used:
                n += 1
            alias = f'{base}{n}'
        used.add(alias)
        aliases.append((h, alias))
    return aliases

def to_pinch(csv_text, filename):
    reader = csv.reader(io.StringIO(csv_text))
    rows = list(reader)
    if len(rows) < 2:
        return csv_text  # too small to compress
    
    headers = rows[0]
    data_rows = rows[1:]
    aliases = build_aliases(headers)
    
    alias_map = ' | '.join(f'{f}:{a}' for f, a in aliases)
    data_label = filename.rsplit('.', 1)[0]
    alias_header = ','.join(a for _, a in aliases)
    data_lines = '\n'.join(','.join(row) for row in data_rows)
    
    return f"""[AIX-FORMAT v1.0]
Source: {filename} | Rows: {len(data_rows)} | Strategy: TOON tabular + header aliasing
Decode: expand [ALIASES] to reconstruct column names, then parse [DATA] rows in header order.
Generated-by: custom implementation
[/AIX-FORMAT]

[ALIASES]
{alias_map}
[/ALIASES]

[DATA]
{data_label}[{len(data_rows)}]{{{alias_header}}}:
{data_lines}
[/DATA]"""
```

---

## Versioning

The current version is `v1.0`, declared in the `[AIX-FORMAT v1.0]` header tag.

Future versions will increment the version number. Decoders should check the version field and handle unknown versions gracefully.

---

## Contributing

This specification is open. If you implement `.pinch` in another language or find issues with the spec, open an issue or pull request at the repository.

---

## License

MIT License. You are free to implement, use, and distribute this format in any project, commercial or otherwise, with attribution.

## Disclaimer

This specification describes a structural compression format for reducing token overhead when submitting files to Large Language Model platforms.

The `.pinch` format is designed to remove formatting overhead only — repeated headers, XML namespaces, and unnecessary whitespace. It is not intended to summarize, interpret, or alter the semantic content of files.

**This specification is provided "as is" without warranties of any kind.** Implementations based on this specification are the sole responsibility of their authors. The TokenPinch project and its contributors are not liable for:

- Errors or omissions in AI outputs resulting from use of this format
- Data loss or corruption resulting from incorrect implementations
- Use of this format with sensitive, regulated, or legally critical data

**Do not use this format to process** credit card numbers, passwords, medical records, confidential legal documents, or any regulated data without appropriate safeguards.

Always verify AI outputs before using them in any critical decision.

---

MIT License — see [LICENSE](LICENSE) for details.  
For questions: [hello@tokenpinch.com](mailto:hello@tokenpinch.com)

---

*TokenPinch — compress files for AI, save tokens, save money.*  
*tokenpinch.com*
