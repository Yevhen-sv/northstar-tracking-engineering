# NORTHSTAR — Inherited-Site Tracking Specification

## 1. Scope

This specification defines the measurement rules for the inherited-site implementation of the NORTHSTAR React/SPA website.

The goal is to define what should be measured, under which conditions an event is considered valid, which parameters are required, and how duplicate analytical events should be prevented.

Implementation details are documented separately in `02-implementation-notes.md`.

---

## 2. Measurement Goals

The tracking implementation should answer the following questions:

1. What percentage of Contact-page visitors meaningfully see the contact form?
2. What percentage begin entering data into the contact form?
3. What percentage successfully submit the contact form?
4. What percentage of users meaningfully see phone CTAs?
5. What percentage activate phone CTAs?
6. What percentage enter the booking flow?
7. What percentage begin interacting with the booking form?
8. What percentage successfully submit a booking request?
9. Which specific service pages receive internal navigation clicks?

The specification separates:

```text
Business identity
↓
Context
↓
Raw technical evidence
```

The analytical event should represent the business interaction rather than the technical DOM event that happened to detect it.

---

## 3. Event Specification

### 3.1 Contact `form_view`

**Business meaning**

The Contact form received meaningful viewport exposure.

**Trigger condition**

The Contact form is at least:

```text
75% visible
for at least 1 second
```

**Required parameters**

```text
form_id = contact_main
form_location = contact_page
```

**Measurement grain**

Once per Contact form per browser-page lifecycle.

**Do not fire when**

* visibility remains below 75%;
* visibility reaches 75% but lasts less than 1 second;
* the same form has already qualified during the same browser-page lifecycle.

---

### 3.2 Contact `form_start`

**Business meaning**

The user began meaningful data entry in the Contact form.

**Trigger conditions**

The first qualifying interaction is one of:

```text
name
email
phone
message
```

containing at least one non-whitespace character,

or:

```text
subject
```

changing away from its default value:

```text
order
```

**Required parameters**

```text
form_id = contact_main
form_location = contact_page
```

**Measurement grain**

Once per Contact form per browser-page lifecycle.

**Do not fire when**

* the user only focuses a field;
* the user enters whitespace only;
* the user only interacts with the consent checkbox;
* the subject field is opened but its value does not change;
* another qualifying interaction occurs after `form_start` has already been recorded.

Input validity is not required for `form_start`.

For example, an incomplete email value can still represent meaningful form engagement.

---

### 3.3 Contact `form_submit`

**Business meaning**

A Contact form submission was successfully confirmed.

**Trigger condition**

A valid pending Contact submission is correlated with:

```text
/contact/success
```

and a valid confirmation reference matching:

```text
MSG-*
```

**Required parameters**

```text
form_id = contact_main
form_location = contact_page
```

**Measurement grain**

One event per confirmed successful pending Contact submission.

**Do not fire when**

* the form submission is invalid;
* a submit attempt does not lead to confirmed success;
* an unrelated form is submitted;
* an old success URL is opened directly;
* the success page is reloaded without a new pending submission.

A new valid submission creates a new eligible business occurrence.

---

### 3.4 `phone_view`

**Business meaning**

A phone CTA received meaningful viewport exposure.

**Trigger condition**

An eligible phone link is:

```text
100% visible
for at least 1 second
```

**Required parameters**

```text
phone_label
phone_location
```

Supported semantic phone labels include:

```text
general
sales
rental
support
```

Example locations include:

```text
footer
product_detail
rental_page
contact_page
service_detail
```

**Measurement grain**

Once per unique:

```text
phone_label + phone_location
```

per browser document lifecycle.

**Do not fire when**

* visibility remains below 100%;
* visibility lasts less than 1 second;
* the same semantic phone key has already been recorded during the current browser document lifecycle.

Different physical DOM elements representing the same semantic phone CTA must not create additional analytical events.

---

### 3.5 `phone_click`

**Business meaning**

The user intentionally activated a phone CTA.

**Trigger condition**

An eligible:

```text
tel:
```

link is activated.

