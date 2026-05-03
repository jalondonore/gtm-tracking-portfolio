# Project 02 — SmileVirtual SPA Step Tracking (v1)

> **Stack:** GTM · GA4 · Meta Pixel · Next.js (React) SPA  
> **Technique:** MutationObserver · switch/case · Boolean flag deduplication

---

## Problem

SmileVirtual is a third-party virtual consultation booking platform rendered as a standalone **Next.js (React) SPA** hosted on the vendor's domain. Dental clinics link patients to this form as their virtual consultation flow.

The form has **3 steps** that render dynamically — no URL changes occur between steps. Standard GTM click or visibility triggers cannot detect step transitions in a React SPA.

**Root causes:**
- React's virtual DOM updates the page without triggering browser navigation events
- The step indicator (`STEP 1`, `STEP 2`, `STEP 3`) is a text node inside a deeply nested element
- GTM has no native hook into React's render cycle

---

## Solution Architecture

```
GTM tag fires on SmileVirtual page load
  → 1s delay (wait for React initial render)
    → MutationObserver starts watching document.body
      → React updates DOM on step change
        → observer callback fires → detectSteps() runs
          → step text found → boolean flag checked (has this step fired already?)
            → NO: push formStep event to dataLayer + set flag to true
            → YES: skip (deduplication)
              → GTM Custom Event trigger fires → GA4 + Meta Pixel tags fire
```

---

## Script

- [`scripts/smilevirtual-step-tracker-v1.js`](scripts/smilevirtual-step-tracker-v1.js)

---

## Event Schema

| Field | Value |
|-------|-------|
| `event` | `formStep` |
| `formStepText` | `step1` / `step2` / `step3` |

---

## GTM Components

| Type | Name | Role |
|------|------|------|
| Custom HTML Tag | SmileVirtual Form Track | Injects observer script |
| Custom Event Trigger | SmileDental Form step 2 | Fires when `formStepText = step2` |
| Custom Event Trigger | SmileDental Form step 3 | Fires when `formStepText = step3` |
| GA4 Event Tag | SmileVirtual Consultation Form Step 2 | Logs step 2 to GA4 |
| GA4 Event Tag | SmileVirtual Consultation Form Step 3 | Logs step 3 to GA4 |
| GA4 Event Tag | SmileVirtual Consultation Form Submit | Fires on confirmation page view |
| Meta Pixel Tag | Lead_Formsubmit_smilevirtual | Fires on confirmation page (Pixel custom event) |

---

## Known Limitations & Improvements (original) — corrected in v1.1.0

The original v1 script had four issues, all fixed in the current file:

- ~~Global variable scope~~ → wrapped in IIFE ✓
- ~~MutationObserver runs indefinitely~~ → `observer.disconnect()` called after STEP 3 ✓
- JSX hash class selectors still present — fragile to platform updates → see v2 for the fix
- Boolean flags still present — back-navigation won't re-fire step events → see v2 for the fix
