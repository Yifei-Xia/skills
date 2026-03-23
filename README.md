# Semantic Column Matcher

A rule-based semantic column matching tool for Chinese and mixed-language text. Extracts text columns from two data sources, cross-matches them using a multi-signal similarity algorithm, and outputs a structured Excel report.

## Features

- **Multiple data sources** -- Local Excel (.xlsx), CSV, JSON, and any online spreadsheet URL (DingTalk/AliDocs, Feishu/Lark, Tencent Docs, Google Sheets, WPS, Notion, etc.)
- **Automatic deduplication** -- Order-preserving deduplication on source column; each unique value produces exactly one output row
- **Status filtering** -- Filter target rows by any column value (e.g. only match entries with status "Published")
- **Multi-signal matching** -- Combines Longest Common Substring (LCS), character N-gram Jaccard similarity, key entity overlap, and domain keyword co-occurrence
- **Zero external dependencies** -- Uses only Python standard library; no pip install required
- **Highly configurable** -- Column selection, header labels, matching thresholds, and domain rules are all customizable

## Directory Structure

```
semantic-column-matcher/
  SKILL.md                    # Skill workflow document
  matching-rules.md           # Detailed matching rules reference
  README.md                   # Chinese readme
  README_EN.md                # This file (English readme)
  scripts/
    semantic_match.py         # Core matching script
```

## Requirements

- **Python 3.8+** (standard library only, no third-party packages)

If Python is not installed, download the embeddable version:

```bash
# Windows
curl -o python-embed.zip "https://www.python.org/ftp/python/3.12.3/python-3.12.3-embed-amd64.zip"
```

After extracting, uncomment `import site` in the `python312._pth` file, then run the script with `python.exe`.

## Quick Start

### Basic Usage

```bash
python scripts/semantic_match.py \
  --source source_data.json \
  --target target_data.json \
  --output result.xlsx
```

### Full Example

```bash
python scripts/semantic_match.py \
  --source keywords.json \
  --target sheet1.xml \
  --target-col A \
  --status-col E \
  --status "Published" \
  --source-label "Keyword" \
  --target-label "Article Title" \
  --domain-rules custom_rules.json \
  --output match_result.xlsx
```

## CLI Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--source` | Yes | - | Source data file path (JSON / CSV / xlsx worksheet XML) |
| `--source-col` | No | `A` | Column to extract from source (letter for XML, 0-based index for CSV) |
| `--target` | Yes | - | Target data file path |
| `--target-col` | No | `A` | Column to match against in target |
| `--status-col` | No | None | Status column in target for row filtering |
| `--status` | No | None | Only include target rows where the status column equals this value |
| `--source-label` | No | `Source` | Column header for the source column in output Excel |
| `--target-label` | No | `Target` | Column header for the target column in output Excel |
| `--domain-rules` | No | None | Path to a JSON file with custom domain rules |
| `--output` | No | `result.xlsx` | Output Excel file path |

## Supported Input Formats

### JSON

Flat array of strings:

```json
["keyword1", "keyword2", "keyword3"]
```

Or an array of objects:

```json
[
  {"title": "Article Title A", "status": "Published"},
  {"title": "Article Title B", "status": "Draft"}
]
```

### CSV

Standard CSV format. The first row is treated as a header and automatically skipped. Use `--source-col` / `--target-col` to specify column indices (0-based).

### Excel (.xlsx)

Copy the `.xlsx` file to `.zip` and extract it. Use `xl/worksheets/sheet1.xml` from the extracted archive as the input file. Specify columns by letter (A, B, C, ...).

## Output Format

The output is a standard `.xlsx` file with four columns:

| Column | Description |
|--------|-------------|
| Source (customizable) | Deduplicated source values |
| Target (customizable) | Matched target values; multiple matches joined with semicolons |
| Is Related | "Yes" or "No" |
| Reason | Description of why the match was made; multiple reasons joined with semicolons; shows "No semantic match found" when unmatched |

