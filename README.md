# VETASSESS.CQV
Automation for filling VETASSESS's Chinese Qualification Verification
CQV_MASTER_WORKFLOW.md — The complete prompt/instruction set covering the end-to-end pipeline. When you start a new client case, this prompt tells Claude to:
Ask upfront: semester or no-semester? English translation available?
Dual-extract from the uploaded transcript PDF (pdfplumber text + Tesseract OCR with chi_sim+eng)
Cross-validate both extractions, producing a discrepancy report with ⚠️ flags for real errors (column shifts, OCR misreads, missing rows)
Present merged data for your review before proceeding
Generate XFDF(s) — up to 3 files depending on the case (CN with/without semester + EN)
