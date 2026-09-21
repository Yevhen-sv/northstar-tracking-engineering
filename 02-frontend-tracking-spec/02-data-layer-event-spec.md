# NORTHSTAR — Data Layer Event Specification

## 1. Purpose

This document defines the canonical frontend Data Layer events for the NORTHSTAR React/SPA website.

Each event contract defines:

* the application fact represented by the event;
* the exact trigger moment;
* required and optional parameters;
* parameter types and controlled values;
* frontend repeat behavior;
* technical duplicate rules;
* conditions under which the event must not be pushed.

All events defined here are destination-neutral.

GTM is responsible for mapping these canonical frontend events to GA4, Google Ads, Meta, or other analytics destinations.

---

# 2. General Event Rules

All canonical events MUST follow the general requirements defined in:

```text
01-developer-requirements.md
```

In particular:

* required parameters must be available at push time;
* the event and its required context must be sent in one atomic push;
* current application state must be used;
* PII must not be included;
* one real application occurrence must not create multiple technical pushes;
* genuine repeated user actions must remain observable;
* analytics-level deduplication must not be implemented in the frontend unless explicitly specified.

---

# 3. Contact Form Events

## 3.1 `contact_form_start`

### Application Fact

The Contact form has transitioned from a pristine state to a meaningfully started state.

### Exact Trigger

Push once when the current Contact form instance receives its first meaningful interaction.

Meaningful interactions are:

* non-whitespace input in `name`;
* non-whitespace input in `email`;
* non-whitespace input in `phone`;
* non-whitespace input in `message`;
* `subject` changed away from its default value `order`.

The consent checkbox alone does not qualify.

### Required Parameters

| Parameter       | Type   | Required Value |
| --------------- | ------ | -------------- |
| `form_id`       | string | `contact_main` |
| `form_location` | string | `contact_page` |

### Optional Parameters

None required for the current implementation.

### Example

```javascript
dataLayer.push({
  event: 'contact_form_start',
  form_id: 'contact_main',
  form_location: 'contact_page'
});
```

### Repeat Behavior

One canonical event per real Contact form instance.

Further meaningful interactions in the same form instance MUST NOT generate another `contact_form_start`.

A new form instance may generate a new canonical event.

### Do Not Push

Do not push when:

* the user only focuses a field;
* the user enters whitespace only;
* the user only interacts with consent;
* the subject field is opened but remains at its default value;
* the same form instance has already entered the started state.

---

## 3.2 `contact_form_submit_success`

### Application Fact

The Contact request has been successfully accepted and confirmed by the application/backend.

### Exact Trigger

Push only after confirmed successful submission.

Do not push on button click, browser submit event, validation success, or request initiation.

### Required Parameters

| Parameter       | Type   | Requirement                           |
| --------------- | ------ | ------------------------------------- |
| `form_id`       | string | `contact_main`                        |
| `form_location` | string | `contact_page`                        |
| `submission_id` | string | unique confirmed submission reference |

Expected submission-reference format currently follows:

```text
MSG-*
```

The frontend must use the actual application-generated identifier rather than generate an analytics-only replacement.

### Example

```javascript
dataLayer.push({
  event: 'contact_form_submit_success',
  form_id: 'contact_main',
  form_location: 'contact_page',
  submission_id: 'MSG-K7SL7D'
});
```

### Repeat Behavior

One canonical success event per real successful Contact submission.

Different confirmed submission IDs represent different real outcomes.

### Technical Duplicate Rule

The same `submission_id` MUST NOT generate repeated canonical success events because of:

* success-page rerender;
* component rerender;
* repeated effect execution;
* duplicate callback execution.

### Do Not Push

Do not push when:

* form validation fails;
* a request starts but has not been confirmed successful;
* the backend rejects the submission;
* a network error occurs;
* an existing success state is rendered again for the same confirmed submission.

---

# 4. Phone Events

## 4.1 `phone_click`

### Application Fact

The user activated an eligible phone CTA.

### Exact Trigger

Push for every real user activation of a tracked phone link.

### Required Parameters

| Parameter        | Type   | Allowed Values                          |
| ---------------- | ------ | --------------------------------------- |
| `phone_label`    | string | `general`, `sales`, `rental`, `support` |
| `phone_location` | string | controlled location value               |

Current `phone_location` values include:

```text
footer
product_detail
service_detail
rental_page
contact_page
```

New values may be added only when they represent a stable semantic placement.