## Matching Algorithm

The script uses a multi-signal weighted matching strategy, evaluated top-down by priority (first match wins):

| Priority | Signal | Condition | Score |
|----------|--------|-----------|-------|
| 1 | Exact match | Cleaned texts are identical | 1.0 |
| 2 | Containment | One text fully contains the other | 0.95 |
| 3 | Longest Common Substring (LCS) | LCS length >= 6 AND LCS ratio >= 0.4 | Weighted |
| 4 | Character bigram Jaccard | Jaccard >= 0.45 AND LCS >= 4 | Weighted |
| 5 | Character trigram Jaccard | Jaccard3 >= 0.35 AND LCS >= 4 | Weighted |
| 6 | Key entity overlap | Shared Chinese entity with length >= 3 AND LCS >= 3 | >= 0.5 |
| 7 | Domain keyword co-occurrence | Both texts contain keywords from the same domain group | Rule-defined |
| 8 | Person/entity name match | Same person name appears in both texts | 0.65 |

The algorithm follows a **first-hit** model -- signals are evaluated in priority order, and the first signal that meets its threshold determines the final result.

For detailed rule explanations, see [matching-rules.md](matching-rules.md).

## Customization

### Custom Domain Rules

Create a JSON file and pass it via the `--domain-rules` argument:

```json
{
  "domain_rules": [
    [["electric vehicle", "EV", "charging station"], "EV-related topic", 0.55, 3],
    [["housing", "real estate", "mortgage"], "Real estate topic", 0.6, 0]
  ],
  "person_names": ["John Doe", "Jane Smith"]
}
```

Each domain rule follows the format: `[keyword_list, reason_string, score, min_lcs_length]`

### Adjusting Thresholds

To make matching stricter or more lenient, edit the thresholds in the `semantic_match()` function in `scripts/semantic_match.py`:

```python
# Stricter: raise minimum LCS length
if lcs >= 8 and lcs_ratio >= 0.5:  # was: lcs >= 6, lcs_ratio >= 0.4

# More lenient: lower Jaccard threshold
if jaccard >= 0.35 and lcs >= 3:   # was: jaccard >= 0.45, lcs >= 4
```

### Modifying Default Rules

The script includes two built-in lists that can be edited directly for your use case:

- `DEFAULT_DOMAIN_RULES` -- Domain keyword co-occurrence rules
- `DEFAULT_PERSON_NAMES` -- Person/entity name list

## Online Document Support

When a data source is an online spreadsheet URL (invoked through the Qoder Skill workflow), the following platforms are supported:

- DingTalk / AliDocs
- Feishu / Lark
- Tencent Docs
- Google Sheets
- WPS Online
- Notion

**Login handling**: If the document requires authentication, the tool takes a screenshot of the login page and prompts the user to complete login manually (QR code scan or username/password). Data extraction proceeds automatically once authentication is confirmed.

## Usage Examples

### Example 1: Local JSON matched against Excel XML

```bash
# 1. Extract the Excel file
cp data.xlsx data.zip && unzip data.zip -d data_unzipped

# 2. Run matching
python scripts/semantic_match.py \
  --source keywords.json \
  --target data_unzipped/xl/worksheets/sheet1.xml \
  --target-col B \
  --output result.xlsx
```

### Example 2: CSV files with status filtering

```bash
python scripts/semantic_match.py \
  --source source.csv \
  --source-col 0 \
  --target target.csv \
  --target-col 1 \
  --status-col 4 \
  --status "Published" \
  --source-label "Search Term" \
  --target-label "Content Title" \
  --output filtered_result.xlsx
```

### Example 3: Using custom domain rules

```bash
python scripts/semantic_match.py \
  --source data1.json \
  --target data2.json \
  --domain-rules my_rules.json \
  --output custom_result.xlsx
```

## License

For internal use only.
