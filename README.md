# SHA-256 Custom Tarot Studio + Aura Photo Lab Blueprint

## Overview (short)
A ready-to-run boutique system that creates a personalized 78-card tarot deck in ~60 minutes using SHA-256-derived prompts, on-site printing/finishing, and an adjacent aura photo studio for bundled experiences. Affordable ($120–$180 core price target), youth-friendly, and operationally lean.

## Part 1 – 78-Card SHA Tarot Generation System

### 1. Input data model
- Collect concise but meaningful fields:
  - `name` (full legal or chosen name)
  - `dob` (YYYY-MM-DD)
  - `time` (HH:MM, 24h, optional if unknown use `00:00`)
  - `location` (city, country)
  - `intention` (short free-text mantra/goal)
- Concatenate as a single UTF-8 string with pipe separators to avoid ambiguity: `"{name}|{dob}|{time}|{location}|{intention}"`.
- Example JSON payload:
  ```json
  {
    "name": "Aria Lumen",
    "dob": "2004-09-12",
    "time": "18:45",
    "location": "Portland, USA",
    "intention": "align my art with my purpose"
  }
  ```

### 2. SHA-256 pipeline (Python-like pseudocode)
```python
import hashlib
from textwrap import wrap

INPUT_FIELDS = ["name", "dob", "time", "location", "intention"]

# 1) Ingest JSON and normalize blanks
payload = {f: data.get(f, "") for f in INPUT_FIELDS}
concatenated = "|".join(payload[f] for f in INPUT_FIELDS)

# 2) Hash
hash_hex = hashlib.sha256(concatenated.encode("utf-8")).hexdigest()  # 64 hex chars

# 3) Segment hash into 2-hex chunks (0–255 each)
segments = wrap(hash_hex, 2)  # list of 32 segments

# Mapping helpers
major_segments = segments[:22]          # first 22 → Major Arcana
minor_segments = segments[22:]          # remaining 10 segments; reuse/loop to cover 56

def cycle_segments(seg_list, count):
    out = []
    idx = 0
    while len(out) < count:
        out.append(int(seg_list[idx % len(seg_list)], 16))
        idx += 1
    return out

major_values = [int(s, 16) for s in major_segments]  # 0–255
minor_values = cycle_segments(minor_segments, 56)
```
- Determinism: same payload → same hash → same card mappings.
- Re-seeding: changing any field regenerates the deck.

### 3. Card mapping logic
- **Major Arcana (22 cards):**
  - Order majors as standard list; assign `major_values[i] % 22` to pick each card position’s traits (palette, sigil, tone) but keep canonical names/ordering.
  - Each major gets one unique segment for extra traits (color, sigil seed).
- **Minor Arcana (56 cards):**
  - Suits ordered `[Wands, Cups, Swords, Pentacles]` (rename allowed, but fixed for determinism).
  - Map `minor_values[i]` to suit: `suit_index = minor_values[i] % 4`.
  - Map rank: `rank_index = minor_values[i] % 14` → Ace–10, Page, Knight, Queen, King.
- **Derived attributes from value bits:**
  - Color palette seed: map segment to HSL → `h = (value / 255) * 360`, `s = 55–75%`, `l = 40–65%`.
  - Element: `value % 4` → Fire, Water, Air, Earth.
  - Personality tone: bucket by value range (0–63 nurturing, 64–127 analytical, 128–191 chaotic, 192–255 visionary).
  - Sigil/icon seed: use segment as RNG seed for symbol generator.

### 4. AI art prompt system
- Base template:
  - "{Card Name}, {suit/rank if minor}, depicted with {element} symbolism, {personality tone} mood, dominant colors H{h}/S{s}%/L{l}%, client theme: {intention/theme}, style: vintage occult illustration + soft-grain film, subtle aura glow, crisp linework, border-safe composition."
- Include 3–5 symbols from sigil generator seeded by segment.
- Sample prompts:
  - **Major – The Fool**
    - "The Fool stepping onto a glowing path, faithful companion at side, element: Air, mood: visionary, dominant colors H210 S65 L60, aura rim light, sigils of spiral, open eye, feather; vintage occult engraving meets soft-grain film, border-safe, tarot card illustration."
  - **Minor – Three of Cups**
    - "Three of Cups, joyful trio raising chalices in a moonlit alley, element: Water, mood: nurturing, dominant colors H280 S60 L58, shimmering aura overlay, sigils of wave, crescent, lotus; vintage mystical watercolor + etched linework, soft matte, print-ready composition."
  - **Minor – Knight of Swords**
    - "Knight of Swords charging through electric storm, element: Air, mood: analytical, dominant colors H190 S70 L52, glowing sigils of sword, lightning, compass; cinematic chiaroscuro, soft-film grain, tarot framing."

