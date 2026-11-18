# CMM — Creating PDFs (branded one-sheet)

How CMM turns any deliverable into a printable, on-brand one-page PDF. Read this whenever you offer or build a PDF (Principle 4). **Run the template below verbatim** — do not improvise PDF code; improvised layouts drift off-brand and break.

The layout is a **design system with explicit measurements**, not a feel. Do not substitute your own judgment for the numbers below ("looks airy," "seems generous"). Fill in the content, keep the geometry.

---

## What this is for

A student reaches a milestone (model locked, offer locked, Sacred Code built) and wants something they can keep offline or print. You hand them a single, clean, branded page — not a report. **One sheet, one purpose.** If the content won't fit on one page, it's too much for a one-sheet — tell them, and offer to split it.

---

## The hard constraint — no internet in the sandbox

Code Interpreter has **no network access**. It cannot download a logo, a font, or anything from a URL. So:

- **The logo** must be uploaded into the GPT's **Knowledge** (`cmm-emblem.png`). Then it lives in the sandbox filesystem and the template can embed it. If it isn't there, the template falls back to a clean text wordmark — never error out over a missing logo.
- **Brand fonts** (Cormorant Garamond) would likewise need the `.ttf` uploaded as Knowledge. The Google Fonts download is two **variable** files — `CormorantGaramond-VariableFont_wght.ttf` (upright) and `CormorantGaramond-Italic-VariableFont_wght.ttf` (italic). The template deliberately loads the **upright** one; the italic must not win the glob or every heading turns italic. If absent, it falls back to a built-in serif. This is expected and fine.
- **Colors** need nothing — they're hex values, baked in below.

**Never tell the student to "add a logo URL" or fetch anything online.** If an asset isn't in Knowledge, degrade gracefully and move on.

---

## Brand tokens (the theme)

Light one-sheet — dark ink on cream, gold accent. These are the CMM colors from the launch page.

| Token | Hex | Use |
|---|---|---|
| Ink | `#1A1438` (midnight) | Body text, headings |
| Ink-soft | `#5A5175` | Labels, captions, secondary |
| Accent | `#A8801A` (gold-deep) | Rules, section headings, the wordmark — legible on cream |
| Accent-bright | `#D4AF37` (gold) | Fine hairlines only |
| Paper | `#FAF7F2` (cream) | Page background |

Fonts: **serif** for titles/headings (Cormorant Garamond → Georgia/Times fallback), **sans/system** for body and labels.

---

## Layout — the one-sheet

