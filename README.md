给不熟悉程序员这套的RMA同行一句话解释:
把CQV_MASTER_WORKFLOW.md下载下来做为附件和申请人的中英文成绩单一起丢到Claude的Prompt里面运行, 它会判断是否有Semester然后生成两个.xfdf文件, 一个对应Editable PDF Chinese一个对应English. .然后用Acrobat打开VETASSESS给的Editable PDF模板, 选Prepare Form-Import Data打开对应的xfdf文件(Acrobat默认是fdf文件, 需要在右下角文件类型选择换成xfdf), 等几秒种所有信息就自动导入进来了; 
# CQV XFDF Toolkit

Automated pipeline for converting Chinese university transcripts into XFDF import files for CQV (Course Qualification Verification) editable PDF forms, commonly used in Australian migration and education applications.

## Overview

This toolkit handles three CQV PDF form variants:

| Form | Columns | Capacity | Use Case |
|------|---------|----------|----------|
| **CN with semesters** | 课程名, 分数, 学分, 学时, 学时单位, 课程类别, 学期 | 119 rows / 5 pages | Transcripts with semester info |
| **CN without semesters** | 课程名, 分数, 学分, 学时, 学时单位, 课程类别 | 119 rows / 5 pages | Transcripts without semester info |
| **EN translation** | 中文, 翻译 | 160 rows / 4 pages | Chinese–English course name pairs |

## Quick Start

```bash
# CN with semesters
python cqv_unified.py transcript.csv --form cn_semester -o output.xfdf

# CN without semesters
python cqv_unified.py transcript.csv --form cn_no_semester -o output.xfdf

# EN translation table
python cqv_unified.py translation.csv --form en -o output.xfdf

# Sort by semester before filling
python cqv_unified.py transcript.csv --form cn_semester --sort -o output.xfdf

# Generate CN + EN in one command
python cqv_unified.py transcript.csv --form cn_semester --en-csv translation.csv -o output
```

## CSV Input Formats

### CN with semesters (7 columns)

```csv
课程名,分数,学分,学时,学时单位,课程类别,学期
高等数学(一),89,5,80,学时,必修,2018-2019-1
大学英语(一),85,4,64,学时,必修,2018-2019-1
```

### CN without semesters (6 columns)

```csv
课程名,分数,学分,学时,学时单位,课程类别
高等数学(一),89,5,80,学时,必修
大学英语(一),85,4,64,学时,必修
```

### EN translation (2 columns)

```csv
中文,翻译
高等数学(一),Advanced Mathematics (I)
大学英语(一),College English (I)
```

English column aliases are also accepted (e.g. `course_name`, `score`, `credits`, `semester`, `chinese`, `english`).

## Importing into Acrobat

1. Open the blank CQV editable PDF in Adobe Acrobat
2. Navigate to **准备表单 (Prepare Form)** → **更多 (More)** → **导入数据 (Import Data)**
3. Select the generated `.xfdf` file
4. All fields populate instantly

## PDF Form Field Schemas

The CQV editable PDFs use a non-obvious field naming convention. This toolkit encapsulates the mapping logic so you never need to think about it, but the schemas are documented here for reference.

### CN With Semesters (7 columns × 119 rows)

Page layout: `[24, 24, 24, 24, 23]` rows across 5 pages.

```python
def get_field_id(page, row, col):  # all 1-indexed
    if page == 1:
        return f"fill_{(row - 1) * 7 + col}"
    elif row == 1:
        return f"fill_{170 + col}_{page}"
    else:
        return f"fill_{(row - 2) * 7 + col}_{page}"
```

**Page 1:** Sequential — `fill_1` through `fill_168` (24 rows × 7 cols).

**Pages 2–5:** Row 1 is a "continuation header" using `fill_171_p` through `fill_177_p`. Remaining rows use `fill_1_p` through `fill_161_p` (or `fill_154_p` on page 5).

### CN Without Semesters (6 columns × 119 rows)

