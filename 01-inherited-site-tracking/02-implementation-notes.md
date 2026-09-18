# NORTHSTAR — Inherited-Site Tracking Implementation Notes

## 1. Implementation Context

The project uses Google Tag Manager and GA4 on a React/SPA website.

The implementation was designed as an inherited-site tracking scenario. Dedicated frontend analytics instrumentation was not assumed to be available, so the tracking layer uses existing browser events, DOM structure, URLs, form states, and application behavior where they provide sufficiently reliable signals.

The general implementation pattern is:

```text
Existing browser / DOM / application fact
↓
GTM detection
↓
Semantic normalization
↓
Deduplication
↓
Internal event
↓
GA4 event
```

Where frontend instrumentation would provide a stronger or more maintainable source of truth, it is documented as the preferred production architecture.

---

## 2. Contact Form Tracking

### 2.1 `form_view`

The Contact form is identified using its stable form action:

```css
form[action="/api/public/contact"]
```

GTM Element Visibility is configured with:

```text
75% minimum visibility
1 second minimum duration
Observe DOM changes = ON
```

The technical form action is mapped to the semantic form identifier:

```text
/api/public/contact
↓
contact_main
```

The final GA4 event includes:

```text
event = form_view

form_id = contact_main
form_location = contact_page
```

This solution depends on the form action remaining stable.

---

### 2.2 `form_start`

Native focus or click events were not considered sufficient evidence of meaningful form engagement.

A document-level delegated listener monitors:

```text
input
change
```

events.

The listener identifies the Contact form and evaluates whether the interaction qualifies as a meaningful start.

Text-like fields qualify when they contain a non-whitespace value:

```text
name
email
phone
message
```

The `subject` field qualifies when it changes away from its default value:

```text
order
```

The consent checkbox alone does not qualify.

The listener pushes an internal `form_start` event only on the first meaningful interaction.

A browser-page lifecycle flag prevents repeated `form_start` events from additional qualifying interactions.

Final GA4 parameters:

```text
form_id = contact_main
form_location = contact_page
```

---

### 2.3 `form_submit`

A browser submit event is not treated as proof of successful submission.

The Contact form performs a real document navigation after successful submission:

```text
/contact
↓
successful submission
↓
/contact/success?ref=MSG-*
```

The implementation uses a two-step confirmation process.

First, a valid Contact form submission creates a temporary pending state in `sessionStorage` containing:

* form identity;
* submission timestamp.

No GA4 success event is sent at this stage.

After navigation to:

```text
/contact/success
```

GTM validates:

```text
pending submission exists
+
correct form action
+
reasonable timestamp
+
valid MSG-* reference
```

The pending state is consumed before the final success event is pushed.

This prevents:

```text
success-page reload
direct opening of an old success URL
```

from creating false conversions.

Final GA4 event:

```text
form_submit

form_id = contact_main
form_location = contact_page
```

---

## 3. Phone Tracking

Phone tracking uses two semantic parameters:

```text
phone_label
phone_location
```

Examples:

```text
sales + product_detail
general + footer
rental + rental_page
```

---

### 3.1 `phone_view`

Phone links are detected using:

```css
a[href^="tel:"]
```

with GTM Element Visibility:

```text
100% visible
1 second
Observe DOM changes = ON
```

The phone number is read from the element `href`.

A Lookup Table maps technical phone URLs to semantic labels:

```text
tel:+48555000000 → general
tel:+48555100101 → sales
tel:+48555200202 → rental
tel:+48555300303 → support
```

`phone_location` is derived from DOM and route context.

Examples:

```text
inside footer
→ footer

/product/*
→ product_detail

/services/*
→ service_detail

/rental
→ rental_page
```

The original implementation relied on:

```text
Once per element
```

but QA showed that DOM-element identity did not match the required semantic grain.

Two different DOM elements could represent:

```text
general + footer
```

and React could recreate the same semantic phone CTA after SPA navigation.

The final implementation therefore adds semantic deduplication based on:

```text
phone_label + phone_location
```

and pushes:

