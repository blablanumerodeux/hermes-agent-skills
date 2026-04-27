---
name: ocr-documents
description: Extract text from scanned documents, PDFs, and images using tesseract OCR with PIL preprocessing. Handles image-based PDFs that raw tesseract can't read.
tags: [ocr, tesseract, pdf, image-processing, pytesseract]
---

# ocr-documents Skill

Extract text from scanned documents and images using tesseract OCR with preprocessing.

## Prerequisites

```bash
# System packages
apt-get install -y tesseract-ocr tesseract-ocr-fra poppler-utils

# Python packages (in hermes-agent venv)
pip install pytesseract Pillow
```

## Pipeline

### Step 1 — Convert PDF to images (if PDF)

```python
import subprocess

def pdf_to_images(pdf_path, output_dir, dpi=200):
    """Convert PDF pages to JPEG images."""
    subprocess.run([
        'pdftoppm', '-r', str(dpi), '-jpeg', pdf_path,
        f'{output_dir}/page'
    ], check=True)
    # Returns list of image paths
    import glob
    return sorted(glob.glob(f'{output_dir}/page-*.jpg'))
```

### Step 2 — Preprocess with PIL

```python
from PIL import Image, ImageEnhance

def preprocess_for_ocr(image_path):
    """Grayscale + high contrast — unblackens faint scanned text."""
    img = Image.open(image_path).convert('L')
    enhancer = ImageEnhance.Contrast(img)
    img = enhancer.enhance(2.0)  # 2x contrast
    return img
```

### Step 3 — OCR with fallback PSM modes

```python
import pytesseract

def ocr_image(img, languages='fra+eng'):
    """Try multiple PSM modes; return best result."""
    best = ''
    # PSM modes: 6=block, 11=sparse text, 12=raw line
    for psm in [6, 11, 12]:
        text = pytesseract.image_to_string(img, lang=languages, config=f'--psm {psm}')
        if len(text.strip()) > len(best.strip()):
            best = text
    return best
```

## Full Example

```python
from PIL import Image, ImageEnhance
import pytesseract, subprocess, glob, os

pdf_path = '/tmp/document.pdf'
output_dir = '/tmp/ocr_work'
os.makedirs(output_dir, exist_ok=True)

# 1. Convert PDF to images
subprocess.run(['pdftoppm', '-r', '200', '-jpeg', pdf_path,
                f'{output_dir}/page'], check=True)

# 2. Preprocess + OCR each page
all_text = []
for img_path in sorted(glob.glob(f'{output_dir}/page-*.jpg')):
    img = Image.open(img_path).convert('L')
    img = ImageEnhance.Contrast(img).enhance(2.0)
    for psm in [6, 11, 12]:
        text = pytesseract.image_to_string(img, lang='fra+eng', config=f'--psm {psm}')
        if len(text.strip()) > 20:
            all_text.append(text)
            break

print('\n'.join(all_text))
```

## Tips

- **Start with PSM 6** — works well for most printed documents
- **Use `fra+eng`** for bilingual French/English docs
- **Grayscale + contrast** is critical for faint scanned text — without it, tesseract often returns empty
- **DPI 200+** for pdftoppm; lower DPI can cause empty OCR
- If tesseract still returns empty, try `tesseract file.jpg stdout -l fra --psm 6` directly to debug
