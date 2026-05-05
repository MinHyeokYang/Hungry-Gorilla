# SKUCheck — K-Beauty Global Regulatory Risk Scanner

> AI-powered SKU-level regulatory pre-screening tool for K-Beauty brands entering US and UK markets.
> Built as a single-file HTML/JS prototype. Claude API (Anthropic) powers the analysis engine.

---

## Project Context

### Problem Statement
K-Beauty indie brands receive buyer/platform inquiries for overseas sales, but have no fast way to check regulatory compliance before responding. They either Google manually or pay expensive consultants — both are too slow for real opportunities.

### Core User Goal
> "Is our product ready to sell in this country? If not, what exactly needs to be fixed?"

### Target User
- Small K-Beauty brand (founder or export manager)
- Has a SKU ready for domestic sale
- Received a buyer/platform inquiry for US or UK
- No in-house regulatory expert
- Needs a first-pass answer within the same day

---

## Current State (v0.1 — Prototype)

### What's Built
- `sku-risk-scanner.html` — Single-file prototype (HTML + CSS + Vanilla JS)
- Full input flow: country select → product info → ingredients → label upload → claims → docs
- Mock data results for both US and UK markets
- Claude API integration (optional — falls back to mock if no key)
- Checklist-style results UI with 🔴🟡🟢 risk levels
- English PDF report export via browser print
- Email capture after results (lead collection)

