# SafeDoc — Project Context for Claude Code

## What This Project Is
Static HTML site on GitHub Pages (`www.safe-dr.com`) for SafeDoc — a service connecting travelers with local doctors worldwide. Single file: `index.html`.

## What Was Built (Doctor Registration Form)
A 4-step registration form was added in May 2026 to replace a non-functional placeholder. The form submits to a **Cloudflare Worker** which creates records in Airtable and uploads file attachments.

**Do not modify the form structure, field IDs, or JS functions without understanding the Airtable field mapping below.**

---

## Architecture

```
Browser (index.html)
  → POST multipart/form-data
  → Cloudflare Worker (safedoc-form.tal-997.workers.dev)
    → Airtable REST API (record creation + duplicate check)
    → Cloudflare R2 (file storage)
    → Airtable REST API PATCH (attach R2 URLs to record)
```

- **Worker URL:** `https://safedoc-form.tal-997.workers.dev`
- **Airtable base:** `appRhlu9LqRwjI5To`
- **Airtable table:** `tblbGe6gl8xD9Pmvo`
- **Attachment field:** `מסמכים שצורפו` (PATCH via REST API with R2 public URLs)
- **R2 bucket:** `safedoc-temp-files` (public URL: `https://pub-ad6f0437d11748eaa90f7a639f2df155.r2.dev`)
- **R2 lifecycle:** objects auto-deleted after 30 days
- **Worker secrets:** `AIRTABLE_TOKEN` | **Worker binding:** `bucket` (R2)

---

## Form Field IDs → Airtable Fields

### Text / URL / Phone (sent as FormData key)
| Form ID | Airtable field |
|---------|---------------|
| `clinic_name` | שם המרפאה |
| `clinic_name2` | שם מרפאה נוסף |
| `address` | כתובת  |
| `maps_link` | לינק גוגל מפס |
| `phone` | נייד \| וואצאפ |
| `office_phone` | טלפון משרד |
| `email` | אימייל |
| `website` | אתר |
| `hours` | שעות פעילות |
| `about_clinic` | אודות המרפאה |
| `fee_clinic` | עלות טיפול |
| `fee_hotel` | עלות הגעה למלון |
| `fee_online` | עלות ייעוץ אונליין |
| `fee_description` | עלויות לפי תיאור הרופא |
| `notes` | הערות ושאלות?:) |

### Single Selects
| Form ID | Airtable field |
|---------|---------------|
| `medical_type` | סוג רפואי |
| `hotel_visits` | מגיע למקום לינה |
| `emergency` | מוקד חירום 24-7? |
| `diagnostic` | בדיקות אבחנתיות |
| `lab` | בדיקות מעבדה? |
| `evacuations` | פינויים |
| `online_consult` | ייעוץ אונליין |
| `currency` | מטבע עלות |

### Multiple Selects (arrays — sent as repeated FormData keys)
| Form key | Airtable field |
|----------|---------------|
| `country_en` | מדינה \| אנגלית |
| `city_en` | עיר \| אנגלית |
| `specialization` | Specialization |
| `days` | ימי פעילות |
| `languages` | שפות |

### Files
| Form key | Airtable field |
|----------|---------------|
| `files` (up to 3) | מסמכים שצורפו (fldxNMOEXwKHniARG) |

---

## Required vs Optional Fields

**Required** (validated before step advance / submit):
- Step 1: country, city
- Step 2: clinic_name, medical_type, specialization chips (≥1), hours, days chips (≥1), all 6 Y/N service rows
- Step 3: phone, email, currency
- Step 4: about_clinic, languages chips (≥1)

**Optional** (no validation):
- address, maps_link, office_phone, website, clinic_name2, fee_clinic, fee_hotel, fee_online, fee_description, notes, files

---

## Key JavaScript Functions — Do Not Remove or Rename

| Function | Purpose |
|----------|---------|
| `populateCountries()` | Fills country dropdown from COUNTRY_CITIES on load |
| `filterCities()` | Filters city dropdown when country changes |
| `applyCountryMeta(country)` | Auto-fills phone dial code and currency |
| `validateStep(n)` | Returns bool, highlights invalid fields |
| `goStep(n)` | Validates current step, saves draft, navigates |
| `toggleChip(el)` | Multi-select chip toggle |
| `toggleYN(el, field)` | Radio-style yes/no chip toggle |
| `showFiles(inp)` | Shows file thumbnails in upload box |
| `saveFormState()` | Saves form to localStorage |
| `restoreFormState()` | Shows restore banner if draft exists |
| `detectBrowserLanguage()` | Pre-selects language chip from browser lang |
| `detectLocation()` | IP geolocation → pre-fills country/city/phone/currency |
| `setupLiveValidation()` | Green border on valid required fields |
| `submitForm()` | Builds FormData, POSTs to Worker, handles errors |

---

## HTML Structure (form section)

```html
<section id="join">
  <div class="fwrap">
    <div class="stepper" id="stepper">...</div>  <!-- step indicators -->
    <div class="prog">...</div>                   <!-- progress bar -->
    <div class="fstep active" id="step1">...</div>
    <div class="fstep" id="step2">...</div>
    <div class="fstep" id="step3">...</div>
    <div class="fstep" id="step4">...</div>
    <div class="success-scr" id="success-scr">...</div>
  </div>
</section>
```

---

## Bilingual Support
The site supports Hebrew (RTL) and English. Every user-visible string uses:
```html
<span data-he="">טקסט בעברית</span><span data-en="">English text</span>
```
The `setLang(lang)` function shows/hides these based on `document.body.className` (`lang-he` or `lang-en`).

---

## Cloudflare Worker
File: `safedoc-worker.js` (deployed separately — NOT auto-deployed from git).
After any change to the worker file, deploy manually via Cloudflare dashboard:
Workers → safedoc-form → Edit code → paste → Deploy.

The worker performs:
1. Email duplicate check against Airtable (returns 409 if duplicate — fail-open on API error)
2. Creates Airtable record from all text/select fields (status defaulted to "שלח לנו פרטים")
3. Uploads files to Cloudflare R2 bucket (`safedoc-temp-files`, binding name: `bucket`)
4. PATCHes the Airtable record with the R2 public URLs as attachments

Note: Airtable's `content.airtable.com` upload API does not work with PAT tokens — hence the R2 approach.

---

## What NOT to Touch Without Checking This File
- The `id` attributes of all form inputs (they are the FormData keys sent to the Worker)
- The `data-field` attributes on `.yn-chips` divs
- The `data-val` attributes on all chips
- The `COUNTRY_CITIES` and `COUNTRY_META` objects (46 countries mapped)
- The Worker URL in `submitForm()`
- The CSS classes `.fstep`, `.fg`, `.chip`, `.bnxt`, `.bbk`, `.upload`, `.yn-row`, `.yn-chips`, `.yn-label`
