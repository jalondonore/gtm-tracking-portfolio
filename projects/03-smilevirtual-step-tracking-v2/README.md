# Project 03 — SmileVirtual SPA Step Tracking (v2)

> **Stack:** GTM · GA4 · Google Ads · Next.js (React) SPA  
> **Technique:** MutationObserver · Semantic CSS selectors · `lastStep` string deduplication · IIFE scope isolation

---

## Problem

Same SmileVirtual SPA tracking challenge as Project 02, implemented for a second dental clinic. The v1 approach was revisited to address three specific weaknesses:

1. **Selector fragility** — v1 used JSX hash class names (`jsx-4241655398`) that break on any Next.js rebuild
2. **One-way deduplication** — v1's boolean flags prevent re-firing if a user navigates backward through the form, causing missing step 2 events on users who correct their input
3. **No step title capture** — v1 only recorded the step number; no context about what question the user was on

---

## Improvements Over v1

| Aspect | v1 | v2 |
|--------|----|----|
| Selector strategy | Full path with JSX hash classes | Semantic utility classes only |
| Deduplication | Boolean flags — one-directional | `lastStep` string — bidirectional |
| Step title | Not captured | Captured via adjacent sibling selector |
| Scope isolation | Global variables (risk of collision) | IIFE — fully isolated |
| Initial render wait | 1000ms | 500ms |
| Observer disconnect | Never | Possible — add when `lastStep === 'STEP 3'` |

---

## Solution Architecture

```
GTM tag fires on SmileVirtual page load (IIFE executes)
  → lastStep initialized to null
    → MutationObserver set on document.body
    → detectStep() runs immediately (500ms delay)
      → React updates DOM on step change → observer fires
        → detectStep() reads step label + title text
          → stepText !== lastStep? (has this step changed?)
            → YES: push formStep event + update lastStep
            → NO: skip
              → GTM trigger fires → GA4 + Google Ads tags fire
```

---

## Script

- [`scripts/smilevirtual-step-tracker-v2.js`](scripts/smilevirtual-step-tracker-v2.js)

---

## Event Schema

| Field | Value | Example |
|-------|-------|---------|
| `event` | `formStep` | `formStep` |
| `formStepText` | Step label text | `STEP 1` / `STEP 2` / `STEP 3` |
| `dl_formStepTitle` | Step title / heading text | `Tell us about yourself` |

---

## GTM Components

| Type | Name | Role |
|------|------|------|
| Custom HTML Tag | SmileVirtual Form Track | Injects observer script (v2) |
| DataLayer Variable | `dl_formStep` | Reads `formStepText` from dataLayer |
| Custom Event Trigger | SmileDental Form step 2 | Fires when `dl_formStep = STEP 2` |
| Custom Event Trigger | SmileDental Form step 3 | Fires when `dl_formStep = STEP 3` |
| Custom Event Trigger | SmileVirtual started filling | Fires when user begins form |
| GA4 Event Tag | SmileVirtual Form Step 2 | Logs step 2 + title to GA4 |
| GA4 Event Tag | SmileVirtual Form Step 3 | Logs step 3 + title to GA4 |
| GA4 Event Tag | SmileVirtual Form Submit | Fires on confirmation page view |
| GA4 Event Tag | SmileVirtualForm - Started | Fires on first form interaction |
| GA4 Event Tag | SmileVirtual Page View | Fires on SPA page view |
| Google Ads Tag | BOOK NOW Click | Tracks initial CTA entry point |

---

## Remaining Improvement — corrected in current file

- ~~MutationObserver runs indefinitely~~ → `observer.disconnect()` called when `lastStep === 'STEP 3'` ✓
