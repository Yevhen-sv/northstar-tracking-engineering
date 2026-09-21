# NORTHSTAR — Frontend Tracking QA Acceptance Criteria

## 1. Purpose

This document defines the QA acceptance criteria for the NORTHSTAR frontend tracking implementation.

The frontend tracking work is considered complete only when:

* canonical Data Layer events match the specification;
* required parameters are present and correct;
* business identity remains consistent across multi-step flows;
* technical duplicate events are prevented;
* real repeated user interactions remain observable;
* tracking works across SPA navigation and localization;
* no PII is exposed;
* legacy tracking does not create duplicate events;
* tracking changes do not break existing site functionality.

---

## 2. QA Tools

Frontend tracking validation should use:

* browser DevTools;
* Data Layer inspection;
* GTM Preview / Tag Assistant where applicable;
* React application behavior;
* controlled positive and negative test cases.

At this stage, the primary objective is to validate the **frontend tracking contract**, not final GA4 configuration.

---

# 3. General Acceptance Criteria

## QA-G01 — Atomic Event Payload

### Test

Trigger any canonical frontend event.

### Expected

The event and all required parameters appear in the same `dataLayer.push()`.

Example:

```javascript
{
  event: 'booking_cta_click',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001'
}
```

### Fail If

Required event context is supplied through separate previous pushes.

---

## QA-G02 — Required Parameters

### Test

Trigger every canonical event.

### Expected

All required parameters defined in `02-data-layer-event-spec.md` are:

* present;
* non-empty;
* valid;
* correctly typed.

### Fail If

Any required parameter is:

```text
missing
null
undefined
empty string
```

---

## QA-G03 — Current Application State

### Test

Interact with business entity A, then business entity B.

### Expected

Events for entity B contain only B's current context.

### Fail If

Any parameter from entity A appears in B's event.

---

## QA-G04 — Language Independence

### Test

Repeat equivalent tracking actions in:

```text
English
Polish
Ukrainian
```

### Expected

Machine values remain unchanged.

Example:

```text
booking_type = session_booking
service_id = guided_trips
```

must not change with UI language.

---

## QA-G05 — No PII

### Test

Complete Contact and Booking forms using test personal data.

Inspect all Data Layer pushes.

### Expected

No user-entered personal values appear in the Data Layer.

### Must Not Include

* name;
* email address;
* personal phone number;
* message content;
* notes;
* address;
* other free-text personal values.

---

## QA-G06 — Technical Duplicate Prevention

### Test

Perform one real user action.

### Expected

One canonical frontend event.

### Fail If

One real action generates multiple pushes because of:

* nested handlers;
* duplicate listeners;
* rerender;
* repeated React effect;
* duplicated success callback.

---

## QA-G07 — Preserve Real Repeated Actions

### Test

Perform the same eligible real interaction multiple times.

Example:

```text
phone click
phone click
phone click
```

### Expected

Each real interaction remains observable as a canonical frontend event.

Frontend must not apply analytics-level suppression.

---

## QA-G08 — Tracking Failure Must Not Break UX

### Test

Verify normal application behavior with tracking instrumentation active.

### Expected

Tracking does not block:

* navigation;
* phone links;
* service links;
* modal opening;
* form interaction;
* form submission;
* success-state rendering.

---

# 4. Contact Form QA

## QA-C01 — Contact DOM Identity

### Expected DOM

```html
<form
  data-track-id="contact-form"
  data-form-id="contact_main"
  data-form-location="contact_page"
>
```

### PASS

Attributes exist and match the specification.

---

## QA-C02 — Focus Only

### Action

Focus a Contact field without entering meaningful data.

### Expected

No:

```text
contact_form_start
```

---

## QA-C03 — Whitespace Only

### Action

Enter whitespace only.

### Expected

No `contact_form_start`.

---

## QA-C04 — Meaningful Start

### Action

Enter the first non-whitespace value in an eligible Contact field.

### Expected

Exactly one:

```javascript
{
  event: 'contact_form_start',
  form_id: 'contact_main',
  form_location: 'contact_page'
}
```

---

## QA-C05 — Additional Contact Input

### Action

After `contact_form_start`, continue editing multiple fields.

### Expected

No second `contact_form_start` for the same form instance.

---

## QA-C06 — Subject Change

### Action

Change `subject` from the default value:

```text
order
```

to another valid value.

### Expected

One `contact_form_start` if the form has not already started.

---

## QA-C07 — Consent Only

### Action

Interact only with the consent checkbox.

### Expected

No `contact_form_start`.

---

## QA-C08 — Invalid Contact Submission

### Action

Attempt invalid submission.

### Expected

No:

```text
contact_form_submit_success
```

---

## QA-C09 — Confirmed Contact Success

### Action

Submit a valid Contact request and receive confirmed application success.

### Expected

One event:

```javascript
{
  event: 'contact_form_submit_success',
  form_id: 'contact_main',
  form_location: 'contact_page',
  submission_id: 'MSG-*'
}
```

---

## QA-C10 — Contact Success Technical Duplicate

### Action

Cause success UI to rerender or revisit the same confirmed success state.

### Expected

