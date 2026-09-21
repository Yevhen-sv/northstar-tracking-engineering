# NORTHSTAR — Frontend Tracking Specification

## Purpose

This specification defines the frontend tracking interface for the NORTHSTAR React/SPA website.

The objective is to replace tracking logic that currently depends on DOM reconstruction, URL inference, route-based assumptions, and success-state observation with a stable application-level tracking contract.

The frontend should expose reliable business facts and business context.

Google Tag Manager should remain responsible for flexible analytics policy, destination mapping, visibility thresholds, and reporting-specific logic.

The target architecture is:

```text
Application state / confirmed user action
↓
Canonical frontend tracking event
↓
dataLayer
↓
Google Tag Manager
↓
GA4 / Ads / other destinations
```

The frontend must not reproduce destination-specific analytics logic.

---

## Architecture Principle

The implementation follows one primary rule:

> **Frontend owns truth. GTM owns analytics policy.**

Frontend is responsible for facts that the application knows directly, including:

* business entity identity;
* actual user interactions;
* form state transitions;
* current booking context;
* confirmed application success;
* stable machine-readable values;
* technical duplicate prevention.

GTM remains responsible for:

* visibility thresholds;
* analytical deduplication;
* destination-specific event mapping;
* GA4 event naming where different from the frontend event;
* Google Ads / Meta routing;
* reporting-specific transformations.

A canonical frontend event should represent what actually happened in the application, not how the analytics platform currently wants to count it.

---

## Scope

The specification covers the tracking interface for:

### Contact

```text
Contact form identity
Contact form meaningful start
Contact form confirmed submission
Contact form visibility metadata
```

### Phone

```text
Phone CTA identity
Phone CTA clicks
Phone CTA visibility metadata
```

### Booking

```text
Booking CTA activation
Booking item identity
Booking form start
Booking confirmed submission
Booking identity propagation across the complete flow
```

### Service Navigation

```text
Specific service identity
Specific service link activation
```

Existing shared frontend instrumentation such as email and outbound-link tracking should also be reviewed so that legacy and new tracking implementations do not create duplicate events.

---

## Target Responsibility Split

| Requirement                    | Frontend |                  GTM |
| ------------------------------ | -------: | -------------------: |
| Stable business IDs            |      Yes |              Consume |
| Application event truth        |      Yes |              Consume |
| Confirmed submission success   |      Yes |              Consume |
| Selected booking item          |      Yes |              Consume |
| Technical duplicate prevention |      Yes | Defensive validation |
| Stable DOM tracking metadata   |      Yes |              Consume |
| Visibility threshold           |       No |                  Yes |
| Visibility duration            |       No |                  Yes |
| Analytical deduplication       |       No |                  Yes |
| Destination mapping            |       No |                  Yes |
| GA4 / Ads / Meta configuration |       No |                  Yes |
| Reporting-specific rules       |       No |                  Yes |

---

## Canonical Frontend Events

The frontend tracking contract will include the following canonical application events:

```text
contact_form_start
contact_form_submit_success

phone_click

booking_cta_click
booking_form_start
booking_form_submit_success

service_link_click
```

These events are destination-neutral.

For example:

```text
booking_form_submit_success
```

may later be mapped by GTM to:

```text
GA4 → form_submit
Google Ads → conversion
Meta → Lead or another configured event
```

without changing the frontend implementation.

---

## DOM-Based Tracking Contracts

Not every analytical event should be pushed by the frontend.

Visibility measurement remains in GTM because visibility thresholds are analytics policy and may change without requiring a frontend deployment.

The frontend must instead expose stable element metadata.

Examples:

```html
<form
  data-track-id="contact-form"
  data-form-id="contact_main"
  data-form-location="contact_page"
>
```

```html
<a
  href="tel:+48555100101"
  data-track-id="phone-link"
  data-phone-label="sales"
  data-phone-location="product_detail"
>
```

GTM can then apply configurable visibility rules without reconstructing semantic context from URLs, routes, or visible text.

---

## Data Layer Design Rules

Canonical events must be pushed with their complete required context in one atomic `dataLayer.push()`.

Correct:

```javascript
dataLayer.push({
  event: 'booking_cta_click',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001'
});
```

Do not split event context across multiple pushes.

Do not rely on previously stored Data Layer values to supply required context for a later event.