**Required parameters**

```text
phone_label
phone_location
```

**Measurement grain**

Once per unique:

```text
phone_label + phone_location
```

per browser document lifecycle.

**Do not fire when**

* a non-phone link is activated;
* the same semantic phone key has already been recorded during the current browser document lifecycle.

A different phone label or location represents a different eligible semantic occurrence.

---

### 3.6 `booking_cta_click`

**Business meaning**

The user intentionally entered the booking flow.

**Trigger condition**

An eligible booking-button interaction is followed by the appearance of the known booking form within the defined confirmation window.

**Required parameter**

```text
booking_type
```

Allowed values:

```text
equipment_reservation
session_booking
```

**Measurement grain**

Once per `booking_type` per browser document lifecycle.

**Do not fire when**

* an unrelated button is clicked;
* a booking form does not appear within the confirmation window;
* the same `booking_type` has already been recorded during the current browser document lifecycle.

The current implementation tracks booking category, not individual item identity.

---

### 3.7 Booking `form_start`

**Business meaning**

The user began meaningful data entry in the booking form.

**Trigger conditions**

The first qualifying interaction is one of:

```text
name
email
phone
notes
```

containing at least one non-whitespace character,

or:

```text
date
```

becoming non-empty,

or:

```text
people
```

changing away from its default value:

```text
2
```

**Required parameters**

```text
form_id = booking_request_modal
form_location = modal
booking_type
```

**Measurement grain**

Once per `booking_type` per browser document lifecycle.

**Do not fire when**

* the user only focuses a field;
* the user enters whitespace only;
* the consent checkbox is the only interaction;
* the people selector is opened but remains at its default value;
* the date field is opened without selecting a value;
* the same booking type has already recorded `form_start` during the current browser document lifecycle.

---

### 3.8 Booking `form_submit`

**Business meaning**

A booking request was successfully confirmed.

**Trigger condition**

The booking form is replaced by a confirmed success state containing a valid unique reference matching:

```text
BK-*
```

**Required parameters**

```text
form_id = booking_request_modal
form_location = modal
booking_type
```

**Measurement grain**

One event per unique confirmed `BK-*` reference.

**Do not fire when**

* the submission is invalid;
* a submit attempt occurs without confirmed success;
* the event originates from an unrelated form;
* the success state does not contain a valid booking reference;
* the same confirmed `BK-*` reference has already been counted.

A different valid `BK-*` reference represents a new legitimate booking outcome.

The booking reference is used for internal confirmation and deduplication and is not required as a GA4 reporting parameter.

---

### 3.9 `service_link_click`

**Business meaning**

The user activated an internal link leading to a specific service page.

**Trigger condition**

The destination matches:

```text
/services/<service-slug>
```

Examples:

```text
/services/sup-lessons
/services/equipment-rental
/services/guided-trips
```

**Required parameter**

```text
service_id
```

Example normalization:

```text
/services/sup-lessons
→ service_id = sup_lessons
```

**Measurement grain**

Once per `service_id` per browser document lifecycle.

**Do not fire when**

* the same `service_id` has already been recorded during the current browser document lifecycle;
* the link is not a specific service destination;
* the destination is a generic route such as:

```text
/services
/rental
/campaign
```

Query strings or URL fragments must not change the semantic `service_id`.

---

## 4. Deduplication Rules

| Event                 | Semantic key / grain              | Reset boundary                 |
| --------------------- | --------------------------------- | ------------------------------ |
| Contact `form_view`   | `form_id = contact_main`          | New browser-page lifecycle     |
| Contact `form_start`  | `form_id = contact_main`          | New browser-page lifecycle     |
| Contact `form_submit` | One confirmed pending submission  | New valid confirmed submission |
| `phone_view`          | `phone_label + phone_location`    | New browser document lifecycle |
| `phone_click`         | `phone_label + phone_location`    | New browser document lifecycle |
| `booking_cta_click`   | `booking_type`                    | New browser document lifecycle |
| Booking `form_start`  | `booking_type`                    | New browser document lifecycle |
| Booking `form_submit` | Unique confirmed `BK-*` reference | Different confirmed reference  |
| `service_link_click`  | `service_id`                      | New browser document lifecycle |

