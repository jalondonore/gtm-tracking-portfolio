# Project 01 — Multi-Location Form Tracking with Dynamic Event Names

> **Stack:** GTM · GA4 · Google Ads · Webflow  
> **Technique:** DOM polling · Submit event listener · Value sanitization · Dynamic event naming

---

## Problem

A multi-location dental practice used one website with a shared appointment form and a contact/get-in-touch form. Both forms contained a **location dropdown** so patients could select their preferred clinic. The forms were built in Webflow and loaded asynchronously inside a modal — meaning the form's DOM element does not exist at GTM's standard page load time.

**Why standard GTM triggers failed:**
- The form element only appears in the DOM after the user opens the modal
- GTM's Form Submission trigger couldn't find the form on page load
- The location value needed to be sanitized before use in GA4 event names

---

## Solution Architecture

Two Custom HTML tags deployed in GTM — one per form — using the pattern:

```
Page loads → GTM tag fires → polls DOM every 500ms
  → form found → submit listener attached
    → user submits → location read from dropdown
      → value sanitized → dynamic event pushed to dataLayer
        → GTM Custom Event trigger fires → GA4 + Ads tags fire
```

---

## Scripts

- [`scripts/appointment-form-tracker.js`](scripts/appointment-form-tracker.js) — Tracks the appointment request form
- [`scripts/contact-form-tracker.js`](scripts/contact-form-tracker.js) — Tracks the get-in-touch/contact form

---

## Event Schema

| Field | Value | Example |
|-------|-------|---------|
| `event` | `form_submit_Appointment_{location}` | `form_submit_Appointment_eastcobb` |
| `dlv_form_appointment_Location` | Sanitized location slug | `eastcobb` |
| `dlv_form_appointment_Location_raw` | Original dropdown value | `East Cobb` |

Contact form events follow the same schema with `form_submit_Contact_{location}`.

---

## Value Sanitization Pipeline

```
Raw dropdown value (e.g. "East Cobb")
  → strip quotes          → "East Cobb"
  → collapse whitespace   → "EastCobb"
  → remove hyphens        → "EastCobb"
  → to lowercase          → "eastcobb"
```

---

## GTM Components

| Type | Name | Role |
|------|------|------|
| Custom HTML Tag | Appointment Form | Polling + listener + dataLayer push |
| Custom HTML Tag | Contact Form | Same pattern, different IDs |
| Custom Event Trigger | Appointment Form Submission | Matches `form_submit_Appointment_*` |
| Custom Event Trigger | Contact Form Submission | Matches `form_submit_Contact_*` |
| DataLayer Variable | `dlv_form_appointment_Location` | Clean location value |
| DataLayer Variable | `dlv_form_appointment_Location_raw` | Original dropdown value |
| GA4 Event Tag | Appointment Location Submission | Passes location to GA4 |
| GA4 Event Tag | Contact Form Location Submission | Passes location to GA4 |

---

## Known Limitations & Improvements — corrected in v1.1.0

- ~~Polling interval has no maximum attempt cap~~ → `maxAttempts` guard added (60 attempts × 500ms = 30s max) ✓
- Form IDs (`wf-form-Appointment-Form`, `location-3`) are Webflow-specific — update if the form is rebuilt