### 5. Card meaning / guidebook logic
- Start from canonical meanings; tilt via hash-derived tone and intention keywords.
- Adjust keywords: select 2–3 standard terms + 1–2 custom terms derived from intention tokens.
- Upright/reversed meaning: append a sentence that references the client tone (e.g., "visionary" or "nurturing").
- JSON schema per card:
  ```json
  {
    "card_name": "The Fool",
    "arcana": "Major",
    "suit": null,
    "rank": null,
    "keywords": ["beginnings", "trust", "leap", "visionary"],
    "upright_meaning": "A fresh start with bold curiosity, guided by visionary impulses tied to your intention.",
    "reversed_meaning": "Hesitation or reckless leaps when vision outruns grounding; pause and re-center.",
    "client_specific_note": "Integrate your mantra into the first step you take this week.",
    "hash_signature": "f3a9..."  
  }
  ```

### 6. Data output for layout
- Final deck JSON object:
  ```json
  {
    "client_id": "hash_hex",
    "cards": [
      {
        "card_name": "The Fool",
        "arcana": "Major",
        "suit": null,
        "rank": null,
        "image_path": "output/{client_id}/images/00_the_fool.png",
        "back_image_path": "output/{client_id}/images/back.png",
        "palette": {"h": 210, "s": 65, "l": 60},
        "element": "Air",
        "tone": "visionary",
        "upright_text": "...",
        "reversed_text": "...",
        "hash_signature": "f3a9..."
      }
      // ... 77 more
    ],
    "layout": {
      "sheet_size": "A4 or Letter",
      "cards_per_sheet": 9,
      "bleed_mm": 3,
      "safe_margin_mm": 2
    }
  }
  ```
- Layout script consumes JSON to place fronts/backs into 9-up sheets; exports print-ready PDF.

## Part 2 – 1-Hour Production Workflow Map

### Phase breakdown (~60 minutes)
- 0–5 min: Intake & SHA capture (data entry, consent, payment auth).
- 5–15 min: Hash → deck JSON → prompt generation; launch AI renders.
- 15–30 min: AI rendering in batch; staff preps box/foils, cuts card backs for reprint templates.
- 30–40 min: Layout & print prep; generate 9-up PDFs for fronts/backs.
- 40–55 min: Print fronts/backs (8–10 sheets), cure/dry if needed.
- 55–65 min: Cutting (Cricut/guillotine) + corner rounding.
- 65–75 min: Boxing, optional foil/charm, quality check, handover (aim to tighten to 60 by practice).

### Staffing (1–3 people)
- Front-of-house (FOH): greets, intake, payments, explains options, starts aura session if bundled.
- Tech operator: runs hash script, AI queue, layout, printer.
- Finisher: handles cutting, rounding, boxing, foil accents; can be FOH if quiet hours.

### Parallelism
- While AI renders: Finisher preps boxes/foil stickers; FOH runs aura photos or upsell pitch.
- Printing fronts while backs render/queue; cut early sheets while later sheets print.
- Guidebook content generation can run right after card JSON is ready.

### Equipment & stations
- Intake/kiosk: tablet/laptop with web form.
- Compute station: desktop/laptop with GPU or API access for AI, running hash+layout scripts.
- Print/cut: high-res cardstock printer (300–600 dpi), 9-up templates, guillotine or cutting plotter, corner rounder.
- Finishing: foil stamping (manual hot foil press or foil overlay sheets), box folding jig, sleeves.

### Process diagram
```
Client → Intake/Kiosk → SHA-256 + Deck JSON → AI Render Queue → Layout PDFs → Print → Cut/Round → Box/Finish → Handover
```

### Optional guidebook (+~60 minutes)
- 0–10 min: Generate guidebook text from deck JSON (upright/reversed + client notes).
- 10–25 min: Flow into zine template (A5/half-letter), paginate PDF.
- 25–45 min: Print duplex, fold/staple.
- 45–60 min: Cover embellish (foil sticker), QC, handover.

## Part 3 – Aura Photo Studio Plan

### Experience design
- Flow: Check-in → choose backdrop/filters → quick questionnaire (mood/intention) → photo capture (3–5 poses) → select favorite → apply aura overlay/color map → print + digital share.
- Tarot integration:
  - Use aura photo as card back or a special "Significator" card.
  - Aura colors inform deck palette tweaks.
  - Bundle with deck as "Soul Print Package" (aura + full deck).

### Tech + setup
- Camera: mirrorless/DSLR with tether; constant soft lights + backlight halo.
- Software: tether capture + overlay app (custom script maps questionnaire answers or live colors to aura gradients); auto-export PNG/JPEG to tarot asset folder.
- File handoff: aura image saved as `output/{client_id}/aura/aura_main.png`; layout script references for backs or inserts.