### What's NOT Built Yet
- Backend / server (none — all client-side)
- Real ingredient database (using Claude's knowledge only)
- Authentication / user accounts
- SKU history / saved results
- Payment / subscription

---

## Tech Stack

```
Frontend:   Vanilla HTML + CSS + JavaScript (no framework)
AI Engine:  Anthropic Claude API (claude-opus-4-5)
            - Direct browser fetch to https://api.anthropic.com/v1/messages
            - Requires header: anthropic-dangerous-direct-browser-access: true
Fonts:      Google Fonts (DM Sans, DM Mono, Instrument Serif)
PDF:        Browser window.print() → Save as PDF
Storage:    localStorage (API key only)
Hosting:    Static file — deployable to Vercel / Netlify / GitHub Pages
```

---

## File Structure

```
/
├── sku-risk-scanner.html     # MAIN FILE. Entire app lives here.
├── README.md                 # This file
└── (planned)
    ├── /api                  # Backend proxy for API key security
    ├── /db                   # Ingredient database JSON files
    │   ├── fda-prohibited.json
    │   ├── fda-restricted.json
    │   └── uk-annex-ii-iii.json
    └── /components           # If migrating to React/Next.js
```

---

## Architecture

### Data Flow

```
User Input
  │
  ├─ productName       (text input)
  ├─ category          (tag selection)
  ├─ country           ('us' | 'uk')
  ├─ ingredients       (textarea OR image/PDF → base64)
  ├─ labelImage        (image → base64 → Claude Vision)
  ├─ claims            (textarea)
  └─ ownedDocs         (tag multi-select)
  │
  ▼
buildPrompt() + buildSystemPrompt(country)
  │
  ▼
POST https://api.anthropic.com/v1/messages
  │
  ▼
Parse JSON from Claude response
  │
  ▼
renderResults() → checklist UI + stat grid + progress bar
  │
  ▼
exportPDF() → window.open() + window.print()
```

### Claude API Request Shape

```json
{
  "model": "claude-opus-4-5",
  "max_tokens": 4000,
  "system": "<country-specific system prompt>",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "<base64 string>"
          }
        },
        {
          "type": "text",
          "text": "<analysis prompt with all product data>"
        }
      ]
    }
  ]
}
```

Note: The `image` block is only included when `uploadedLabelBase64` is set. Content array always ends with the `text` block.

### Expected JSON Response Schema

Claude is instructed to return ONLY this JSON (no surrounding text):

```json
{
  "verdict": "fail | warn | pass",
  "verdictLabel": "string (Korean — e.g. '현재 판매 불가')",
  "summary": "string (1-2 sentence Korean summary)",
  "stats": {
    "danger": 0,
    "warning": 0,
    "pass": 0,
    "pending": 0
  },
  "sections": [
    {
      "icon": "emoji",
      "name": "string (Korean section label)",
      "items": [
        {
          "status": "fail | warn | pass | pending",
          "title": "string (Korean)",
          "desc": "string (Korean, 1-2 sentences, specific)",
          "action": "string (Korean, concrete next step) | null",
          "alt": "string (Korean, alternative suggestion) | null"
        }
      ]
    }
  ]
}
```

### Verdict Logic

```
verdict = "fail"   → any item.status === "fail"
verdict = "warn"   → no fail; at least one "warn" or "pending"
verdict = "pass"   → all items.status === "pass"
```

---

## Analysis Sections

### 🇺🇸 US (FDA / MoCRA 2022)

| Section Key | name field | What Claude checks |
|-------------|-----------|-------------------|
| ingredients | 성분 안전성 (Ingredient Safety) | FDA 21 CFR 700 prohibited/restricted list |
| claims | 마케팅 클레임 (Claim Risk) | OTC Drug claim — structure/function language |
| label | 라벨링 요건 (Label Requirements) | Responsible Person, net qty, warnings, INCI |
| mocra | MoCRA 준비도 | Facility registration, product listing, safety substantiation, adverse event SOP |

### 🇬🇧 UK (OPSS / UK RP / SCPN / CPSR)

| Section Key | name field | What Claude checks |
|-------------|-----------|-------------------|
| ingredients | 성분 안전성 (Ingredient Safety) | UK Cosmetics Regulation Annex II/III; Retinol ≤0.3% face limit |
| claims | 마케팅 클레임 (Claim Risk) | OPSS misleading claim rules; efficacy substantiation |
| label | 라벨링 요건 (UK Label) | UK RP name+address, PAO/expiry, batch no., origin |
| readiness | UK 규제 준비도 (SCPN · CPSR) | UK RP designation, SCPN notification, CPSR by Safety Assessor |

---

## Key Functions

### Input & Setup

| Function | Description |
|----------|-------------|
| `showInput()` | Hides landing, shows input section |
| `selectCountry(c)` | Sets `selectedCountry`, toggles active class on country buttons |
| `toggleTag(el)` | Toggles `.active` on category/doc tag buttons |
| `mockUpload(type)` | Opens real `<input type=file>`, reads file. For label images: sets `uploadedLabelBase64`. For text files: populates ingredients textarea |
| `toggleApiModal()` | Shows/hides API key modal |
| `saveApiKey()` | Saves key to `localStorage`, updates nav button state |

### Analysis

| Function | Description |
|----------|-------------|
| `startAnalysis()` | Entry point — hides input, shows loading, animates steps, branches to `runClaudeAnalysis()` or mock timeout |
| `runClaudeAnalysis(loadingInterval)` | Builds content array, POSTs to Claude API, parses JSON, stores in `analysisResult[country]`, calls `showResults()` |
| `buildSystemPrompt(country)` | Returns system prompt string for 'us' or 'uk'. Instructs Claude to respond ONLY in valid JSON |
| `buildPrompt(productName, ingredients, claims, countryLabel, country)` | Builds user message. Injects all form data + JSON schema + analysis instructions |

### Results

| Function | Description |
|----------|-------------|
| `showResults()` | Hides loading, shows results section, sets product name/meta |
| `getResultData(country)` | Returns `analysisResult[country]` (real) or `mockData[country]` (fallback). Adds `isReal` flag |
| `renderResults(country)` | Renders verdict badge, stat grid, progress bar, market tabs, section + item HTML |
| `switchTab(country)` | Re-renders results for selected country. If no cached result and API key exists, re-runs analysis |
| `exportPDF()` | Builds English print report HTML, opens in new window, calls `window.print()` |
| `resetApp()` | Clears `analysisResult`, `uploadedLabelBase64`, resets loading steps, shows landing |

---

## State Variables

```javascript
let selectedCountry = 'us';          // 'us' | 'uk' — current active market
let apiKey = '';                      // Anthropic API key — loaded from localStorage on init
let analysisResult = null;            // { us: ParsedJSON, uk: ParsedJSON } — null if not yet analyzed
let uploadedLabelBase64 = null;       // { data: string, mediaType: string } — null if no image uploaded
```

---

## Mock Data

Located in the `mockData` object. Used when:
- `apiKey` is not set or invalid
- API call fails
- Tab switch to a country not yet analyzed (and no API key)

Scenario: face serum containing `Hydroquinone` (FDA banned) and `Retinol 0.5%` (UK concentration warning), with drug claims ("주름을 없애주는") and missing Responsible Person info.

Both `mockData.us` and `mockData.uk` follow the exact same schema as the Claude JSON response. When adding new mock data, match the schema exactly.

---

## CSS Variables (Design Tokens)

```css
--bg: #F5F3EE          /* Page background — warm off-white */
--surface: #FDFCFA     /* Card background */
--surface2: #F0EDE6    /* Section headers, secondary surfaces */
--accent: #2D4A8A      /* Primary blue — buttons, links, active states */
--accent-light: #E8EDF7

--red: #C0392B   --red-bg: #FDF0EE   --red-border: #E8C5C0     /* fail */
--yellow: #A0620A  --yellow-bg: #FDF6EC  --yellow-border: #E8D5B0  /* warn */
--green: #1E6B3A   --green-bg: #EEF7F2   --green-border: #B8DFC8   /* pass */
```

Check item left-border color = risk level color (3px solid strip).

---

## Deployment

### Option A: Vercel (recommended)
```bash
# No build step. Deploy the HTML file directly.
npx vercel deploy --prod
# or drag sku-risk-scanner.html into vercel.com dashboard
```

### Option B: GitHub Pages
```bash
git init && git add . && git commit -m "init"
git remote add origin <repo-url>
git push -u origin main
# Settings → Pages → deploy from main /root
```

### API Key Security Note
Current implementation calls Anthropic API directly from the browser. The key is visible in localStorage and network requests.

**For any public deployment, proxy the API call through a server:**
```
Browser → POST /api/analyze (your server, key stored in env)
        → POST https://api.anthropic.com/v1/messages
```

---

## Next Steps for Codex

Work items are ordered by priority. Each item is self-contained.

### P0 — Must do before any public URL

**1. Backend API proxy**
- Create `POST /api/analyze` endpoint (Next.js API route or Vercel Edge Function)
- Move API key to `process.env.ANTHROPIC_API_KEY`
- Accept `{ country, productName, ingredients, claims, labelImageBase64, labelMediaType }` from client
- Forward to Claude API, return parsed JSON
- Update `runClaudeAnalysis()` in the HTML to call `/api/analyze` instead of Anthropic directly

**2. Input validation**
- Require at least ingredients OR label image before allowing analysis
- Show inline error if both are empty
- Validate product name is not empty

### P1 — Improve analysis quality

**3. Ingredient database**
- Create `/db/fda-prohibited.json` — list of INCI names banned under FDA 21 CFR 700
- Create `/db/uk-annex-ii.json` — UK/EU banned ingredients
- Create `/db/restricted.json` — ingredients with concentration limits (Retinol, Salicylic Acid, etc.)
- In the API route, load DB, match ingredients from input, inject matches into prompt
- This makes results deterministic for known ingredients, reducing hallucination risk

**4. Prompt improvement**
- Current prompt relies entirely on Claude's training knowledge
- After adding DB, change prompt to: "The following ingredients were matched in our database: [...]. Explain why each is problematic. For unmatched ingredients, flag as pending."

### P2 — User experience

**5. Proper PDF generation**
- Install `html2pdf.js` or use Puppeteer on server
- Replace current `window.print()` with proper PDF download
- Match the checklist visual style from the results screen

**6. Email capture backend**
- Create `POST /api/subscribe` endpoint
- Store email + product name + verdict in Supabase or Airtable
- Send confirmation email via Resend

**7. Deploy + custom domain**
- Deploy to Vercel
- Connect `skucheck.kr` domain (or desired domain)

### P3 — v1.0 features

**8. User auth + SKU history**
- Add Supabase Auth (email magic link or Google OAuth)
- Save each analysis run to `analyses` table: `{ user_id, product_name, country, result_json, created_at }`
- Add `/history` page listing past analyses

**9. Canada market**
- Add `ca` option to country selector
- Add Health Canada Cosmetic Ingredient Hotlist to DB
- Write `buildSystemPrompt('ca')` and update `buildPrompt()` sections for Canada

**10. Freemium gate**
- Allow 1 free analysis (track in localStorage: `skucheck_free_used`)
- On second analysis attempt, show paywall modal
- Integrate Stripe for subscription (monthly plan)

---

## Known Issues

| Issue | Impact | Fix |
|-------|--------|-----|
| API key exposed client-side | Security risk for production | P0: Add backend proxy |
| Claude may hallucinate ingredient rules | Wrong risk ratings | P1: Add deterministic ingredient DB |
| No input validation | User can submit empty form | P1: Add validation before API call |
| PDF is basic browser print | Poor formatting on some browsers | P2: Use html2pdf.js |
| Only one country analyzed per session unless both tabs clicked | UX friction | Known; acceptable for v0.1 |

---

## Disclaimer

This service provides AI-based preliminary regulatory screening only. Results have no legal validity. Final regulatory decisions require review by a qualified regulatory expert (FDA consultant, UK Responsible Person, or CPSR Safety Assessor).

---

*v0.1 prototype · Last updated: 2026.05*
