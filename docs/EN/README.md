# GTM Tracking Portfolio — Full Documentation (English)

> **Context:** All projects were implemented for multi-location dental clinic groups. The core challenge in each project was capturing form submission and interaction events from pages and widgets that live outside the client's own domain or are rendered dynamically by third-party JavaScript frameworks, making standard GTM triggers unreliable or unavailable.

---

## Project 01 — Multi-Location Form Tracking with Dynamic Event Names

### Business Context

A dental group operating multiple clinic locations used a single website with a shared appointment form and a contact form. Both forms included a **location selector dropdown**, allowing patients to choose their preferred clinic before submitting. The goal was to track which location each form submission came from, so that Google Ads and GA4 reports could be segmented by clinic location without creating separate properties or views.

### Technical Challenge

The forms were built in Webflow and rendered inside a **modal that loads asynchronously** — meaning the form's DOM element does not exist at GTM's initial page load. Standard GTM Form Submission and Click triggers could not reliably target the form because:

1. The form element (`#wf-form-Appointment-Form`) is injected into the DOM only when the user opens the modal
2. The submit button click fires before the form data is available for reading
3. The location dropdown value needed to be sanitized before use in an event name

### Solution: DOM Polling + Submit Listener + Value Sanitization

Two Custom HTML tags were deployed in GTM — one for the appointment form, one for the contact/get-in-touch form. Both follow the same pattern.

---

#### Script: Appointment Form Tracker

```javascript
(function() {

  // STEP 1 — Define the form initialization function
  // This is called only after the form is confirmed to exist in the DOM
  function initFormTracking() {
    var form = document.getElementById('wf-form-Appointment-Form');
    if (!form) return;

    // STEP 2 — Attach a submit event listener to the form element
    // This fires when the user submits the form, before the page navigates
    form.addEventListener('submit', function(e) {

      // STEP 3 — Read the selected location from the dropdown
      var locationSelect = document.getElementById('Location');
      if (!locationSelect) return;

      var rawCity = locationSelect.value || 'unknown';

      // STEP 4 — Sanitize the raw dropdown value
      // Remove quotes, collapse whitespace, strip hyphens, convert to lowercase
      // Result: "East Cobb" → "eastcobb", "Johns-Creek" → "johnscreek"
      var cleanCity = rawCity
        .replace(/['"]/g, '')    // remove single and double quotes
        .replace(/\s+/g, '')     // remove all spaces
        .replace(/-/g, '')       // remove hyphens
        .toLowerCase();          // normalize to lowercase

      // STEP 5 — Build a dynamic event name using the sanitized location
      // Pattern: form_submit_Appointment_{location}
      // Example: form_submit_Appointment_eastcobb
      var eventName = 'form_submit_Appointment_' + cleanCity;

      // STEP 6 — Push the event to the dataLayer
      // Two variables are sent: the clean value (for event naming) and the raw
      // value (for debugging or fallback display purposes)
      window.dataLayer = window.dataLayer || [];
      window.dataLayer.push({
        event: eventName,
        dlv_form_appointment_Location: cleanCity,
        dlv_form_appointment_Location_raw: rawCity
      });
    });
  }

  // STEP 7 — Poll the DOM every 500ms until the form element appears
  // This handles the case where the form is inside a modal or is
  // lazy-loaded after the initial page render
  var checkInterval = setInterval(function() {
    var form = document.getElementById('wf-form-Appointment-Form');
    if (form) {
      clearInterval(checkInterval); // stop polling once found
      initFormTracking();           // initialize tracking
    }
  }, 500);

})();
```

**How it connects to GTM:**
- A Custom Event trigger in GTM listens for events matching `form_submit_Appointment_.*` (regex) or each specific event name
- A GA4 Event tag fires on that trigger, passing `dlv_form_appointment_Location` as an event parameter
- Google Ads conversion tags can be scoped per location using separate triggers for each event name variant

---

#### Script: Contact Form Tracker