```text
phone_view_unique
```

Only this internal event triggers the final GA4:

```text
phone_view
```

---

### 3.2 `phone_click` — Independent GTM Fallback

The inherited-site fallback uses a GTM Just Links trigger for:

```text
tel:
```

links.

`phone_label` is derived from the clicked phone URL.

`phone_location` is derived from:

```text
Click Element
+
DOM context
+
current route
```

Semantic deduplication uses:

```text
phone_label + phone_location
```

For example:

```text
general|footer
```

is counted once during the current browser document lifecycle.

A different semantic key remains eligible:

```text
general|footer
sales|product_detail
rental|footer
```

The deduplicated internal event is:

```text
phone_click_unique
```

which triggers the final GA4 event:

```text
phone_click
```

---

### 3.3 `phone_click` — Preferred Frontend-Based Architecture

The website can also provide a frontend Data Layer event such as:

```javascript
dataLayer.push({
  event: 'phone_click',
  phone_label: 'sales',
  phone_location: 'product_detail',
  click_url: 'tel:+48555100101'
});
```

When this instrumentation is available, it is the preferred production source because the application already knows the semantic phone context.

Recommended architecture:

```text
frontend phone_click
↓
phone_label + phone_location
↓
GTM semantic deduplication
↓
phone_click_unique
↓
GA4 phone_click
```

The frontend event can represent every eligible physical click.

GTM can then apply the analytical deduplication rule separately.

This keeps the frontend responsible for the raw interaction and the analytics layer responsible for measurement grain.

---

## 4. Booking Modal Tracking

The booking funnel is:

```text
booking_cta_click
↓
form_start
↓
form_submit
```

Tracking is intentionally performed at `booking_type` level rather than individual rental or service item level.

Supported values:

```text
equipment_reservation
session_booking
```

---

### 4.1 `booking_cta_click`

Booking buttons do not have stable machine identifiers such as:

```text
id
data-*
href
```

and unrelated buttons also exist across the site.

Maintaining separate CSS selectors for every booking-button placement would be fragile.

Instead, the inherited-site implementation uses behavioral confirmation:

```text
button click
↓
temporary candidate
↓
known booking form appears shortly afterwards
↓
booking CTA confirmed
```

If no booking form appears within the allowed confirmation window, the click is ignored.

The booking category is derived from the route:

```text
/rental
→ equipment_reservation

/services
/services/*
/campaign
→ session_booking
```

Semantic deduplication uses:

```text
booking_type
```

per browser document lifecycle.

The final event is:

```text
booking_cta_click
```

with:

```text
booking_type
```

#### Limitation

Because the CTA is identified by the result of the interaction, a genuine booking CTA that is broken and fails to open the modal cannot be confirmed by this fallback implementation.

A frontend Data Layer event from the actual booking-button handler would be a stronger production source.

---

### 4.2 Booking `form_start`

The booking form is dynamically inserted into the DOM.

A delegated document-level listener is therefore used instead of attaching listeners directly to individual form fields.

The booking form is identified structurally through its expected fields:

```text
name
email
phone
date
people
notes
consent
```

This avoids dependence on translated user-facing text.

An earlier approach used modal `aria-label` values such as:

```text
Book your session
Reserve equipment
```

but localization testing showed that these values change between languages.

The final implementation therefore avoids translated `aria-label` text as a machine identifier.

Meaningful start conditions include:

```text
non-whitespace name
non-whitespace email
non-whitespace phone
non-whitespace notes
non-empty date
people changed away from default 2
```

Consent-only interaction does not qualify.

Deduplication is performed by:

```text
booking_type
```

per browser document lifecycle.

Final GA4 parameters:

```text
form_id = booking_request_modal
form_location = modal
booking_type
```

---

### 4.3 Booking `form_submit`

Testing showed that the browser `submit` event fires for both:

```text
invalid submission
valid submission
```

Therefore:

```text
submit event
≠
confirmed success
```

Successful booking replaces the form with a success state containing a unique booking reference:

```text
BK-*
```

