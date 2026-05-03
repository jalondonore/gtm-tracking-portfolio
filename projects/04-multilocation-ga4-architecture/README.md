# Project 04 — Multi-Location GA4 + Google Ads Architecture

> **Stack:** GTM · GA4 · Google Ads · Microsoft Clarity  
> **Technique:** URL-based trigger segmentation · Per-location conversion tags · Enhanced Conversions

---

## Architecture Diagram

![GTM multi-location architecture](gtm-architecture-diagram.svg)

---

## Problem

A dental group with multiple clinic locations needed to attribute digital conversions (phone calls, appointment bookings, map clicks, SMS text clicks) to each individual clinic for Google Ads bid optimization. All locations shared one website and one GTM container.

**Requirements:**
- Separate Google Ads conversion actions per location and per action type
- GA4 events with location context for audience segmentation
- No custom JavaScript — only native GTM trigger conditions

---

## Solution Architecture

The website used distinct URL path segments per location (e.g., `/location-a/`, `/location-b/`, `/location-c/`). GTM's built-in **Link Click** triggers were configured with **compound conditions** combining element attributes with page URL path matching.

**Trigger condition pattern:**

```
Click URL starts with "tel:"         → phone call
  AND Page URL contains "/location-a/"  → scoped to Location A
```

```
Click URL contains "booking-platform.com"  → appointment scheduler
  AND Page URL contains "/location-b/"        → scoped to Location B
```

This creates a full **3 locations × 4 action types** matrix of triggers and tags.

---

## Action Types Tracked

| Action | Trigger Type | Condition |
|--------|-------------|-----------|
| Phone call click | Link Click | `href` starts with `tel:` |
| Appointment scheduler click | Link Click | `href` contains booking platform URL |
| Google Maps / Get Directions click | Link Click | `href` contains `maps.google.com` |
| SMS text click | Link Click | `href` starts with `sms:` |

---

## GTM Components

### Shared (fire on all locations)

| Type | Name | Role |
|------|------|------|
| GA4 Config Tag | GA4 Configuration | Single GA4 measurement ID for all locations |
| Conversion Linker | Conversion Linker Tag | Preserves click IDs across redirects |
| Custom HTML Tag | Microsoft Clarity | Session recording + heatmaps (all pages) |
| Variable | Enhanced Conversions | Hashed user data for Google Ads |

### Per-Location Tags (example: one location)

| Type | Name | Role |
|------|------|------|
| Link Click Trigger | Phone Call Click — Location A | `tel:` + page URL contains `/location-a/` |
| Link Click Trigger | Appointment Click — Location A | booking URL + page URL contains `/location-a/` |
| Link Click Trigger | Maps Click — Location A | `maps.google` + page URL contains `/location-a/` |
| Link Click Trigger | SMS Click — Location A | `sms:` + page URL contains `/location-a/` |
| Google Ads Conversion | Phone Call Click — Location A | Fires on phone trigger, Location A only |
| Google Ads Conversion | Appointment Click — Location A | Fires on appointment trigger, Location A only |
| Google Ads Conversion | Maps Click — Location A | Fires on maps trigger, Location A only |
| GA4 Event Tag | Phone Call Click — Location A | GA4 event with location label |
| GA4 Event Tag | Appointment Click — Location A | GA4 event with location label |

*(Pattern repeats for each location)*

---

## GA4 Event Schema

All location-specific GA4 events follow this pattern:

| Event Name | Description |
|------------|-------------|
| `click_call_[location]` | Phone number click from that location's page |
| `click_appointment_[location]` | Booking scheduler click from that location's page |
| `click_maps_[location]` | Get directions click from that location's page |
| `click_sms_[location]` | SMS text button click from that location's page |
| `click_schedule_online` | Generic schedule online button (any page) |

---

## Key Design Decisions

- **One GTM container for all locations** — simplifies maintenance; location segmentation is handled entirely through trigger conditions
- **Separate Google Ads conversion actions per location** — allows Google Ads to optimize bids independently for each clinic
- **Microsoft Clarity on all pages** — session recordings are not segmented by location; heatmaps are viewed per page URL in the Clarity dashboard
- **Enhanced Conversions variable** — captures hashed email/phone at conversion time for improved Google Ads match rates

---

## Known Limitations

- If a location-specific phone number or booking link appears in the site-wide footer, it will match the trigger for *all* location pages — add a negative URL condition if needed
- SMS clicks (`sms:`) have lower browser support on desktop; trigger fires on mobile only in most cases
- Location slug must remain stable in the URL — if the site is restructured (e.g., `/location-a/` renamed), all corresponding trigger conditions must be updated manually
