---
name: semantic-column-matcher
description: Perform semantic matching between any two columns from two data sources. Supports any online spreadsheet URL (DingTalk, AliDocs, Feishu, Tencent Docs, Google Sheets, WPS, Notion, etc.) and local files (Excel/CSV/JSON). If login is required, prompts user to scan QR or log in manually. Deduplicates source column, optionally filters target by a status column, matches using character N-gram similarity, longest common substring, and entity overlap, then outputs an Excel report. Use when the user wants to cross-match, semantically associate, or correlate any two lists of text across spreadsheets or online documents.
---

# Semantic Column Matcher

Cross-match text values from **Document 1 Column X** against **Document 2 Column Y**, using rule-based Chinese/mixed-language semantic similarity, and output a structured Excel report.

Supports any online spreadsheet link and local files. If a document requires login, the user will be prompted to complete authentication (e.g. scan QR code) before data extraction proceeds.

## Workflow

```
Task Progress:
- [ ] Step 1: Clarify user requirements (which columns, filter conditions)
- [ ] Step 2: Extract source list from Document 1
- [ ] Step 3: Extract target list from Document 2
- [ ] Step 4: Deduplicate source list
- [ ] Step 5: Filter target list by condition if requested
- [ ] Step 6: Run semantic matching
- [ ] Step 7: Generate output Excel
```

### Step 1: Clarify Requirements

Ask the user:
- **Document 1**: path/URL, which column to extract (default: A)
- **Document 2**: path/URL, which column to match against (default: A)
- **Filter**: whether to filter Document 2 by a status column (e.g. column E = "已发布")
- **Output**: desired output file name

### Step 2 & 3: Extract Data

**Online spreadsheet (any URL)**:

Supported platforms: DingTalk/AliDocs, Feishu/Lark, Tencent Docs, Google Sheets, WPS, Notion, etc.

1. Navigate to the URL via browser tool
2. **If login is required**: take a screenshot, show it to the user, and ask them to scan QR code or log in manually. Wait for login to complete before proceeding.
3. Once the spreadsheet is loaded, try extracting data in order of preference:
   - **JS extraction (DingTalk/AliDocs)**: access `window.rawDataStore.docData.value.documentContent.checkpoint.content`, parse JSON → `content[sheetId].rows`
   - **JS extraction (generic)**: look for global data stores, iterate `window` keys for spreadsheet-related objects
   - **DOM extraction**: read cell values from table DOM elements
   - **Keyboard shortcut**: click cell A1, Ctrl+A to select all, evaluate JS to read selection
4. Parse extracted data → save as JSON for the matching script

**Login handling**:
- After navigating to URL, check if the page shows a login/QR screen (take screenshot)
- If login needed: inform user "该文档需要登录，请在浏览器中完成登录（扫码/账号密码）"
- Poll with screenshots every 10-15 seconds until the spreadsheet content appears
- Once logged in, proceed with data extraction

**Local Excel (.xlsx)**:
1. Copy .xlsx → .zip, unzip
2. Parse `xl/worksheets/sheet1.xml` as XML
3. Extract values from the specified column letter

**JSON file**: Load directly as array of strings or objects

**CSV file**: Read with csv module, extract specified column

### Step 4: Deduplicate Source

```python
unique_values = list(dict.fromkeys(values))  # preserve order, remove dupes
```

### Step 5: Filter Target (Optional)

If user specifies a filter condition:
```python
python scripts/semantic_match.py --status-col E --status "已发布"
```

### Step 6: Run Semantic Matching

```bash
python scripts/semantic_match.py \
  --source source_data.json \
  --target sheet1.xml \
  --target-col A \
  --status-col E \
  --status "已发布" \
  --source-label "词" \
  --target-label "标题" \
  --output result.xlsx
```

Multi-signal matching (see [matching-rules.md](matching-rules.md)):
- Exact / containment match
- Longest common substring (LCS) ratio
- Character bigram & trigram Jaccard similarity
- Key entity overlap (Chinese segments >= 3 chars)
- Configurable domain keyword co-occurrence rules

Each deduplicated source value produces **one row**. Multiple target matches are joined with "；".

### Step 7: Output Excel

Default columns: `源值 | 目标值 | 是否关联 | 关联理由`

Column headers are customizable via `--source-label` and `--target-label`.

## Environment Notes

- **Zero pip dependencies** -- uses only Python stdlib (`json`, `re`, `zipfile`, `xml.etree`, `csv`, `argparse`)
- If Python not installed, use embeddable version:
  ```bash
  curl -o python-embed.zip "https://www.python.org/ftp/python/3.12.3/python-3.12.3-embed-amd64.zip"
  ```

## Customization

- **Column selection**: `--target-col B` to match against column B
- **Filter**: `--status-col F --status "审核通过"` for any column/value filter
- **Labels**: `--source-label "关键词" --target-label "文章标题"` customizes Excel headers
- **Domain rules**: edit `DOMAIN_RULES` list in script to add/remove topic keyword groups
- **Person names**: edit `PERSON_NAMES` list for entity-level matching
- **Thresholds**: adjust `lcs`, `jaccard`, `jaccard3` thresholds in `semantic_match()`