A `MutationObserver` watches for relevant DOM changes.

The implementation confirms:

```text
known booking dialog
+
form has disappeared
+
valid BK-* reference exists
```

before pushing:

```text
booking_form_submit_success
```

Deduplication is based on the unique booking reference.

For example:

```text
BK-111111 → form_submit
BK-222222 → form_submit
```

while repeated observation of:

```text
BK-111111
```

is suppressed.

The booking reference is used internally for confirmation and deduplication and is not sent to GA4 as a reporting parameter.

Final GA4 parameters:

```text
form_id = booking_request_modal
form_location = modal
booking_type
```

---

## 5. Service-Link Tracking

The event measures navigation intent toward a specific service rather than generic service-category navigation.

Eligible destinations follow:

```text
/services/<service-slug>
```

Examples:

```text
/services/sup-lessons
/services/equipment-rental
/services/guided-trips
```

Excluded destinations include:

```text
/services
/rental
/campaign
```

A GTM Just Links trigger detects eligible links.

Instead of maintaining a Lookup Table for every service, the service slug is extracted dynamically from the clicked URL.

Examples:

```text
/services/sup-lessons
→ sup_lessons

/services/equipment-rental
→ equipment_rental

/services/guided-trips
→ guided_trips
```

URL decoration such as:

```text
?source=test
#faq
```

does not change the semantic identity.

For example:

```text
/services/sup-lessons?source=test
```

still produces:

```text
service_id = sup_lessons
```

Semantic deduplication uses:

```text
service_id
```

per browser document lifecycle.

The internal event is:

```text
service_link_click_unique
```

which triggers:

```text
service_link_click
```

with:

```text
service_id
```

Dynamic slug extraction was preferred to a Lookup Table because new `/services/<slug>` routes can be supported without adding new mappings.

---

## 6. Internal Event Layer

Several implementations use internal GTM events between raw browser/application behavior and the final GA4 event.

Examples include:

```text
phone_view_unique
phone_click_unique
booking_form_start
booking_form_submit_success
service_link_click_unique
```

The general pattern is:

```text
raw browser / frontend / DOM fact
↓
technical validation
↓
semantic normalization
↓
deduplication
↓
internal GTM event
↓
GA4 event
```

The internal event name does not have to match the final GA4 event name.

This separation helps avoid event-name collisions and makes semantic validation and deduplication easier to manage.

---

## 7. State and Deduplication Mechanisms

Different technical mechanisms are used depending on the required event grain.

Examples include browser-level objects such as:

```text
window.__nsPhoneViewSeen
window.__nsPhoneClickSeen
window.__nsBookingFormStarted
window.__nsBookingSubmitSeen
window.__nsServiceClickSeen
```

These store semantic keys during the current browser document lifecycle.

The Contact submission flow uses:

```text
sessionStorage
```

because the pending submission state must survive the real navigation from:

```text
/contact
```

to:

```text
/contact/success
```

These are implementation mechanisms only.

The actual semantic measurement grain is defined separately in `01-tracking-spec.md`.

---

## 8. Architecture Boundary

GTM and browser-side inference were used where the required fact already existed in a reasonably stable and maintainable form.

Examples include:

```text
tel: URL
service URL slug
stable form action
confirmed success DOM state
two stable booking categories
```

Frontend changes are recommended when GTM would otherwise need to reconstruct application business state.

For example, this implementation tracks:

```text
booking_type
```

but intentionally does not attempt item-level attribution such as:

```text
booking_item_id
```

An item-level booking funnel would require the selected item identity to remain available across:

```text
CTA click
↓
modal
↓
form_start
↓
successful submission
```

That state is better owned and exposed by the application itself.

A preferred frontend implementation could provide:

```javascript
dataLayer.push({
  event: 'booking_cta_click',
  booking_type: 'equipment_reservation',
  booking_item_id: 'loon_sup_set'
});
```

and preserve the same item identity through confirmed booking success.

The architectural rule used in this project is:

> **GTM should consume and normalize application facts. It should not become a second frontend application responsible for recreating complex business state.**