Identical pattern to the appointment form tracker. Differences:
- Targets `#wf-form-Contact-Form` and dropdown `#location-3`
- Event name pattern: `form_submit_Contact_{location}`

---

### GTM Configuration Summary

| Component | Name | Purpose |
|-----------|------|---------|
| Custom HTML Tag | Appointment Form | Polls for form, attaches listener, pushes dynamic event |
| Custom HTML Tag | Contact Form | Same pattern for the contact/get-in-touch form |
| Custom Event Trigger | Appointment Form Submission | Listens for `form_submit_Appointment_*` events |
| Custom Event Trigger | Contact Form Submission | Listens for `form_submit_Contact_*` events |
| DataLayer Variable | `dlv_form_appointment_Location` | Captures clean location value |
| DataLayer Variable | `dlv_form_appointment_Location_raw` | Captures original dropdown value |
| GA4 Event Tag | GA4 - Appointment Location Submission | Fires on appointment submission, passes location |
| GA4 Event Tag | GA4 - Contact Form Location Submission | Fires on contact submission, passes location |

---

---

## Project 02 — SmileVirtual SPA Step Tracking (v1 — Boolean Flags)

### Business Context

SmileVirtual is a third-party virtual consultation booking platform rendered as a **standalone Next.js (React) application** hosted on the provider's own domain. Dental clinics embed or link to this form as their primary virtual consultation flow. The form is multi-step (3 steps), and each step renders different content dynamically without a traditional page navigation.

The goal was to track **how far users progressed** through the 3-step form, enabling funnel analysis in GA4 and retargeting triggers in Meta Pixel.

### Technical Challenge

Because the SmileVirtual form is a React SPA:
- There are **no page views** between steps — the URL does not change
- Standard GTM visibility or click triggers cannot detect the step changes
- The DOM is manipulated by React's virtual DOM reconciliation — elements appear and disappear without traditional load events
- The step indicator text (`STEP 1`, `STEP 2`, `STEP 3`) is rendered inside deeply nested elements with dynamically generated CSS class names (JSX hash classes)

### Solution: MutationObserver + Switch/Case with Boolean Flags

```javascript
// STEP 1 — Declare boolean flags to track which steps have already been fired
// This prevents duplicate events if the observer triggers multiple times
// on the same step (React can re-render without actually changing the step)
var step1 = false;
var step2 = false;
var step3 = false;

// STEP 2 — Define the step detection function
function detectSteps() {

  // STEP 3 — Define multiple CSS selectors to target the step indicator element
  // Two selectors are provided because the SPA renders different layouts
  // depending on device/breakpoint — both point to the element that
  // displays the text "STEP 1", "STEP 2", or "STEP 3"
  var selectors = [
    '#__next > div > main > form > div > div.sign-up-column > div.sign-up-content.my-auto > div.text-center > small',
    '#__next > div > main > form > div > div:nth-child(1) > div.sign-up-content > div > p'
  ];

  // STEP 4 — Try each selector until one returns a visible element
  for (var i = 0; i < selectors.length; i++) {
    var el = document.querySelector(selectors[i]);
    if (el) {
      var text = el.innerText.trim();

      // STEP 5 — Evaluate which step is visible and push to dataLayer
      // Boolean flags ensure each step event fires only once per session
      switch (text) {
        case 'STEP 1':
          if (step1 === false) {
            window.dataLayer = window.dataLayer || [];
            window.dataLayer.push({
              event: 'formStep',
              formStepText: 'step1'
            });
            step1 = true; // mark as fired, will not fire again
          }
          break;
        case 'STEP 2':
          if (step2 === false) {
            window.dataLayer = window.dataLayer || [];
            window.dataLayer.push({
              event: 'formStep',
              formStepText: 'step2'
            });
            step2 = true;
          }
          break;
        case 'STEP 3':
          if (step3 === false) {
            window.dataLayer = window.dataLayer || [];
            window.dataLayer.push({
              event: 'formStep',
              formStepText: 'step3'
            });
            step3 = true;
          }
          break;
      }
    }
  }
}

// STEP 6 — Set up a MutationObserver on document.body
// This watches for ANY DOM change (child additions, subtree mutations)
// and calls detectSteps() each time React updates the DOM
var observer = new MutationObserver(function() {
  detectSteps();
});

// STEP 7 — Start observing after a 1-second delay
// The delay ensures React has finished its initial render before
// the observer begins watching for changes
setTimeout(function() {
  observer.observe(document.body, {
    childList: true,  // watch for added/removed child nodes
    subtree: true     // watch all descendants, not just direct children
  });
  detectSteps(); // also run once immediately in case we're already on a step
}, 1000);
```

