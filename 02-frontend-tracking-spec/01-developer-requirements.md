# NORTHSTAR — Frontend Tracking Developer Requirements

## 1. Purpose

This document defines the general implementation requirements for frontend tracking instrumentation on the NORTHSTAR React/SPA website.

These requirements apply to all canonical tracking events and tracking-related DOM attributes defined in the accompanying specification.

The frontend is responsible for exposing reliable application facts and stable business context.

Google Tag Manager remains responsible for analytics policy, analytical deduplication, visibility thresholds, destination mapping, and reporting-specific transformations.

---

## 2. Core Architecture Rule

The implementation MUST follow this principle:

> **Frontend owns application truth. GTM owns analytics policy.**

Frontend tracking MUST expose facts that the application knows directly.

Frontend tracking MUST NOT reproduce reporting rules that can be managed more flexibly in GTM.

Examples of frontend-owned truth include:

* selected business entity;
* booking type;
* booking item identity;
* actual CTA activation;
* meaningful form start;
* confirmed submission success;
* unique application-generated submission or booking reference.

Examples of GTM-owned policy include:

* session-level deduplication;
* document-level analytical deduplication;
* visibility percentage;
* visibility duration;
* GA4 event naming;
* conversion classification;
* Google Ads / Meta routing.

---

## 3. Canonical Tracking Interface

Frontend tracking SHOULD use one shared tracking utility rather than direct `window.dataLayer.push()` calls distributed across unrelated application components.

Conceptual usage:

```javascript
trackEvent('phone_click', {
  phone_label: 'sales',
  phone_location: 'product_detail'
});
```

The utility SHOULD be responsible for:

* Data Layer availability;
* canonical push format;
* lightweight validation;
* development diagnostics;
* prevention of technical implementation errors.

The utility MUST NOT implement destination-specific analytics rules.

---

## 4. Data Layer Initialization

The implementation MUST preserve an existing Data Layer.

Correct initialization pattern:

```javascript
window.dataLayer = window.dataLayer || [];
```

The implementation MUST NOT overwrite an existing Data Layer using:

```javascript
window.dataLayer = [];
```

after GTM or other tracking logic may already have initialized it.

---

## 5. Atomic Event Payloads

Each canonical tracking event MUST be pushed together with all required event context in a single atomic `dataLayer.push()`.

Correct:

```javascript
dataLayer.push({
  event: 'booking_cta_click',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001'
});
```

Incorrect:

```javascript
dataLayer.push({
  booking_type: 'equipment_reservation'
});

dataLayer.push({
  booking_item_id: 'sup_board_001'
});

dataLayer.push({
  event: 'booking_cta_click'
});
```

Required event context MUST NOT depend on Data Layer values pushed previously.

Each canonical event must be independently interpretable from its own payload.

---

## 6. Required Parameters

All parameters marked as required in the Data Layer Event Specification MUST:

* be present at the time of the push;
* contain a valid value;
* use the required data type;
* comply with any defined allowed-value list.

Required parameters MUST NOT be:

```text
missing
null
undefined
empty string
```

If required context is not available, the instrumentation implementation is considered incomplete or defective.

GTM MUST NOT be relied upon to reconstruct required application context that the frontend already owns.

---

## 7. Optional Parameters

Optional parameters MAY be included when a valid value is available.

If an optional value is unavailable, the parameter SHOULD be omitted.

Preferred:

```javascript
{
  event: 'phone_click',
  phone_label: 'sales',
  phone_location: 'product_detail'
}
```

Avoid:

```javascript
{
  event: 'phone_click',
  phone_label: 'sales',
  phone_location: 'product_detail',
  click_url: null
}
```

Optional parameters SHOULD NOT be populated with placeholder values solely to keep a fixed object shape.

---

## 8. Current Application State

Canonical event parameters MUST be resolved from the current application state at the moment the event occurs.

The implementation MUST prevent stale context from a previous interaction from being reused.

Example:

```text
User selects booking item A
→ modal opened
→ modal closed

User selects booking item B
→ modal opened
```

The second interaction MUST contain the identity of item B.

The previous item identity MUST NOT survive accidentally through stale React state, stale closures, or reused component state.

---

## 9. Stable Business Identity

Business identity MUST come from stable application data.

Tracking MUST NOT derive business identity from:

* translated UI text;
* button text;
* headings;
* CSS classes intended for presentation;
* visual position;
* arbitrary DOM order.

Preferred source:

```javascript
service.id
booking.id
booking.type
```

rather than:

```javascript
element.innerText
pathname parsing
DOM hierarchy reconstruction
```

when the application already owns the semantic value directly.

---

## 10. Machine-Readable Values

Tracking values MUST be:

* stable;
* language-independent;
* machine-readable;
* independent of display copy.

Example:

```text
booking_type = session_booking
```

MUST remain the same across:

```text
English
Polish
Ukrainian
```

Visible text may change.

Tracking values must not.

---

## 11. Naming Convention

Data Layer event names and parameter names MUST use `snake_case`.

Examples:

```text
booking_cta_click
booking_form_start
booking_item_id
booking_type
phone_label
service_id
```

DOM-facing role values MAY use `kebab-case`.

Examples:

```html
data-track-id="booking-cta"
data-track-id="phone-link"
data-track-id="service-link"
```

Tracking names MUST remain consistent with the specification.

---

## 12. Canonical Event Names

Canonical frontend event names are interface contracts.

Once implemented, names MUST NOT be renamed independently by frontend developers.

Example:

```text
booking_form_submit_success
```

MUST NOT be changed to:

```text
booking_success
submit_booking
bookingSubmit
```

without coordinated tracking changes.

The same rule applies to parameter names.

---

## 13. Technical Duplicate Prevention

The frontend MUST prevent technical duplicate events.

One real application occurrence MUST produce one canonical Data Layer event.

Technical duplicates caused by:

* duplicate listeners;
* repeated React effects;
* component rerenders;
* duplicated handlers;
* development-mode behavior;
* repeated success-state rendering;

MUST NOT create multiple canonical events for the same real occurrence.

Example:

```text
one confirmed booking
→ one booking_form_submit_success
```

---

## 14. Real Repeated Interactions

The frontend MUST preserve real repeated user interactions.

Example:

```text
user activates phone CTA three times
```

SHOULD produce:

```text
three phone_click canonical events
```

The frontend MUST NOT apply analytics-level deduplication such as:

```text
once per session
once per document
once per semantic key
```

unless explicitly defined as part of the application fact itself.

Analytical deduplication remains a GTM responsibility.

---

## 15. Form Start Semantics

A form-start event MUST represent the transition of a form instance from:

```text
pristine
→
meaningfully started
```

It MUST NOT represent every `input`, `change`, or `focus` browser event.

A real form instance SHOULD produce one canonical form-start event when it first qualifies as meaningfully started.

Further field interactions in the same form instance MUST NOT generate additional form-start events.

---

## 16. Confirmed Success Semantics

Events ending in:

```text
_submit_success
```

MUST represent confirmed application success.

They MUST NOT be triggered merely by:

* submit-button click;
* browser `submit` event;
* validation success;
* request initiation;
* loading state.

A success event MUST be emitted only after the application or backend confirms that the relevant business operation succeeded.

---

## 17. Unique Success References

Where the application produces a unique success reference such as:

```text
submission_id
booking_id
```

the reference MUST remain stable for that confirmed business outcome.

The same success reference MUST NOT generate multiple canonical success events because of rerendering or repeated observation.

A different valid success reference represents a different real business occurrence.

---

## 18. PII Prohibition

Personal user-entered information MUST NOT be pushed into the Data Layer.

Do not include:

* personal name;
* email address;
* personal phone number;
* free-text message;
* free-text notes;
* postal address;
* other user-entered personally identifying content.

This restriction applies even if the value is intended only for debugging.

Tracking payloads should contain business and technical metadata only.

---

## 19. Destination Independence

Frontend tracking MUST remain destination-neutral.

The frontend MUST NOT contain:

* GA4 Measurement IDs;
* Google Ads conversion IDs;
* Google Ads conversion labels;
* Meta Pixel IDs;
* destination-specific conversion rules;
* primary/secondary conversion configuration;
* destination-specific event naming logic.

The same canonical frontend fact may later be routed by GTM to multiple destinations.

---

## 20. SPA Compatibility

Tracking MUST work correctly for components reached through:

```text
initial document load
SPA navigation
```