A reset boundary defines when the same semantic key becomes eligible again.

A different semantic key represents a different analytical occurrence and does not require a reset.

---

## 5. Known Assumptions and Limitations

### Contact `form_view`

The Contact form is identified using the stable form action:

```text
/api/public/contact
```

If the action or form implementation changes, the visibility configuration may require an update.

---

### Contact `form_start`

The implementation assumes stable Contact form field names, field types, and the current default subject value.

Changes to these fields may require an update to the interaction logic.

Newly added fields should be reviewed to determine whether they should qualify as meaningful form engagement.

---

### Contact `form_submit`

Successful submission confirmation currently depends on:

* the Contact form action;
* a temporary pending submission state;
* navigation to `/contact/success`;
* the `MSG-*` confirmation-reference format.

Changes to submission behavior, routing, or confirmation-reference format require review.

---

### Phone Tracking

In the inherited-site fallback architecture, `phone_label` and `phone_location` may be derived from:

* `tel:` URLs;
* DOM context;
* current route.

Changes to phone numbers, DOM structure, or routing can therefore require updates to GTM mappings or logic.

Frontend-provided semantic values would be more maintainable where available.

---

### Booking Tracking

Booking tracking depends on:

* stable booking-form structure;
* route-to-`booking_type` rules;
* relevant form fields;
* modal success behavior;
* the `BK-*` confirmation-reference format.

Changes to these structures require review.

---

### `booking_cta_click`

The inherited-site fallback confirms a booking CTA retrospectively when the known booking form appears shortly after the click.

If a genuine booking CTA is broken and fails to open the modal, the click cannot be confirmed by this implementation.

Frontend instrumentation from the booking-button handler would provide a stronger source of truth.

---

### Service-Link Tracking

Specific service identity depends on the route structure:

```text
/services/<service-slug>
```

If the route pattern changes, the trigger and slug parser require review.

---

### SPA Lifecycle

Several semantic deduplication states are stored for the current browser document lifecycle.

They survive normal SPA route navigation.

Therefore, navigating away from a route and returning to it does not automatically make the same semantic key eligible again.

A route-specific lifecycle would require additional SPA lifecycle logic.

---

### Booking Measurement Scope

Booking tracking intentionally uses:

```text
booking_type
```

rather than item-level identity.

If future measurement requires the same individual item to be connected across:

```text
CTA click
→ form_start
→ confirmed booking
```

the frontend should expose and preserve a stable value such as:

```text
booking_item_id
```

rather than requiring GTM to reconstruct that application state.

---

## 6. QA Status

### Overall Status

**PASS after one documented defect was fixed and regression-tested.**

| Tracking Area            | Status               |
| ------------------------ | -------------------- |
| Contact tracking         | PASS                 |
| Phone tracking           | PASS after QA-01 fix |
| Booking tracking         | PASS                 |
| Service-link tracking    | PASS                 |
| GTM Preview validation   | PASS                 |
| GA4 DebugView validation | PASS                 |

### QA-01 Summary

**Event**

```text
phone_view
```

**Issue type**

```text
DUPLICATE / WRONG_GRAIN
```

**Severity**

```text
Medium
```

**Expected**

One `phone_view` per unique:

```text
phone_label + phone_location
```

per browser document lifecycle.

**Actual**

Native GTM Element Visibility with `Once per element` allowed duplicate analytical events when:

* two different DOM elements represented the same semantic phone CTA;
* React recreated a previously viewed CTA after SPA navigation.

**Root cause**

The native rule deduplicated by DOM-element identity rather than semantic phone identity.

**Fix**

A semantic deduplication layer was introduced using:

```text
phone_label + phone_location
```

before the final GA4 event.

**Regression result**

PASS.

All final events were validated in GTM Preview, and selected final events and semantic parameters were confirmed in GA4 DebugView.

Detailed test cases and the full QA-01 analysis are documented in `03-qa-report.md`.