**How it connects to GTM:**
- A Custom Event trigger listens for `formStep` where `formStepText` equals `step2`
- A separate trigger for `step3`
- GA4 Event tags fire on each, logging the funnel stage

---

---

## Project 03 — SmileVirtual SPA Step Tracking (v2 — CSS Selector + Title Capture)

### Business Context

A second dental clinic implemented the same SmileVirtual integration but required a more resilient and informative tracking script. The v1 approach had two limitations in this context:

1. The deeply nested JSX hash selectors were fragile — a platform update could break them
2. No form step title was captured, limiting the usefulness of the funnel data

### Solution: Semantic CSS Selectors + Step Title + `lastStep` Deduplication

```javascript
(function() {

  // STEP 1 — Track the last seen step to avoid firing duplicate events
  // Unlike v1's boolean flags, this approach allows re-detection if a
  // user navigates backward and then forward through the form
  var lastStep = null;

  function detectStep() {

    // STEP 2 — Use a semantic, framework-agnostic CSS selector
    // Instead of a full deep path with JSX-generated class hashes,
    // this targets any <p> inside the form with those three stable utility classes
    // These classes are unlikely to change with minor React updates
    var stepEl = document.querySelector(
      '#__next form p.text-primary.text-uppercase.font-weight-bold'
    );

    // STEP 3 — Also capture the step's title (h1 element following the step label)
    // The CSS adjacent sibling combinator (+) selects the h1 that immediately
    // follows the step label paragraph
    var titleEl = document.querySelector(
      '#__next form p.text-primary.text-uppercase.font-weight-bold + h1'
    );

    // STEP 4 — Bail out if either element is not found in the current DOM state
    if (!stepEl || !titleEl) return;

    var stepText = stepEl.innerText.trim();   // e.g. "STEP 1", "STEP 2"
    var titleText = titleEl.innerText.trim(); // e.g. "Tell us about yourself"

    // STEP 5 — Only fire if the step has actually changed since last detection
    // This is the deduplication gate — prevents redundant pushes on
    // React re-renders that don't change the visible step
    if (stepText && stepText !== lastStep) {

      // STEP 6 — Push step and title to the dataLayer
      window.dataLayer = window.dataLayer || [];
      window.dataLayer.push({
        event: 'formStep',
        formStepText: stepText,       // "STEP 1", "STEP 2", "STEP 3"
        dl_formStepTitle: titleText   // human-readable title of the current step
      });

      lastStep = stepText; // update the reference for next comparison
    }
  }

  // STEP 7 — Observe DOM mutations (same as v1, but wrapped in IIFE for scope safety)
  var observer = new MutationObserver(detectStep);
  observer.observe(document.body, {
    childList: true,
    subtree: true
  });

  // STEP 8 — Run once immediately with a small delay for initial render
  setTimeout(detectStep, 500);

})();
```

**Key improvements over v1:**

| Aspect | v1 | v2 |
|--------|----|----|
| Selector strategy | Full deep path with JSX class hashes | Semantic utility classes only |
| Deduplication | Boolean flags (one-way) | `lastStep` string comparison (allows back-navigation) |
| Step title | Not captured | Captured in `dl_formStepTitle` |
| Scope isolation | Global variables | IIFE — all variables are scoped |
| Initial render wait | 1000ms timeout | 500ms timeout |

---

---

## Project 04 — Multi-Location GA4 + Google Ads Architecture (No Custom JS)