Page layout: `[23, 24, 24, 24, 24]` rows across 5 pages.

```python
def get_field_id(page, row, col):  # all 1-indexed
    if page == 1:
        return f"fill_{(row - 1) * 6 + col}"
    elif row == 1:
        return f"fill_{170 + col}_{page}"
    else:
        return f"fill_{(row - 2) * 6 + col}_{page}"
```

Same continuation-header pattern on pages 2–5, but with 6 columns instead of 7.

### EN Translation (2 columns × 160 rows)

Page layout: `[40, 40, 40, 40]` rows across 4 pages. No continuation quirks.

```python
def get_field_id(page, row, col):  # all 1-indexed; col 1=中文, col 2=翻译
    num = (row - 1) * 2 + col
    if page == 1:
        return f"fill_{num}"
    else:
        return f"fill_{num}_{page}"
```

Odd-numbered fields = Chinese, even-numbered = English.

## Full Workflow (with Claude)

For end-to-end processing starting from a raw transcript PDF, see `CQV_MASTER_WORKFLOW.md`. The workflow includes:

1. **Form selection** — semester / no-semester, EN translation needed?
2. **Dual extraction** — pdfplumber text parsing + Tesseract OCR (`chi_sim+eng`)
3. **Cross-validation** — row-by-row comparison with discrepancy report
4. **Human review** — flagged items presented for confirmation
5. **XFDF generation** — using the verified, merged data

### Dual Extraction Helper

```bash
# Requires: pdfplumber, pypdfium2, pytesseract, tesseract-ocr-chi-sim
python dual_extract.py transcript.pdf output_dir/
```

Outputs:
- `method_a_text.txt` — pdfplumber text extraction
- `method_a_tables.json` — pdfplumber table extraction
- `method_b_ocr.txt` — Tesseract OCR text
- `pages/page_N.png` — page images for visual review

### Claude Prompt Template

See `PROMPT_TEMPLATE.md` for a reusable prompt that instructs Claude to convert raw OCR text into the structured CSV format expected by `cqv_unified.py`.

## Files

| File | Description |
|------|-------------|
| `cqv_unified.py` | Main converter — CSV to XFDF for all three form types |
| `csv_to_xfdf.py` | Standalone converter for CN-with-semesters only (legacy) |
| `dual_extract.py` | Dual extraction helper (pdfplumber + OCR) |
| `CQV_MASTER_WORKFLOW.md` | Full workflow prompt for use with Claude |
| `PROMPT_TEMPLATE.md` | Prompt template for structuring OCR text into CSV |

## Dependencies

```bash
pip install pdfplumber pypdfium2 pytesseract
# For OCR:
apt install tesseract-ocr tesseract-ocr-chi-sim
```

## Common Transcript Layouts

Chinese university transcripts vary widely. The toolkit handles:

- **Semester-grouped** — courses listed under semester headers (most common). Semester is propagated to all courses in the section.
- **Full table** — every row includes a semester column. Direct mapping.
- **Side-by-side** — two course columns per page (left/right halves). Each half parsed separately.
- **Mixed CN/EN** — bilingual course names. Chinese used for CN form; both used for EN translation form.

## Known Pitfalls

- **Column shifting** — PDF text extraction frequently concatenates adjacent columns (e.g. score merges with course name). This is the primary reason for dual extraction.
- **Multi-line course names** — long names wrap to the next line, creating phantom rows in text extraction.
- **CID fonts** — some PDFs return `(cid:XX)` garbage instead of text. Rely on OCR in these cases.
- **Grade formats** — keep as-is from the transcript: 百分制 (0–100), 五级制 (优/良/中/及格/不及格), letter grades (A–F), or pass/fail (P/F, Pass, 合格).
- **Repeated headers** — column headers repeated on each transcript page should be stripped.
- **Summary rows** — rows like "总学分", "平均绩点", "GPA" at the bottom should be excluded.

## License

For internal use. CQV editable PDF forms are proprietary to the respective assessment bodies.