Required values must be valid and available at the moment the canonical event occurs.

Optional parameters that are unavailable should be omitted rather than populated with empty strings, `null`, or `undefined`.

---

## Stable Machine Values

Tracking values must be:

```text
stable
language-independent
machine-readable
independent of visible UI text
```

For example:

```text
booking_type = session_booking
```

must remain unchanged when the user interface changes between English, Polish, or Ukrainian.

Business identity must not be derived from translated labels.

---

## Technical vs Analytical Deduplication

The frontend must prevent technical duplicates.

One real application occurrence must produce one canonical frontend event.

For example:

```text
one confirmed booking
→ one booking_form_submit_success
```

React rerenders, duplicate listeners, or repeated effects must not generate duplicate canonical events.

However, the frontend must preserve real repeated user actions.

For example:

```text
three real phone CTA activations
→ three phone_click events
```

Whether those three events should later be counted as three analytical interactions or deduplicated is a GTM measurement-policy decision.

---

## Business Identity Propagation

Business identity must remain available throughout multi-step application flows.

Example booking flow:

```text
booking_cta_click
↓
booking_form_start
↓
booking_form_submit_success
```

If the selected item is:

```text
booking_item_id = sup_board_001
```

the same identity must remain available throughout the complete booking flow.

GTM must not reconstruct this identity from:

```text
URL
route
button text
DOM hierarchy
modal text
```

when the application already owns the information directly.

---

## PII Boundary

Tracking instrumentation must not push personal form values into the Data Layer.

Do not include:

```text
customer name
email address
personal phone number
message content
free-text notes
postal address
other user-entered personal data
```

Tracking events should contain only the business and technical metadata required by the tracking contract.

---

## SPA Requirements

Canonical tracking behavior must work correctly regardless of whether a component is reached through:

```text
initial document load
or
React SPA navigation
```

Tracking implementation must not depend on `DOMContentLoaded` or `window.onload` for dynamically rendered application components.

Event parameters must reflect the current application state at the moment the event occurs.

Stale entity or booking context must not survive into a new interaction.

---

## Non-Blocking Requirement

Analytics instrumentation must not change or block existing user-facing functionality.

Tracking must not:

```text
prevent normal navigation
delay primary actions unnecessarily
block form submission
prevent modal opening
break application behavior when tracking fails
```

The website must continue to function if the tracking utility or Data Layer is unavailable.

---

## Existing Instrumentation

Existing frontend tracking must be reviewed before the new contract is implemented.

Legacy and new implementations must not run in parallel if they represent the same application fact.

For example:

```text
existing phone_click push
+
new canonical phone_click push
```

must not result in two events for one real phone activation.

Existing instrumentation should be consolidated into the canonical contract.

---

## Documentation

The complete frontend tracking specification is divided into:

```text
01-developer-requirements.md
02-data-layer-event-spec.md
03-tracking-identifiers.md
04-state-and-lifecycle.md
05-qa-acceptance-criteria.md
```

### 01 — Developer Requirements

Defines the general implementation rules that apply to all frontend tracking instrumentation.

### 02 — Data Layer Event Specification

Defines the exact contract for each canonical frontend event:

```text
trigger
parameters
types
allowed values
repeat behavior
technical deduplication
do-not-fire conditions
example payload
```

### 03 — Tracking Identifiers

Defines stable machine identifiers and DOM tracking attributes.

### 04 — State and Lifecycle

Defines SPA lifecycle, form lifecycle, booking-context persistence, reset behavior, and business identity propagation.

### 05 — QA Acceptance Criteria

Defines the tests that must pass before the frontend tracking implementation is accepted.

---

## Migration Objective

The existing inherited-site implementation currently uses techniques such as:

```text
DOM inference
URL parsing
route inference
Lookup Tables
Custom JavaScript
MutationObserver
sessionStorage correlation
semantic reconstruction inside GTM
```

These mechanisms were valid fallbacks under the original frontend constraints.

The new frontend tracking contract should remove GTM reconstruction where the application already knows the business fact directly.

The target architecture is not to eliminate GTM logic completely.

The target is to make GTM a thinner and more maintainable integration layer:

```text
Frontend
→ reliable application facts

Data Layer
→ stable interface

GTM
→ measurement policy and destinations
```