### Business Context

A dental group with three clinic locations (served by a single GTM container) needed each location's page to fire **separate Google Ads conversion tags** for phone calls, appointment clicks, map/directions clicks, and SMS text button clicks — without writing any custom JavaScript.

### Approach: URL-Based Trigger Segmentation

The website used distinct URL paths per location (e.g., `/location-a/`, `/location-b/`). GTM's built-in **Link Click** and **Click** triggers were configured with URL path conditions, creating a matrix of triggers and tags:

**Trigger pattern per location:**
- Phone call link click → matches `href` starting with `tel:` AND page URL contains `/[location-slug]/`
- Appointment scheduler link click → matches `href` containing the booking platform URL AND page URL contains `/[location-slug]/`
- Google Maps click → matches `href` containing `maps.google.com` AND page URL contains `/[location-slug]/`
- SMS text click → matches `href` starting with `sms:` AND page URL contains `/[location-slug]/`

**Tag matrix (3 locations × 4 action types = 12 Google Ads conversion tags + 12 GA4 event tags)**

### GTM Architecture Highlights

- All 3 locations share one GA4 Configuration tag and one Conversion Linker
- `Enhanced Conversions` variable (`awec`) collects hashed user-provided data at conversion time
- Google Ads conversion actions are defined per location to allow bid strategy optimization at the location level
- Microsoft Clarity is loaded on all pages for session recording and heatmaps

---

---

## Project 05 — NexHealth Third-Party Booking Widget Integration

### Business Context

NexHealth is a dental practice management platform that provides an **online booking widget** embedded on clinic websites via an iframe or a hosted page. The widget is fully controlled by NexHealth and pushes its own events to `window.dataLayer` when patients complete booking steps.

The goal was to capture these events in GTM without modifying NexHealth's code, and to enrich them with campaign attribution data from URL parameters.

### How NexHealth Pushes Data

NexHealth's widget fires a custom `dataLayer` event when a booking action occurs. The event payload includes fields such as:

```javascript
window.dataLayer.push({
  event: 'nh_booking_event',     // or equivalent event name
  location_id: '...',            // which clinic location
  patient_type: '...',           // new or existing patient
  has_insurance: '...',          // insurance flag
  booking_for: '...',            // self or someone else
  appointment_type: '...'        // type of appointment selected
});
```

### GTM Configuration

**Custom Event Trigger:**  
Fires when the NexHealth booking event is detected in the dataLayer.

**DataLayer Variables (capture NexHealth payload fields):**

| Variable Name | dataLayer Key | Purpose |
|---------------|---------------|---------|
| NH Event | `event` | Identifies the event type |
| Location ID | `location_id` | Which clinic location was booked |
| Patient Type | `patient_type` | New vs existing patient |
| Has Insurance | `has_insurance` | Boolean for insurance filter |
| Booking For | `booking_for` | Self or dependent |
| Appointment Type | `appointment_type` | Type of service selected |

**URL Parameter Variables (capture campaign attribution):**

| Variable Name | URL Parameter | Purpose |
|---------------|--------------|---------|
| Raw URL Param - nh | `?nh=` | NexHealth-generated link identifier |
| Raw URL Param - ocb | `?ocb=` | One-click booking flag |
| Campaign Type | `?ct=` | Campaign type from ad platform |
| Campaign ID | `?cid=` | Campaign identifier |
| Campaign Medium | `?cm=` | Channel/medium |

**Smart Matching Variables:**

Two JavaScript Macro variables evaluate whether the user arrived via a special link:

- **Is NH-Generated Link:** Returns `true` if the `?nh=` parameter is present, `false` otherwise
- **Is One-Click Booking Link:** Returns `true` if the `?ocb=` parameter is present, `false` otherwise

These allow GA4 and Ads tags to segment bookings by the type of link the patient used to arrive at the booking page — distinguishing organic visits from email campaign links from one-click booking links.

---

*Best practices and recommendations → [best-practices.md](best-practices.md)*
