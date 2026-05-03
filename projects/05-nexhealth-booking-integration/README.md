# Project 05 — NexHealth Third-Party Booking Widget Integration

> **Stack:** GTM · GA4 · NexHealth  
> **Technique:** Third-party dataLayer event consumption · URL parameter variables · Smart matching macros

---

## Integration Flow

![NexHealth GTM integration flow](nexhealth-flow-diagram.svg)

---

## Problem

NexHealth is a dental practice management platform that provides an **online booking widget** embedded on clinic websites. The widget is fully controlled by NexHealth — no access to its source code, no ability to add GTM directly to it.

NexHealth's widget does, however, push structured data to `window.dataLayer` when patients complete booking actions. The challenge was to:

1. Capture those third-party dataLayer events in GTM
2. Extract the booking metadata fields as GTM variables
3. Enrich the events with campaign attribution data from URL parameters
4. Distinguish between different types of patient arrival (direct link vs. email campaign vs. one-click booking link)

---

## How NexHealth Pushes Data to the dataLayer

When a booking action occurs, the NexHealth widget fires an event to `window.dataLayer` with the following structure:

```javascript
window.dataLayer.push({
  event: '<nh_event_name>',
  location_id: '<clinic_location_id>',
  patient_type: '<new|existing>',
  has_insurance: '<true|false>',
  booking_for: '<self|dependent>',
  appointment_type: '<appointment_type_label>'
});
```

GTM listens for this event via a Custom Event trigger configured to match the NexHealth event name.

---

## Solution Architecture

```
Patient arrives on booking page
  → URL parameters captured by GTM URL variables (ct, cid, cm, nh, ocb)
    → NexHealth widget loads and patient completes booking
      → NexHealth pushes dataLayer event
        → GTM Custom Event trigger fires
          → DataLayer variables read booking metadata
          → Smart matching variables evaluate link type
            → GA4 Event tag fires with all enriched parameters
```

---

## GTM Variables

### DataLayer Variables — NexHealth Booking Payload

| GTM Variable Name | dataLayer Key | Captured Value |
|-------------------|--------------|----------------|
| NH Event | `event` | Event name fired by NexHealth |
| Location ID | `location_id` | Clinic location identifier |
| Patient Type | `patient_type` | `new` or `existing` |
| Has Insurance | `has_insurance` | `true` or `false` |
| Booking For | `booking_for` | `self` or dependent |
| Appointment Type | `appointment_type` | Service/appointment label |

### URL Parameter Variables — Campaign Attribution

| GTM Variable Name | URL Parameter | Purpose |
|-------------------|--------------|---------|
| Raw URL Param - nh | `?nh=` | Identifies NexHealth-generated links |
| Raw URL Param - ocb | `?ocb=` | Identifies one-click booking links |
| Campaign Type | `?ct=` | Ad platform campaign type |
| Campaign ID | `?cid=` | Campaign identifier for reporting |
| Campaign Medium | `?cm=` | Channel or medium (email, paid, etc.) |

### Smart Matching Variables (JavaScript Macro)

These variables evaluate the URL parameters and return a boolean result:

**Is NH-Generated Link**
```javascript
// Returns true if the ?nh= URL parameter is present, false otherwise
// Used to distinguish NexHealth-generated email/SMS links from organic visits
```

**Is One-Click Booking Link**
```javascript
// Returns true if the ?ocb= URL parameter is present, false otherwise
// One-click booking links pre-fill patient data; flagging them separately
// allows segmentation in GA4 between assisted and unassisted bookings
```

---

## GTM Components

| Type | Name | Role |
|------|------|------|
| Custom Event Trigger | NH Online Booking Event | Fires on NexHealth dataLayer push |
| DataLayer Variable ×6 | (see table above) | Captures booking payload fields |
| URL Variable ×5 | (see table above) | Captures campaign attribution |
| JavaScript Macro | Is NH-Generated Link | Evaluates `?nh=` presence |
| JavaScript Macro | Is One-Click Booking Link | Evaluates `?ocb=` presence |

---

## Key Design Decisions

- **No modification to NexHealth code** — integration is 100% read-only, consuming the dataLayer events NexHealth publishes
- **URL params captured at page load** — GTM reads them from the URL variable before any navigation occurs, ensuring they're available when the booking event fires later in the session
- **Smart matching macros use default value `false`** — if the URL param is absent, the variable safely returns `false` rather than `undefined`, preventing errors downstream in tag conditions

---

## Maintenance Notes

- NexHealth may update their dataLayer event schema (field names, event name) between platform versions without prior notice — validate variables after any NexHealth release
- The `?nh=` and `?ocb=` parameters are appended by NexHealth's link generation tools — coordinate with the clinic's NexHealth account manager if these are not appearing in attribution URLs
- If GA4 reports show `undefined` for any booking variable after a NexHealth update, use GTM Preview mode to inspect the raw dataLayer push and compare field names against the variable configuration