### Pricing & packages
- Quick aura shot (solo): $25–$35, 1 print + digital.
- Friends/duo session: $45–$60, 2–3 prints.
- Aura + Tarot bundle: add $20–$30 to deck; includes aura back design or significator.
- Premium "Soul Print" shoot: $80–$100, multiple looks + mini zine.

### Throughput
- 6–10 sessions/hour with one operator; runs parallel to deck rendering/printing.
- Can start aura immediately after intake while AI renders.

## Part 4 – Branding Slab

### Brand name options
- Tarot boutique:
  - **One-Hour Oracle** – "Your code becomes your deck."
  - **Hash & Halo Tarot** – "Cryptic data, luminous fate."
  - **SHA-Sigil Studio** – "From your digits to divination."
- Aura studio:
  - **AuraField Studio** – "Step into your color field."
  - **Prism Booth** – "Photos with a pulse."

### Brand pillars
- Instant personalization
- Mystical meets tech
- Accessible luxury (affordable, tactile)
- Transparency and play
- Same-day satisfaction

### Visual direction
- Palette: deep ink (navy/black), moonstone silver, ember gold, mist lavender; aura gradients as accents.
- Typography: serif for headers (occult vibe), rounded grotesk for body (friendly/modern).
- Iconography: hash-inspired sigils, concentric aura rings, foil micro-icons on boxes.

### Voice & tone
- Warm, invitational, a bit nerdy: "code meets fate"; clear promises, minimal jargon; celebrate uniqueness, no gatekeeping.

## Part 5 – Pricing Matrix

| Product/Tier | Includes | Est. cost (materials+time) | Suggested price | Approx. gross margin |
| --- | --- | --- | --- | --- |
| **Basic Deck** | 78 cards, standard box, matte finish | ~$40–$50 | $120 | ~58–67% |
| **Standard Deck** | Basic + foil accents on box/cards | ~$50–$60 | $150 | ~60–67% |
| **Premium Deck** | Standard + crystal laminate/charm + custom back | ~$65–$75 | $180 | ~58–64% |
| **Guidebook Add-on** | 32–48 page zine, b/w or light color | ~$10–$15 | +$20–$30 | ~50–66% |
| **Aura Bundle** | Aura shoot + deck back or significator | ~$10–$15 (time + print) | +$20–$30 | ~50–66% |
| **Soul Print Package** | Aura shoot + full deck + foil box | ~$60–$75 | $170–$200 | ~56–63% |

_Assumptions: cardstock/laminate/foil per deck $20–$30; labor baked into cost column; tighten margins with efficiency and bulk stock._

## Part 6 – Tech Stack & Printer Setup

### Software stack
- Intake: simple web form (React/Flask/etc.) saves JSON.
- Hash + mapping: Python script (hashlib) producing deck JSON.
- AI generation: API calls to image model (batch prompts, 512–768px square, upscaling if time allows).
- Layout automation: Python + Pillow/WeasyPrint/ReportLab; takes deck JSON and template SVGs/PNGs to generate 9-up front/back PDFs.
- Orchestration pseudocode:
  ```python
  payload = collect_form()
  deck = build_deck_json(payload)
  prompts = prompts_from_deck(deck)
  images = render_ai(prompts, client_id=deck["client_id"])
  merge_into_layout(deck, images, template="templates/9up.svg")
  export_pdf(fronts="output/fronts.pdf", backs="output/backs.pdf")
  send_to_printer(["output/fronts.pdf", "output/backs.pdf"])
  ```

### Hardware / printer setup
- Card size: tarot 70×120 mm (2.75×4.75 in) with 3 mm bleed.
- Resolution: 300–600 dpi; render art at 900×1500 px or higher.
- Sheet layout: 9-up on A4/Letter; 8–10 sheets total for 78 fronts/backs.
- Printer: high-quality inkjet/laser rated for 300+ gsm cardstock; straight paper path.
- Cutting: guillotine for stacks or cutting plotter (Cricut/Silhouette) with registration marks; corner rounder.
- Foil: small hot-foil press or foil overlay/laminator; foil sticker sheets for speed.

### File formats & naming
- Card fronts: `output/{client_id}/images/{##}_{card_name_snake}.png`
- Card backs: `output/{client_id}/images/back.png`
- Layout PDFs: `output/{client_id}/print/fronts.pdf`, `.../backs.pdf`
- Aura assets: `output/{client_id}/aura/aura_main.png`
- Guidebook: `output/{client_id}/guidebook/guidebook.pdf`

### Resilience & backup
- Store intake JSON + hash + prompts + generated images zipped by `client_id`.
- Keep layout PDFs for 90 days for instant reprints.
- Nightly backup to cloud/storage; log versions of prompt templates for reproducibility.
- Offline fallback: cache last good prompts; allow reprint from stored PDFs even if AI/API is down.
