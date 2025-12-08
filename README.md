# SHA-256 Custom Tarot Studio + Aura Photo Lab Blueprint

## Overview (short)
- **Promise:** Personal 78-card SHA-driven tarot deck produced in ~60 minutes while the client is in-store; optional guidebook +60 minutes; aura photo studio next door for bundles.
- **Target price:** ~$150 core deck (affordable, youth-friendly) with upsells for foil, crystals, guidebook, aura bundle.
- **Tone:** Mystical, a little vintage/occult, playful about “code meets fate,” transparent about process.

## Part 1 – 78-Card SHA Tarot Generation System

### 1. Input data model
- Collect concise, repeatable fields (all text normalized and trimmed):
  - `name` (full legal or chosen name)
  - `dob` (YYYY-MM-DD)
  - `time` (HH:MM 24h; if unknown use `00:00`)
  - `location` (city, country)
  - `intention` (short free-text mantra/goal)
- Concatenate with pipe separators (deterministic, human-readable ordering): `"{name}|{dob}|{time}|{location}|{intention}"` encoded as UTF-8.
- Example intake payload:
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
SUITS = ["Wands", "Cups", "Swords", "Pentacles"]
RANKS = ["Ace", "Two", "Three", "Four", "Five", "Six", "Seven", "Eight", "Nine", "Ten", "Page", "Knight", "Queen", "King"]
MAJORS = [
    "The Fool", "The Magician", "The High Priestess", "The Empress", "The Emperor", "The Hierophant",
    "The Lovers", "The Chariot", "Strength", "The Hermit", "Wheel of Fortune", "Justice", "The Hanged Man",
    "Death", "Temperance", "The Devil", "The Tower", "The Star", "The Moon", "The Sun", "Judgement", "The World"
]

def hash_payload(data: dict) -> str:
    payload = {f: data.get(f, "") for f in INPUT_FIELDS}
    concatenated = "|".join(payload[f] for f in INPUT_FIELDS)
    return hashlib.sha256(concatenated.encode("utf-8")).hexdigest()  # 64 hex chars

def segment_hash(hash_hex: str):
    # 32 values (0–255)
    segments = [int(x, 16) for x in wrap(hash_hex, 2)]
    majors = segments[:22]                 # 22 dedicated values
    minors_seed = segments[22:]            # 10 seed values reused
    # expand to 56 by cycling
    minors = [minors_seed[i % len(minors_seed)] for i in range(56)]
    return majors, minors