The same `submission_id` does not generate another canonical success event.

---

## QA-C11 — Second Legitimate Submission

### Action

Create another valid Contact submission with a new `submission_id`.

### Expected

A new `contact_form_submit_success` is generated.

---

# 5. Phone QA

## QA-P01 — Phone DOM Metadata

Each tracked phone link must expose:

```html
data-track-id="phone-link"
data-phone-label="..."
data-phone-location="..."
```

### PASS

DOM metadata matches application business context.

---

## QA-P02 — Phone Click Payload

### Action

Activate a tracked phone link.

### Expected

One canonical event:

```javascript
{
  event: 'phone_click',
  phone_label: 'sales',
  phone_location: 'product_detail'
}
```

Optional `click_url` may also appear.

---

## QA-P03 — Repeated Phone Clicks

### Action

Activate the same phone link three separate times.

### Expected

Three canonical frontend `phone_click` events.

Frontend must not analytically deduplicate them.

---

## QA-P04 — Phone Context Consistency

### Action

Test phone CTAs in:

* footer;
* product detail;
* service detail;
* rental page;
* contact page.

### Expected

`phone_label` and `phone_location` match the actual semantic context.

---

## QA-P05 — DOM vs Data Layer

### Action

Inspect a phone link and then activate it.

### Expected

Example:

```text
DOM:
data-phone-label="sales"
data-phone-location="product_detail"

Data Layer:
phone_label = sales
phone_location = product_detail
```

Values must match.

---

# 6. Email QA

## QA-E01 — Email Click

### Action

Activate a tracked email link.

### Expected

One:

```text
email_click
```

with correct:

```text
email_label
email_location
```

---

## QA-E02 — Repeated Email Activations

Each real activation remains observable.

No frontend analytics-level deduplication.

---

# 7. Outbound QA

## QA-O01 — Eligible Outbound Link

### Action

Activate an eligible external link.

### Expected

One:

```text
outbound_click
```

with correct semantic context.

---

## QA-O02 — Internal Link

### Action

Activate an internal NORTHSTAR link.

### Expected

No `outbound_click`.

---

## QA-O03 — Phone / Email Exclusion

### Action

Activate `tel:` and `mailto:` links.

### Expected

They must not also generate `outbound_click`.

---

# 8. Booking CTA QA

## QA-B01 — Booking CTA DOM Identity

Tracked booking CTAs must expose:

```html
data-track-id="booking-cta"
data-booking-type="..."
data-booking-item-id="..."
```

### PASS

Values match the current application entity.

---

## QA-B02 — Booking CTA Event

### Action

Activate a valid booking CTA.

### Expected

One:

```javascript
{
  event: 'booking_cta_click',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001'
}
```

---

## QA-B03 — Nested CTA Elements

### Action

Click text, span, icon, or other nested content inside the same CTA.

### Expected

One canonical `booking_cta_click`.

Not multiple events.

---

## QA-B04 — Repeated Booking CTA

### Action

Open booking, close it, and activate the same booking CTA again.

### Expected

Two real `booking_cta_click` events.

Frontend must not suppress the second interaction.

---

## QA-B05 — Different Booking Items

### Action

Activate booking item A and then booking item B.

### Expected

Events contain correct current IDs.

Example:

```text
A → booking_item_id = item_a
B → booking_item_id = item_b
```

No stale context.

---

# 9. Booking Form Start QA

## QA-BF01 — Fresh Form Instance

### Action

Open a valid booking CTA.

### Expected

A fresh booking form appears for the selected booking context.

---

## QA-BF02 — Focus Only

Focus fields without meaningful interaction.

### Expected

No:

```text
booking_form_start
```

---

## QA-BF03 — Meaningful Input

Enter meaningful content.

### Expected

Exactly one:

```javascript
{
  event: 'booking_form_start',
  form_id: 'booking_request_modal',
  form_location: 'modal',
  booking_type: '...',
  booking_item_id: '...'
}
```

---

## QA-BF04 — Additional Input

Continue editing the same form instance.

### Expected

No second `booking_form_start`.

---

## QA-BF05 — Consent Only

Interact only with consent.

### Expected

No `booking_form_start`.

---

## QA-BF06 — People Default

Open the people selector and leave it at:

```text
2
```

### Expected

No start from this interaction alone.

---

## QA-BF07 — People Changed

Change people from:

```text
2
```

to another value.

### Expected

One `booking_form_start` if the form has not already started.

---

## QA-BF08 — Booking Identity Continuity

### Action

Open booking for item A and start the form.

### Expected

`booking_form_start` contains the same:

```text
booking_type
booking_item_id
```

as the preceding `booking_cta_click`.

---

# 10. Booking Success QA

## QA-BS01 — Invalid Booking Submission

### Action

Attempt invalid booking submission.

### Expected

No:

```text
booking_form_submit_success
```

---

## QA-BS02 — Submit Attempt Only

### Action

Trigger submission without confirmed backend/application success.

### Expected

No success event.

---

## QA-BS03 — Confirmed Booking Success

### Action

Complete a valid booking.

### Expected

One:

```javascript
{
  event: 'booking_form_submit_success',
  form_id: 'booking_request_modal',
  form_location: 'modal',
  booking_type: '...',
  booking_item_id: '...',
  booking_id: 'BK-*'
}
```

---

## QA-BS04 — Booking Identity Continuity

For one booking flow:

```text
booking_cta_click
↓
booking_form_start
↓
booking_form_submit_success
```

the same:

```text
booking_type
booking_item_id
```

must appear throughout.

---

## QA-BS05 — Success Rerender

### Action

Allow the same confirmed success UI to rerender.

### Expected

Same `booking_id` does not generate another canonical success event.

---

## QA-BS06 — Modal Dismissal Reset

### Action

After successful booking, dismiss the success modal using:

* close `X`;
* backdrop / outside-modal click.

Then activate any booking CTA.

### Expected

A fresh booking form opens.

The previous:

```text
booking_id
success state
form state
```

must not remain active.

---

## QA-BS07 — Send Another Request

### Action

Use the existing:

```text
Send another request
```

control.

### Expected

A fresh booking form is initialized.

Old success context must not generate another success event.

---

## QA-BS08 — Second Legitimate Booking

### Action

Complete another booking with a different confirmed booking reference.

### Expected

Second unique:

```text
booking_form_submit_success
```

with new `booking_id`.

---

# 11. Service-Link QA

## QA-S01 — Service DOM Identity

Specific service links expose:

```html
data-track-id="service-link"
data-service-id="..."
```

---

## QA-S02 — Specific Service Click

### Action

Activate:

```text
guided_trips
```

### Expected

```javascript
{
  event: 'service_link_click',
  service_id: 'guided_trips'
}
```

---

## QA-S03 — Generic Service Navigation

### Action

Activate generic routes such as:

```text
/services
/rental
/campaign
```

### Expected

No `service_link_click`, unless separately defined in a future contract.

---

## QA-S04 — Repeated Service Click

Two real activations of the same service link should produce two canonical frontend events.

GTM may later apply analytical deduplication.

---

## QA-S05 — Route Independence

If the service route changes but `service_id` remains the same business entity, the canonical tracking identity must remain stable.

---

# 12. SPA QA

## QA-SPA01 — Initial Load

Canonical tracking works when a user enters a route through full initial page load.

---

## QA-SPA02 — SPA Navigation

Canonical tracking works after navigating to the same feature through React SPA navigation.

---

## QA-SPA03 — Back / Forward Navigation

Browser history navigation must not create click or form events by itself.

Current application context must remain correct.

---

## QA-SPA04 — Stale Booking Context

### Action

Open booking item A, close, navigate, then open item B.

### Expected

All item B events contain item B context.

No stale item A values.

---

# 13. Localization QA

Repeat representative flows in:

```text
EN
PL
UA
```

### Expected

The UI text changes.

Tracking identity does not.

Examples:

```text
booking_type = session_booking
service_id = guided_trips
phone_label = sales
```

remain stable.

---

# 14. Legacy Instrumentation QA

## QA-L01 — Existing Phone Tracking

Verify that the existing legacy `phone_click` implementation has been consolidated.

### Expected

One physical phone activation creates one canonical frontend `phone_click`.

Not:

```text
old phone_click
+
new phone_click
```

---

## QA-L02 — Existing Email Tracking

Verify that one email activation creates one canonical `email_click`.

---

## QA-L03 — Existing Outbound Tracking

Verify that one outbound activation creates one canonical `outbound_click`.

---

## QA-L04 — No Parallel Legacy Booking Logic

The new frontend contract must not coexist with duplicate frontend tracking implementations representing the same booking fact.

---

# 15. Tracking Attribute QA

Inspect representative elements and verify that tracking attributes are:

* present;
* correctly named;
* semantically correct;
* language-independent;
* not derived from presentation CSS;
* consistent with Data Layer values.

---

# 16. Final Acceptance Matrix

| Area                   | Acceptance Requirement                       |
| ---------------------- | -------------------------------------------- |
| Contact form           | PASS all Contact tests                       |
| Phone                  | PASS click + metadata tests                  |
| Email                  | PASS canonical click tests                   |
| Outbound               | PASS classification tests                    |
| Booking CTA            | PASS identity and repeat tests               |
| Booking form start     | PASS lifecycle and identity tests            |
| Booking success        | PASS success, reset, dedupe and repeat tests |
| Service links          | PASS identity and classification tests       |
| SPA                    | PASS initial + SPA + history tests           |
| Localization           | PASS EN / PL / UA identity consistency       |
| PII                    | No prohibited personal values present        |
| Legacy instrumentation | No duplicate old/new events                  |
| Existing UX            | No regressions caused by tracking            |

---

# 17. Definition of Done

The frontend tracking implementation is accepted only when:

```text
all required canonical events are implemented
+
all required parameters are correct
+
business identity remains consistent
+
technical duplicates are prevented
+
real repeated actions remain observable
+
SPA behavior passes
+
localization passes
+
PII boundary passes
+
legacy duplicates are removed
+
existing site behavior remains unchanged
```

Only after these criteria pass should the inherited GTM fallback logic be removed and the new GTM implementation become the primary analytics integration.