Canonical events MUST NOT depend on `DOMContentLoaded` or `window.onload` for React-rendered components.

Tracking must reflect the current route and current application state where relevant.

SPA navigation MUST NOT cause stale business context to leak into later interactions.

---

## 21. Component Reuse

Reusable components SHOULD receive semantic tracking context through component or business-data props.

Example:

```javascript
<PhoneLink
  label="sales"
  location="product_detail"
/>
```

The component SHOULD NOT infer its semantic identity from the current route when the parent/application already knows that identity.

Similarly, reusable service or booking components SHOULD receive stable entity IDs directly.

---

## 22. Shared Source of Truth

Where possible, DOM tracking metadata and Data Layer event context SHOULD come from the same application-level source.

Example:

```text
service.id = guided_trips
```

should consistently drive:

```text
data-service-id="guided_trips"

and

service_id = guided_trips
```

Separate hardcoded values for DOM metadata, frontend events, and GTM mappings SHOULD be avoided.

---

## 23. Tracking DOM Attributes

Tracking-related `data-*` attributes are part of the tracking interface.

Examples:

```text
data-track-id
data-form-id
data-form-location
data-phone-label
data-phone-location
data-booking-type
data-booking-item-id
data-service-id
```

These attributes MUST NOT be renamed, removed, or repurposed without tracking review.

They must remain independent of styling and presentation logic.

---

## 24. Non-Blocking Behavior

Tracking instrumentation MUST NOT break core site functionality.

Failure of tracking logic MUST NOT prevent:

* navigation;
* telephone-link activation;
* modal opening;
* form interaction;
* form submission;
* success-state rendering.

Tracking SHOULD behave as a non-blocking side effect of application behavior.

---

## 25. User Interaction Preservation

Tracking MUST NOT change normal user interaction unless explicitly required.

For example, phone and service links SHOULD NOT use `preventDefault()` solely to wait for analytics processing.

Tracking must not introduce visible delays or alter standard navigation behavior.

---

## 26. Existing Instrumentation

Existing tracking code MUST be reviewed before implementing the new canonical contract.

Legacy and new tracking MUST NOT create duplicate canonical events for one real interaction.

For example, an existing:

```text
phone_click
```

push must be consolidated rather than left running alongside a new canonical `phone_click` implementation.

Existing shared tracking for phone, email, and outbound links should be reviewed for consistency with the new architecture.

---

## 27. Migration Coordination

Frontend instrumentation changes and GTM migration SHOULD be coordinated.

The intended migration flow is:

```text
new frontend contract implemented
↓
GTM updated to consume new contract
↓
QA completed
↓
legacy inherited-site fallback removed
```

The old fallback implementation SHOULD NOT be removed before the new frontend contract is validated.

---

## 28. Development Diagnostics

In development environments, the shared tracking utility SHOULD support lightweight diagnostics.

Examples:

* log canonical event name;
* warn when required parameters are missing;
* warn when controlled values are invalid;
* help identify accidental duplicate pushes.

Development diagnostics MUST NOT expose PII.

Verbose debugging output SHOULD NOT be required in production.

---

## 29. Error Handling

Tracking failures SHOULD fail safely.

A tracking error MUST NOT cause the application business flow to fail.

If the Data Layer or tracking utility cannot complete a push, the application should continue its normal user-facing operation.

---

## 30. Existing User Experience

The tracking implementation MUST preserve the existing frontend experience.

The purpose of this work is instrumentation, not redesign.

Unless separately specified, tracking changes MUST NOT alter:

* visual layout;
* content;
* route behavior;
* form behavior;
* modal behavior;
* link destinations;
* localization;
* existing business functionality.

---

## 31. Scope Boundary

This phase does not include a redesign of:

* consent management;
* Consent Mode;
* server-side tracking;
* CAPI;
* advertising-platform deduplication;
* ecommerce instrumentation outside the defined NORTHSTAR scope.

Those topics require separate specifications.

---

## 32. Acceptance Dependency

The frontend tracking implementation is considered complete only when:

1. all canonical events match the Data Layer Event Specification;
2. all stable identifiers match the Tracking Identifiers specification;
3. SPA and application-state behavior match the State and Lifecycle specification;
4. all QA Acceptance Criteria pass;
5. no duplicate legacy instrumentation remains for migrated events.