```
- Determinism: same payload → same hash → same mappings. Any change to any field reshuffles attributes.

### 3. Card mapping logic
- **Major Arcana (22 cards)**
  - Fixed order as `MAJORS` list.
  - Each major uses its paired `majors[i]` value for derived traits (palette, element, tone, sigil seed).
- **Minor Arcana (56 cards)**
  - For card index `i` in 0–55, use `value = minors[i]`.
  - Suit: `suit = SUITS[value % 4]`.
  - Rank: `rank = RANKS[value % 14]`.
- **Derived attributes from value bits (both arcana):**
  - **Color palette (HSL):** `h = round((value / 255) * 360)`, `s = 55–75%` (map 0–255 to 55–75), `l = 40–65%` (map 0–255 to 40–65).
  - **Element:** `value % 4` → Fire, Water, Air, Earth.
  - **Personality tone buckets:** 0–63 nurturing; 64–127 analytical; 128–191 chaotic; 192–255 visionary.
  - **Sigil/icon seed:** use `value` as RNG seed for 3–5 icons (e.g., wave, spiral, feather, sunburst, compass).
- **Deterministic uniqueness:** use `(arcana, suit, rank, index)` as anchors and store `hash_signature` for traceability.

### 4. AI art prompt system
- **Prompt template:**
  > "{card_name}{minor_suffix}, element: {element}, mood: {tone}, dominant colors H{h} S{s}% L{l}%, symbols: {sigils}, client theme: {intention/theme}, style: vintage occult illustration + aura photography glow + soft-grain film, border-safe composition, print-ready tarot card."
- **Sigils:** pick 3–5 from a deterministic list seeded by `hash_signature` (e.g., spiral, open eye, feather, wave, starburst, lotus, key, lightning, compass).
- **Minor suffix:** `" ({rank} of {suit})"` for minors, empty for majors.
- **Example prompts:**
  - **Major – The Fool**
    - "The Fool, element: Air, mood: visionary, dominant colors H210 S65% L60%, symbols: spiral, feather, open eye, client theme: align my art with my purpose, style: vintage occult illustration + aura photography glow + soft-grain film, border-safe composition, print-ready tarot card."
  - **Minor – Three of Cups**
    - "Three of Cups, element: Water, mood: nurturing, dominant colors H280 S60% L58%, symbols: wave, crescent, lotus, client theme: align my art with my purpose, style: vintage mystical watercolor + etched linework + aura glow, border-safe composition, print-ready tarot card."
  - **Minor – Knight of Swords**
    - "Knight of Swords, element: Air, mood: analytical, dominant colors H190 S70% L52%, symbols: sword, lightning, compass, client theme: align my art with my purpose, style: vintage occult engraving + cinematic chiaroscuro + film grain, border-safe composition, print-ready tarot card."

### 5. Card meaning / guidebook logic
- Start from canonical meanings, then tilt with tone + intention:
  - Add 1–2 intention-derived keywords to the standard set.
  - Append a custom sentence referencing the client tone (e.g., visionary → "lean into bold experiments").
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
- Final deck JSON:
  ```json
  {
    "client_id": "<hash_hex>",
    "intake": {"name": "...", "dob": "...", "time": "...", "location": "...", "intention": "..."},
    "cards": [
      {
        "card_name": "The Fool",
        "arcana": "Major",
        "suit": null,
        "rank": null,
        "image_path": "output/<client_id>/images/00_the_fool.png",
        "back_image_path": "output/<client_id>/images/back.png",
        "palette": {"h": 210, "s": 65, "l": 60},
        "element": "Air",
        "tone": "visionary",
        "upright_text": "...",
        "reversed_text": "...",
        "hash_signature": "f3a9..."
      }
      // ... 77 more
    ],
    "layout": {"sheet_size": "A4 or Letter", "cards_per_sheet": 9, "bleed_mm": 3, "safe_margin_mm": 2}
  }
  ```
- Layout script consumes JSON → places fronts/backs into 9-up sheets → exports print-ready PDFs.

## Part 2 – 1-Hour Production Workflow Map

### Phase breakdown (~60 minutes)
- **0–5 min:** Intake, consent, payment auth; capture data → SHA → deck JSON.
- **5–15 min:** Generate prompts and launch AI renders (batch 78, 512–768px; upscale if time allows).
- **15–30 min:** Rendering in background; staff preps boxes/foil/charm options; capture aura photos if bundled.
- **30–40 min:** Layout generation (front/back PDFs), include registration marks for cutter.
- **40–55 min:** Print 8–10 sheets (fronts + backs), drying/cure as needed.
- **55–65 min:** Cut/round corners; quality check.
- **65–75 min:** Box/finish (foil sticker, crystal charm), handover. Practice to compress to ~60.

### Staffing (1–3 people)
- **Front-of-house (FOH):** greet, intake, payments, manage aura sessions, upsell options.
- **Tech operator:** run hash script, AI queue, layout, printer; monitor quality.
- **Finisher:** cutting, rounding, boxing, foil accents; FOH can double during slow periods.

### Parallelism
- While AI renders: FOH/Finisher prep box/foil, start aura session, cut pre-printed card backs for templates.
- While fronts print: backs queued; cutting of first sheets can start while later sheets print.
- Guidebook text generation can begin once deck JSON is ready, parallel to printing/cutting.

### Equipment & stations
- **Intake/kiosk:** tablet/laptop web form.
- **Compute station:** desktop/laptop (GPU or API) running hash + AI + layout scripts.
- **Print/cut:** 300–600 dpi cardstock printer; 9-up templates; guillotine or cutting plotter; corner rounder.
- **Finishing:** foil stamping (manual hot-foil or foil overlay/laminator), box folding jig, sleeves/bags.

### Process diagram
```
Client → Intake/Kiosk → SHA-256 + Deck JSON → AI Render Queue → Layout PDFs → Print → Cut/Round → Box/Finish → Handover
```

### Optional guidebook (+~60 minutes)
- **0–10 min:** Generate guidebook text from deck JSON (upright/reversed + client notes).
- **10–25 min:** Flow into zine template (A5/half-letter), paginate.
- **25–45 min:** Print duplex, fold/staple.
- **45–60 min:** Cover embellish (foil sticker) + QC + handover.

## Part 3 – Aura Photo Studio Plan

### Experience design
- Flow: Check-in → choose backdrop/style → quick questionnaire (mood/intention) → photo capture (3–5 poses) → select favorite → apply aura overlay/color map → print + digital share.
- Tarot integration:
  - Use aura photo as card back or special "Significator" card.
  - Aura colors can influence deck palette tweaks.
  - Bundle as "Aura + Tarot" or flagship "Soul Print Package".

### Tech + setup
- Camera: mirrorless/DSLR with tether; constant soft lights + halo/backlight.
- Software: tether capture + overlay app; simple script maps questionnaire answers (or live colors) to aura gradients; auto-export PNG/JPEG to tarot asset folder.
- File handoff: aura image saved as `output/{client_id}/aura/aura_main.png`; layout references for backs/inserts.

### Pricing & packages
- Quick aura shot (solo): $25–$35, 1 print + digital.
- Friends/duo: $45–$60, 2–3 prints.
- Aura + Tarot bundle: +$20–$30 to deck; includes aura back or significator.
- Premium "Soul Print" shoot: $80–$100, multiple looks + mini zine.

### Throughput
- 6–10 sessions/hour with one operator; runs parallel to deck rendering/printing.
- Start aura immediately after intake while AI renders for the deck.

## Part 4 – Branding Slab

### Brand name options
- Tarot boutique: **One-Hour Oracle** – "Your code becomes your deck." | **Hash & Halo Tarot** – "Cryptic data, luminous fate." | **SHA-Sigil Studio** – "From your digits to divination."
- Aura studio: **AuraField Studio** – "Step into your color field." | **Prism Booth** – "Photos with a pulse."

### Brand pillars
- Instant personalization | Mystical meets tech | Accessible luxury (affordable, tactile) | Transparency and play | Same-day satisfaction

### Visual direction
- Palette: deep ink (navy/black), moonstone silver, ember gold, mist lavender; aura gradients as accents.
- Typography: serif for headers (occult vibe), rounded grotesk for body (friendly/modern).
- Iconography: hash-inspired sigils, concentric aura rings, foil micro-icons on boxes.

### Voice & tone
- Warm, invitational, slightly nerdy: "code meets fate." Clear promises, minimal jargon, celebrate uniqueness, no gatekeeping.

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
- AI generation: API calls to image model (batch prompts, 512–768px square, optional upscale).
- Layout automation: Python + Pillow/WeasyPrint/ReportLab; uses deck JSON + template SVGs/PNGs to generate 9-up front/back PDFs.
- Orchestration pseudocode:
  ```python
  payload = collect_form()
  deck = build_deck_json(payload)            # hash, mapping, text
  prompts = prompts_from_deck(deck)
  images = render_ai(prompts, client_id=deck["client_id"])  # saves PNGs
  merge_into_layout(deck, images, template="templates/9up.svg")
  export_pdf(fronts="output/fronts.pdf", backs="output/backs.pdf")
  send_to_printer(["output/fronts.pdf", "output/backs.pdf"])
  ```

### Hardware / printer setup
- Card size: tarot 70×120 mm (2.75×4.75 in) with 3 mm bleed.
- Resolution: 300–600 dpi; render art at 900×1500 px or higher.
- Sheet layout: 9-up on A4/Letter; ~10 sheets total for 78 fronts/backs.
- Printer: high-quality inkjet/laser rated for 300+ gsm cardstock; straight paper path.
- Cutting: guillotine for stacks or cutting plotter with registration marks; corner rounder.
- Foil: small hot-foil press or foil overlay/laminator; foil sticker sheets for speed.

### File formats & naming
- Card fronts: `output/{client_id}/images/{##}_{card_name_snake}.png`
- Card backs: `output/{client_id}/images/back.png`
- Layout PDFs: `output/{client_id}/print/fronts.pdf`, `output/{client_id}/print/backs.pdf`
- Aura assets: `output/{client_id}/aura/aura_main.png`
- Guidebook: `output/{client_id}/guidebook/guidebook.pdf`

### Resilience & backup
- Store intake JSON + hash + prompts + generated images zipped by `client_id`.
- Keep layout PDFs for 90 days for instant reprints; log prompt template version.
- Nightly backup to cloud/storage; test restores weekly.
- Offline fallback: cache last good prompts; allow reprint from stored PDFs even if AI/API is down.