Top to bottom. **Margins: left 0.82" · right 0.82" · top 0.72" · bottom 0.72"** (balanced, still generous — do not revert to 0.9").

1. **Emblem** — centered, ~0.9", at the top. (Or the wordmark "COSMIC MONEY MACHINE" in gold small-caps if no logo file.)
2. **Title** — the deliverable's name in serif (e.g. *Your Magic Model*, *Your Offer*, *Your Sacred Code — One-Sheet*).
3. **Gold hairline** rule under the title/subtitle.
4. **Body** — the content, passed in as typed blocks (Heading · Body · Label — see below).
5. **Footer** — thin gold rule, then a small ink-soft line: the student's name / niche and the date. **Never** name the coach or sign it (unbranded rule holds in PDFs too).

---

## The design system (use these numbers)

### Typography

Point size / leading, applied via `ParagraphStyle` (not by hand):

| Style | Size | Leading | Colour |
|---|---|---|---|
| Title | 25 pt | 29 pt | Ink |
| Subtitle | 10 pt | 13 pt | Ink-soft |
| Section Heading | 14.5 pt | 17 pt | Accent (gold) |
| Body | 11 pt | 16 pt | Ink |
| Label / Secondary | 10.5 pt | 15 pt | Ink-soft |
| Footer | 8.5 pt | 10 pt | Ink-soft |

Use these unless a specific document genuinely requires otherwise.

### Vertical rhythm

Spacing comes from `spaceBefore` / `spaceAfter` on the styles — **not** from stacked blank lines or `Spacer()`:

| Between | Space |
|---|---|
| Logo → Title | 6–8 pt |
| Title → Subtitle | 4–6 pt |
| Subtitle → Gold rule | 8–10 pt |
| Section Heading | 12 pt before · 5 pt after |
| Body paragraph | 4 pt after |
| Label → its value | 1–2 pt after (keep the pair tight) |
| Footer | 10 pt before divider · 4 pt after divider |

Do not leave whitespace to the model's discretion.

### Content blocks — three styles, not one

Every piece of content is one of three block types:

- **Heading** — a section name (gold serif).
- **Body** — a sentence or the student's verbatim line (ink).
- **Label** — a short field name in the Label style (ink-soft), e.g. **Old ceiling**, **Decision**, **Before**, **The Turn**, **The Bridge to Now**.

Use **Label** for field names instead of adding a blank line or bolding a body line. A label sits tight above the value it introduces.

### Grouping — whitespace separates sections, not thoughts

Keep related content together as a group; let the section-heading spacing do the separating. So:

```
What Would Hold You Back
  Old ceiling
  "Rest has to be earned."
  Decision
  "My worth is not a deliverable."
```

**not** a blank line between every label and value. Tight pairs inside a group; real space only between sections.

### Do not build layout with Spacer()

Vertical rhythm should come from `ParagraphStyle` `spaceBefore`, `spaceAfter`, and `leading`. `Spacer()` should normally appear **only**:

- after the logo, and
- before the footer.

### Density target

- **Aim for 65–80 % page occupancy.**
- If the page looks **sparse (< 60 %)** → reduce whitespace *before* touching font size.
- If the page is **crowded (> 90 %)** → reduce paragraph spacing *before* touching font size.
- Maintain **one page** whenever possible; overflow means split, not shrink-to-fit.

---

## The template — run this verbatim

Fill `TITLE`, `SUBTITLE`, and `SECTIONS` (typed blocks) from the student's locked content, then run. Everything else is fixed.

```python
from reportlab.lib.pagesizes import letter
from reportlab.lib.units import inch
from reportlab.lib.colors import HexColor
from reportlab.lib.styles import ParagraphStyle
from reportlab.lib.enums import TA_CENTER
from reportlab.platypus import (SimpleDocTemplate, Paragraph, Spacer,
                                Image, HRFlowable)
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
import glob, os

# --- brand tokens ---
INK   = HexColor("#1A1438")
SOFT  = HexColor("#5A5175")
GOLD  = HexColor("#A8801A")
PAPER = HexColor("#FAF7F2")

# --- fonts: prefer the UPRIGHT Cormorant if uploaded, else built-in serif ---
# Google Fonts ships variable fonts; the italic file must NOT win the glob,
# or every heading renders italic. Load at default instance — fine for a one-sheet.
SERIF = "Times-Roman"
_ttfs = [p for r in ("/mnt", os.path.expanduser("~/.claude")) for p in glob.glob(r + "/**/*ormorant*.ttf", recursive=True)]
_upright = [p for p in _ttfs if "italic" not in p.lower()] or _ttfs
for p in _upright:
    try:
        pdfmetrics.registerFont(TTFont("Cormorant", p)); SERIF = "Cormorant"; break
    except Exception:
        pass

# --- locate the emblem if it was uploaded as Knowledge ---
LOGO = None
for pat in ("/mnt/**/cmm-emblem*", "/mnt/**/*emblem*", "/mnt/**/*cmm*log*"):
    hits = glob.glob(pat, recursive=True)
    if hits: LOGO = hits[0]; break

# ====== FILL THESE FROM THE STUDENT'S CONTENT ======
# Each SECTIONS item is (kind, text) where kind is "head" | "label" | "body".
# Group a label tight above its value; let headings do the separating.
TITLE    = "Your Magic Model"
SUBTITLE = "Jane Doe · Coaches burnt-out nurses"     # name · niche
SECTIONS = [
    ("head",  "The Promise"),
    ("body",  "Reclaim your calm."),
    ("head",  "What Would Hold You Back"),
    ("label", "Old ceiling"),
    ("body",  "“Rest has to be earned.”"),
    ("label", "Decision"),
    ("body",  "“My worth is not a deliverable.”"),
]
OUTFILE  = "CMM-Magic-Model.pdf"
# ===================================================

# --- design system: sizes, leading, and rhythm live in the styles ---
title_s = ParagraphStyle("t", fontName=SERIF, fontSize=25, leading=29,
                         textColor=INK,  alignment=TA_CENTER, spaceAfter=5)
sub_s   = ParagraphStyle("s", fontName="Helvetica", fontSize=10, leading=13,
                         textColor=SOFT, alignment=TA_CENTER, spaceAfter=9)
head_s  = ParagraphStyle("h", fontName=SERIF, fontSize=14.5, leading=17,
                         textColor=GOLD, spaceBefore=12, spaceAfter=5)
body_s  = ParagraphStyle("b", fontName="Helvetica", fontSize=11, leading=16,
                         textColor=INK,  spaceAfter=4)
label_s = ParagraphStyle("l", fontName="Helvetica", fontSize=10.5, leading=15,
                         textColor=SOFT, spaceAfter=1)
foot_s  = ParagraphStyle("f", fontName="Helvetica", fontSize=8.5, leading=10,
                         textColor=SOFT, alignment=TA_CENTER)
STYLE = {"head": head_s, "body": body_s, "label": label_s}

def on_page(canvas, doc):
    canvas.saveState()
    canvas.setFillColor(PAPER)
    canvas.rect(0, 0, letter[0], letter[1], stroke=0, fill=1)
    canvas.restoreState()

doc = SimpleDocTemplate(OUTFILE, pagesize=letter,
                        topMargin=0.72*inch, bottomMargin=0.72*inch,
                        leftMargin=0.82*inch, rightMargin=0.82*inch)
flow = []

# 1. emblem (or wordmark) — the one Spacer() allowed at the top
if LOGO:
    try:
        img = Image(LOGO, width=0.9*inch, height=0.9*inch); img.hAlign = "CENTER"
        flow += [img, Spacer(1, 7)]
    except Exception:
        LOGO = None
if not LOGO:
    flow += [Paragraph('<font color="#A8801A">C O S M I C&nbsp;&nbsp;M O N E Y&nbsp;&nbsp;M A C H I N E</font>',
             ParagraphStyle("w", fontName=SERIF, fontSize=11, alignment=TA_CENTER, spaceAfter=7))]

# 2–3. title, subtitle, gold rule (rhythm from the styles, not Spacers)
flow += [Paragraph(TITLE, title_s), Paragraph(SUBTITLE, sub_s),
         HRFlowable(width="100%", thickness=1.2, color=GOLD, spaceBefore=0, spaceAfter=2)]

# 4. body — typed blocks pick their style
for kind, text in SECTIONS:
    flow.append(Paragraph(text, STYLE[kind]))

# 5. footer — the one Spacer() allowed at the bottom
flow += [Spacer(1, 10),
         HRFlowable(width="40%", thickness=0.6, color=GOLD, spaceAfter=4),
         Paragraph(SUBTITLE, foot_s)]

doc.build(flow, onFirstPage=on_page, onLaterPages=on_page)
print("Saved", OUTFILE)
```

Then give the student the download link to `OUTFILE`.

---

## Rules that still hold in a PDF

- **Unbranded** — never name or sign as the coach. Footer is the student's name/niche only.
- **Their words** — the PDF preserves the locked content verbatim; you're formatting, not rewriting. Don't polish their voice into brochure copy.
- **Density, not sprawl** — hit the 65–80 % occupancy target above; keep it to one page.
- **Filename** — name it for the deliverable: `CMM-Magic-Model.pdf`, `CMM-Offer.pdf`, `CMM-Sacred-Code.pdf`.
- **Fails gracefully** — if the sandbox can't produce the file, format the same one-sheet cleanly in chat instead. Never leave them empty-handed.

---

## Setup checklist (one-time, in the GPT config)

- [ ] **Code Interpreter / Data Analysis** enabled.
- [ ] `cmm-emblem.png` uploaded as a **Knowledge** file (so the logo embeds).
- [ ] *(Optional)* `CormorantGaramond-VariableFont_wght.ttf` uploaded as Knowledge for exact brand type (add the `-Italic-` file too if you want italics available).
- [ ] This file (`cmm-pdf-style.md`) uploaded as Knowledge.