### Optional Parameters

| Parameter   | Type   | Purpose                    |
| ----------- | ------ | -------------------------- |
| `click_url` | string | diagnostic destination URL |

Example:

```javascript
dataLayer.push({
  event: 'phone_click',
  phone_label: 'sales',
  phone_location: 'product_detail',
  click_url: 'tel:+48555100101'
});
```

### Repeat Behavior

Every real eligible activation MUST produce a canonical event.

Example:

```text
real click
real click
real click
```

must remain observable as three frontend events.

GTM may later apply analytical deduplication.

### Technical Duplicate Rule

One physical activation MUST NOT produce multiple `phone_click` pushes because of duplicate event handlers or component behavior.

### Do Not Push

Do not push for:

* non-phone links;
* programmatic DOM updates without real activation;
* repeated handler execution for one physical interaction.

---

# 5. Email Events

## 5.1 `email_click`

### Application Fact

The user activated an eligible email CTA.

### Exact Trigger

Push for every real user activation of a tracked `mailto:` link.

### Required Parameters

| Parameter        | Type   | Requirement                              |
| ---------------- | ------ | ---------------------------------------- |
| `email_label`    | string | stable semantic department/contact label |
| `email_location` | string | stable semantic placement                |

Current semantic labels include:

```text
general
sales
rental
support
```

Typical locations include:

```text
footer
product_detail
service_detail
rental_page
contact_page
```

### Optional Parameters

| Parameter   | Type   | Purpose                          |
| ----------- | ------ | -------------------------------- |
| `click_url` | string | diagnostic `mailto:` destination |

### Example

```javascript
dataLayer.push({
  event: 'email_click',
  email_label: 'sales',
  email_location: 'product_detail',
  click_url: 'mailto:sales@northstar.example'
});
```

### Repeat Behavior

Every real email-link activation should produce one canonical frontend event.

Analytics-level deduplication belongs in GTM.

### Technical Duplicate Rule

One real activation MUST NOT generate multiple canonical pushes.

### Do Not Push

Do not push for:

* non-email links;
* component rerender;
* programmatic changes without user activation;
* duplicate handler execution.

---

# 6. Outbound-Link Events

## 6.1 `outbound_click`

### Application Fact

The user activated a tracked link whose destination is outside the NORTHSTAR application domain.

### Exact Trigger

Push on every real activation of an eligible outbound link.

Internal navigation MUST NOT generate this event.

### Required Parameters

| Parameter           | Type   | Requirement                               |
| ------------------- | ------ | ----------------------------------------- |
| `outbound_label`    | string | stable semantic destination/partner label |
| `outbound_location` | string | stable semantic placement                 |

### Optional Parameters

| Parameter   | Type   | Purpose                       |
| ----------- | ------ | ----------------------------- |
| `click_url` | string | full outbound destination URL |

### Example

```javascript
dataLayer.push({
  event: 'outbound_click',
  outbound_label: 'partner',
  outbound_location: 'service_detail',
  click_url: 'https://example-partner.com/'
});
```

### Repeat Behavior

Every real eligible outbound activation MUST remain observable.

Analytical deduplication belongs in GTM.

### Technical Duplicate Rule

One real activation MUST create one canonical frontend event.

### Do Not Push

Do not push for:

* internal NORTHSTAR navigation;
* phone links;
* email links;
* programmatic navigation not defined as a tracked user activation;
* duplicate handler execution.

---

# 7. Booking Events

## 7.1 `booking_cta_click`

### Application Fact

The user activated a booking CTA for a known booking entity.

### Exact Trigger

Push when the application handles a real eligible booking CTA activation and the current booking context is available.

The event must not depend on waiting for the booking modal to appear.

### Required Parameters

| Parameter         | Type   | Requirement                      |
| ----------------- | ------ | -------------------------------- |
| `booking_type`    | string | controlled enum                  |
| `booking_item_id` | string | stable application-level item ID |

Allowed `booking_type` values:

```text
equipment_reservation
session_booking
```

`booking_item_id` must identify the specific business entity selected by the user.

Examples:

```text
sup_board_001
guided_trip_001
sup_lesson_private
```

Actual identifiers must come from NORTHSTAR application data.

### Optional Parameters

| Parameter           | Type   | Purpose                             |
| ------------------- | ------ | ----------------------------------- |
| `booking_item_name` | string | reporting/debugging display name    |
| `booking_location`  | string | semantic CTA placement where useful |

