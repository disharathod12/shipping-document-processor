# Shipping Document Processor (MVP)

A lightweight, rule-based intelligent document processing (IDP) pipeline that takes a scanned shipping document (image or PDF), preprocesses it, runs OCR, classifies its document type, and extracts key structured fields as JSON.

## Overview

Shipping and logistics teams handle large volumes of semi-structured paper documents — commercial invoices, packing lists, bills of lading — that are usually re-typed into systems by hand. This project is a small, working prototype of the first stage of an automation pipeline for that problem: turning a scanned document into structured, machine-readable data.

This is a portfolio MVP built to demonstrate the pipeline end-to-end using pretrained/classical tools (no model training required), not a production system.

## Problem Statement

Manually reading and re-keying data from shipping documents is slow and error-prone. Before any automation is possible, a system needs to reliably: (1) read the text off a scanned document, (2) work out what *kind* of document it is, and (3) pull out the handful of fields that actually matter (dates, container numbers, weights, amounts, etc.). This project implements and demonstrates that first stage.

## Features

- Accepts image (`.png`, `.jpg`, `.tiff`) or PDF input
- OpenCV-based preprocessing (denoising + adaptive thresholding) to improve OCR quality
- OCR via Tesseract (through `pytesseract`), including word-level bounding boxes
- Rule-based document-type classification (Commercial Invoice / Packing List / Bill of Lading)
- Regex-based extraction of ~10 shipping-document fields (document number, date, shipper, consignee, ports, container number, weights, amount)
- Structured JSON output per document
- Bundled sample-document generator so the pipeline is testable immediately with no dataset download

## Architecture / Pipeline

```
Input file (image / PDF)
        │
        ▼
 preprocess.py      → PyMuPDF (PDF→image) + OpenCV (grayscale, denoise, adaptive threshold)
        │
        ▼
 ocr_processor.py    → pytesseract (Tesseract OCR) → text + word bounding boxes
        │
        ▼
 classifier.py       → keyword/rule-based document-type classification
        │
        ▼
 extractor.py        → regex-based field extraction
        │
        ▼
 output/*.json        → structured result
```

**What is rule-based:** document classification (keyword scoring) and field extraction (regex).
**What is model-based:** OCR text recognition (Tesseract's pretrained model). No model is trained as part of this project.

## Technologies

Python 3, OpenCV, PyMuPDF (`fitz`), Tesseract OCR / `pytesseract`, Pillow, NumPy.

## Installation

```bash
# 1. Clone and enter the repo
git clone <your-repo-url>
cd shipping-document-processor

# 2. Create a virtual environment
python -m venv venv
venv\Scripts\activate            # Windows
# source venv/bin/activate       # macOS/Linux

# 3. Install Python dependencies
pip install -r requirements.txt

# 4. Install the Tesseract OCR engine (separate from the Python package)
#    Windows installer: https://github.com/UB-Mannheim/tesseract/wiki
#    After installing, make sure tesseract.exe is on your PATH, or set:
#    pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"
```

## Usage

```bash
# Generate 3 sample shipping documents (no dataset needed)
python generate_samples.py

# Run the pipeline on all bundled samples
python app.py --demo

# Run on a single document (image or PDF)
python app.py --input sample_documents/sample_invoice.png
```

Output JSON is written to `output/<filename>.json`.

## Example Input / Output

Input: a synthetic sample commercial invoice image (`sample_documents/sample_invoice.png`).

Output (`output/sample_invoice.json`), produced by an actual run of this pipeline:

```json
{
  "source_file": "sample_invoice.png",
  "document_type": "COMMERCIAL_INVOICE",
  "classification_confidence_score": 3,
  "extracted_fields": {
    "document_number": "INV-20595",
    "date": "12-Aug-2026",
    "shipper": "Orion Textiles Pvt Ltd, Kharagpur, IN",
    "consignee": "Baltic Freight Traders, Rotterdam, NL",
    "port_of_loading": "Kolkata (INCCU)",
    "port_of_discharge": "Rotterdam (NLRTM)",
    "vessel": null,
    "container_no": "MSCU2190863",
    "total_weight": "4372 KG",
    "net_weight": null,
    "total_amount": "USD 37608.00"
  },
  "fields_found": 9,
  "fields_attempted": 11,
  "ocr_word_count": 54
}
```

On the 3 bundled sample documents, the pipeline correctly classified all 3 document types and extracted 9/11, 9/11, and 5/11 of the attempted fields respectively (fields not present on a given document type — e.g. "vessel" on an invoice — are expected to come back `null`). This is a result on 3 synthetic demo documents, not a benchmark — see Limitations.

## Dataset

No dataset is required to run this MVP — `generate_samples.py` creates synthetic sample documents locally, and classification/extraction are rule-based (no training data needed).

For further testing against real-world scanned documents, the closest available public dataset is:

- **Name:** ICDAR-SROIE (Scanned Receipts OCR and Information Extraction)
- **Link:** https://huggingface.co/datasets/darentang/sroie
- **Why:** It's the most widely used public dataset combining OCR + key-field-extraction ground truth on scanned documents; easiest to pull with `datasets.load_dataset("darentang/sroie")`.
- **Input:** ~1,000 scanned receipt images.
- **Labels:** word-level bounding boxes + key fields (company, date, address, total).
- **Note:** it's receipts, not shipping documents specifically — there is no well-known public labelled dataset of invoices/packing lists/bills of lading. It's listed here as the nearest fit for future extension, not something the current MVP was built or measured on.

## Limitations

- Classification is keyword-based, not a trained ML classifier — it will not generalize to document layouts/wordings very different from the patterns in `classifier.py`.
- Field extraction assumes reasonably consistent "Label: Value" phrasing; real-world scans with unusual layouts will need extra/adjusted regex patterns.
- Tested only on synthetic sample documents generated by this repo, not on a real-world labelled shipping-document dataset — no accuracy/precision numbers are claimed beyond what's shown in "Example Input/Output" above.
- No deskewing/rotation-correction for photographed (as opposed to scanned) documents yet.
- Single-page documents only (PDF handling uses page 1).

## Future Improvements

- Replace keyword classifier with a trained text classifier once labelled data is available
- Add layout-aware extraction (e.g. using OCR bounding boxes to handle table-style fields)
- Add deskew/rotation correction for phone-photographed documents
- Multi-page PDF support
- Evaluate against a real labelled dataset and report actual precision/recall