`booking_item_name` MUST NOT replace `booking_item_id`.

### Example

```javascript
dataLayer.push({
  event: 'booking_cta_click',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001',
  booking_item_name: 'Loon SUP Board'
});
```

### Repeat Behavior

Every real booking CTA activation should produce one canonical event.

Example:

```text
open booking
close booking
open the same booking again
```

represents two real CTA activations.

GTM may choose to analytically deduplicate them.

### Technical Duplicate Rule

One activation MUST NOT generate multiple canonical events due to:

* nested element handlers;
* duplicate listeners;
* rerender;
* repeated effects.

### Do Not Push

Do not push when:

* an unrelated button is activated;
* required booking identity is unavailable;
* the event is created only because the modal rendered;
* the same physical activation is handled multiple times.

---

## 7.2 `booking_form_start`

### Application Fact

A booking form instance associated with a known booking entity has transitioned from pristine to meaningfully started.

### Exact Trigger

Push once when the current booking form instance first receives a meaningful interaction.

Meaningful interactions include:

* non-whitespace input in `name`;
* non-whitespace input in `email`;
* non-whitespace input in `phone`;
* non-whitespace input in `notes`;
* valid non-empty date selection;
* `people` changed away from default value `2`.

Consent-only interaction does not qualify.

### Required Parameters

| Parameter         | Type   | Requirement                     |
| ----------------- | ------ | ------------------------------- |
| `form_id`         | string | `booking_request_modal`         |
| `form_location`   | string | `modal`                         |
| `booking_type`    | string | controlled enum                 |
| `booking_item_id` | string | selected stable booking item ID |

### Example

```javascript
dataLayer.push({
  event: 'booking_form_start',
  form_id: 'booking_request_modal',
  form_location: 'modal',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001'
});
```

### Identity Requirement

The `booking_item_id` and `booking_type` MUST refer to the same booking entity that caused the current booking flow to open.

The frontend MUST preserve this identity through application state.

GTM MUST NOT be required to reconstruct it from URL, route, modal text, or DOM hierarchy.

### Repeat Behavior

One canonical event per real booking form instance.

A new form instance may generate a new canonical event.

GTM may apply additional analytical deduplication.

### Technical Duplicate Rule

Additional field interactions in the same started form instance MUST NOT create another `booking_form_start`.

### Do Not Push

Do not push when:

* only focus occurs;
* input contains whitespace only;
* consent is the only interaction;
* date remains empty;
* people remains at default;
* required booking identity is unavailable;
* the same form instance has already started.

---

## 7.3 `booking_form_submit_success`

### Application Fact

The application/backend confirmed creation of a booking for a known booking entity.

### Exact Trigger

Push only after the booking operation has been confirmed successful.

The event MUST NOT be based solely on browser submit behavior or visible form disappearance.

### Required Parameters

| Parameter         | Type   | Requirement                        |
| ----------------- | ------ | ---------------------------------- |
| `form_id`         | string | `booking_request_modal`            |
| `form_location`   | string | `modal`                            |
| `booking_type`    | string | controlled enum                    |
| `booking_item_id` | string | selected stable booking item ID    |
| `booking_id`      | string | unique confirmed booking reference |

Current booking-reference format follows:

```text
BK-*
```

The actual application-generated identifier must be used.

### Optional Parameters

| Parameter           | Type   | Purpose                          |
| ------------------- | ------ | -------------------------------- |
| `booking_item_name` | string | reporting/debugging display name |

### Example

```javascript
dataLayer.push({
  event: 'booking_form_submit_success',
  form_id: 'booking_request_modal',
  form_location: 'modal',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001',
  booking_id: 'BK-FLZZ7E'
});
```

### Identity Requirement

The same:

```text
booking_type
booking_item_id
```

must be preserved across:

```text
booking_cta_click
↓
booking_form_start
↓
booking_form_submit_success
```

for the same booking flow.

### Repeat Behavior

Every real successful booking must generate one canonical success event.

Different valid `booking_id` values represent different business outcomes.

### Technical Duplicate Rule

The same `booking_id` MUST NOT generate multiple success events due to:

* repeated success rendering;
* React rerender;
* repeated effect execution;
* duplicate callback execution.

### Do Not Push

Do not push when:

* validation fails;
* request has only started;
* backend returns an error;
* submission is rejected;
* booking identity is missing;
* no confirmed `booking_id` exists;
* an already-reported success state rerenders.

---

# 8. Service Navigation Events

## 8.1 `service_link_click`

### Application Fact

The user activated navigation to a known specific service entity.

### Exact Trigger

Push on every real activation of a tracked specific-service link.

The service identity must come from application data.

### Required Parameters

| Parameter    | Type   | Requirement              |
| ------------ | ------ | ------------------------ |
| `service_id` | string | stable service entity ID |

Current examples:

```text
sup_lessons
equipment_rental
guided_trips
```

### Optional Parameters

| Parameter          | Type   | Purpose                              |
| ------------------ | ------ | ------------------------------------ |
| `click_url`        | string | diagnostic destination URL           |
| `service_location` | string | semantic placement if later required |

### Example

```javascript
dataLayer.push({
  event: 'service_link_click',
  service_id: 'guided_trips',
  click_url: '/services/guided-trips'
});
```

### Repeat Behavior

Every real eligible service-link activation MUST remain observable.

GTM may apply analytical deduplication.

### Technical Duplicate Rule

One physical activation must result in one canonical event.

### Do Not Push

Do not push for:

```text
/services
/rental
/campaign
```

unless those routes are separately defined as tracked business entities in a future contract.

Do not push when:

* navigation is not to a specific service;
* service identity is unavailable;
* repeated frontend handlers process the same activation.

---

# 9. DOM-Based Visibility Contracts

The following analytical events are intentionally NOT canonical frontend Data Layer events:

```text
Contact form_view
phone_view
```

Visibility thresholds are analytics policy and remain configurable in GTM.

The frontend must provide stable semantic DOM metadata instead.

---

## 9.1 Contact Form Visibility Metadata

Required DOM contract:

```html
<form
  data-track-id="contact-form"
  data-form-id="contact_main"
  data-form-location="contact_page"
>
```

Frontend owns:

```text
form identity
form location
stable DOM metadata
```

GTM owns:

```text
visibility percentage
visibility duration
analytical deduplication
destination event
```

---

## 9.2 Phone Visibility Metadata

Each tracked phone CTA must expose:

```html
<a
  href="tel:+48555100101"
  data-track-id="phone-link"
  data-phone-label="sales"
  data-phone-location="product_detail"
>
```

Frontend owns:

```text
phone identity
phone location
stable DOM metadata
```

GTM owns:

```text
visibility threshold
visibility duration
analytical deduplication
destination event
```

The semantic values exposed in DOM metadata MUST match the values used by `phone_click`.

---

# 10. Canonical Event Summary

| Event                         | Required Context                                                            | Frontend Repeat Rule          |
| ----------------------------- | --------------------------------------------------------------------------- | ----------------------------- |
| `contact_form_start`          | `form_id`, `form_location`                                                  | once per real form instance   |
| `contact_form_submit_success` | `form_id`, `form_location`, `submission_id`                                 | once per confirmed submission |
| `phone_click`                 | `phone_label`, `phone_location`                                             | every real activation         |
| `email_click`                 | `email_label`, `email_location`                                             | every real activation         |
| `outbound_click`              | `outbound_label`, `outbound_location`                                       | every real activation         |
| `booking_cta_click`           | `booking_type`, `booking_item_id`                                           | every real activation         |
| `booking_form_start`          | `form_id`, `form_location`, `booking_type`, `booking_item_id`               | once per real form instance   |
| `booking_form_submit_success` | `form_id`, `form_location`, `booking_type`, `booking_item_id`, `booking_id` | once per confirmed booking    |
| `service_link_click`          | `service_id`                                                                | every real activation         |

---

# 11. Destination Mapping Boundary

This specification does not define final GA4 or advertising-platform event names.

Examples of possible GTM mappings:

```text
contact_form_submit_success
↓
GA4 form_submit
```

```text
booking_form_submit_success
↓
GA4 form_submit
↓
Google Ads conversion
```

```text
phone_click
↓
GA4 phone_click
```

These mappings belong to the analytics implementation layer and may change without requiring frontend changes.

---

# 12. Contract Change Rule

Changes to any of the following are considered tracking-interface changes:

```text
canonical event name
required parameter name
required parameter type
allowed controlled value
business identity source
tracking DOM attribute
success semantics
```

Such changes must be coordinated with the analytics implementation.

Changes to downstream reporting rules, analytical deduplication, visibility thresholds, or destination mappings should normally not require modification of this frontend event contract.
