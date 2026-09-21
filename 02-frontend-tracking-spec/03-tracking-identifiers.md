# NORTHSTAR — Tracking Identifiers Specification

## 1. Purpose

This document defines the stable machine-readable identifiers and DOM tracking attributes used by the NORTHSTAR frontend tracking interface.

The purpose of these identifiers is to provide consistent business identity across:

* application state;
* DOM metadata;
* Data Layer events;
* GTM;
* QA;
* future analytics destinations.

Tracking identifiers must remain independent of translated UI text, visual layout, CSS styling, and temporary DOM structure.

---

## 2. Identifier Principles

All tracking identifiers MUST follow these principles:

* stable over time;
* language-independent;
* machine-readable;
* semantically meaningful;
* independent of display text;
* sourced from application/business data where possible;
* reused consistently across DOM and Data Layer events.

Business identity MUST NOT be reconstructed from visible text when a stable application identifier already exists.

---

## 3. Naming Conventions

### DOM Attribute Names

HTML tracking attributes use standard `data-*` naming.

Examples:

```html
data-track-id
data-form-id
data-form-location
data-phone-label
data-phone-location
data-booking-type
data-booking-item-id
data-service-id
```

### DOM Role Values

Values used primarily as DOM roles SHOULD use `kebab-case`.

Examples:

```text
contact-form
phone-link
booking-cta
booking-form
service-link
```

Example:

```html
data-track-id="booking-cta"
```

### Analytics / Business Values

Business identifiers and Data Layer values MUST use `snake_case`.

Examples:

```text
contact_main
booking_request_modal
equipment_reservation
session_booking
product_detail
guided_trips
sup_board_001
```

---

## 4. `data-track-id`

`data-track-id` identifies the tracking role of a DOM element.

It does not represent the business entity itself.

Current controlled values:

```text
contact-form
phone-link
booking-cta
booking-form
service-link
```

Examples:

```html
<form data-track-id="contact-form">
```

```html
<a data-track-id="phone-link">
```

```html
<button data-track-id="booking-cta">
```

`data-track-id` MUST NOT be generated from CSS class names or visible text.

---

## 5. Form Identifiers

### Contact Form

Stable form identifier:

```text
form_id = contact_main
```

DOM contract:

```html
<form
  data-track-id="contact-form"
  data-form-id="contact_main"
  data-form-location="contact_page"
>
```

### Booking Form

Stable form identifier:

```text
form_id = booking_request_modal
```

DOM contract:

```html
<form
  data-track-id="booking-form"
  data-form-id="booking_request_modal"
  data-form-location="modal"
>
```

Form IDs MUST remain stable even if:

* form heading changes;
* field labels change;
* language changes;
* CSS classes change;
* form location in the DOM changes.

---

## 6. `form_location`

Current controlled values:

```text
contact_page
modal
```

`form_location` represents semantic placement, not DOM hierarchy.

For example:

```text
modal
```

is preferred over values such as:

```text
div_3
overlay_container
right_column
```

---

## 7. Phone Identity

Tracked phone links must expose both:

```text
phone_label
phone_location
```

### `phone_label`

Current controlled values:

```text
general
sales
rental
support
```

These values represent semantic contact roles.

They MUST remain independent of the actual phone number.

For example, if the Sales phone number changes:

```text
+48 555 100 101
→
new number
```

the semantic value remains:

```text
sales
```

### DOM Example

```html
<a
  href="tel:+48555100101"
  data-track-id="phone-link"
  data-phone-label="sales"
  data-phone-location="product_detail"
>
```

---

## 8. Phone Location

Current `phone_location` values include:

```text
footer
product_detail
service_detail
rental_page
contact_page
```

The value represents semantic placement.

It MUST NOT depend on translated section headings or CSS selectors.

If a new stable placement is introduced, a new controlled value may be added to the specification.

---

## 9. Email Identity

Tracked email links should use the same semantic pattern as phone links.

### `email_label`

Current controlled values:

```text
general
sales
rental
support
```

### Typical `email_location`

```text
footer
product_detail
service_detail
rental_page
contact_page
```

Example:

```html
<a
  href="mailto:sales@northstar.example"
  data-track-id="email-link"
  data-email-label="sales"
  data-email-location="product_detail"
>
```

If email tracking remains part of the shared frontend contract, `email-link` should be treated as an additional controlled `data-track-id` value.

---

## 10. Booking Type

`booking_type` identifies the high-level booking category.

Controlled values:

```text
equipment_reservation
session_booking
```

These values MUST NOT be derived from:

* page title;
* button text;
* translated modal heading;
* URL keywords when application state already contains the booking type.

Preferred source:

```text
application booking data
```

---

## 11. Booking Item ID

`booking_item_id` identifies the specific business entity being booked.

Examples:

```text
sup_board_001
guided_trip_001
sup_lesson_private
```

Actual final values must come from the NORTHSTAR application data model.

A booking item ID MUST:

* be stable;
* uniquely identify the item within the relevant business domain;
* survive UI copy changes;
* survive translation changes;
* remain consistent across the complete booking flow.

It MUST NOT be derived dynamically from:

```text
visible title
button text
DOM position
translated label
```

---

## 12. Booking CTA DOM Contract

Example:

```html
<button
  data-track-id="booking-cta"
  data-booking-type="equipment_reservation"
  data-booking-item-id="sup_board_001"
>
```

The same application-level values used for Data Layer events SHOULD populate the DOM tracking metadata.

Do not maintain separate hardcoded values for:

```text
DOM
Data Layer
application state
```

when they represent the same business identity.

---

## 13. Booking Form Context

Where useful for QA and debugging, the currently selected booking context MAY also be exposed on the booking form.

Example:

```html
<form
  data-track-id="booking-form"
  data-form-id="booking_request_modal"
  data-form-location="modal"
  data-booking-type="equipment_reservation"
  data-booking-item-id="sup_board_001"
>
```

These attributes should reflect the current selected booking entity.

They MUST NOT retain stale values from a previously opened booking flow.

---

## 14. Booking ID

`booking_id` identifies a confirmed booking outcome.

Current application references follow a format similar to:

```text
BK-FLZZ7E
```

The frontend MUST use the application-generated confirmed identifier.

It MUST NOT create an analytics-only replacement when a real booking reference already exists.

`booking_id` is primarily intended for:

* technical duplicate prevention;
* QA;
* debugging;
* correlation.

It does not automatically need to be sent to GA4.

---

## 15. Submission ID

`submission_id` identifies a confirmed Contact form submission.

Current application references follow a format similar to:

```text
MSG-K7SL7D
```

The application-generated identifier must be used.

Like `booking_id`, this identifier is primarily useful for:

* technical deduplication;
* QA;
* debugging;
* correlation.

It should not automatically be treated as a GA4 reporting dimension.

---

## 16. Service Identity

Specific service entities use:

```text
service_id
```

Current values:

```text
sup_lessons
equipment_rental
guided_trips
```

Example DOM contract:

```html
<a
  href="/services/guided-trips"
  data-track-id="service-link"
  data-service-id="guided_trips"
>
```

Data Layer event:

```javascript
dataLayer.push({
  event: 'service_link_click',
  service_id: 'guided_trips'
});
```

The DOM value and Data Layer value MUST come from the same semantic source.

---

## 17. Service IDs vs URLs

The service URL is not the primary business identity.

For example:

```text
/services/guided-trips
```

may currently correspond to:

```text
service_id = guided_trips
```

but the stable contract is the `service_id`.

If routing changes later:

```text
/services/guided-trips
→
/experiences/guided-water-trips
```

the business identifier may remain:

```text
guided_trips
```

This prevents analytics history from fragmenting because of URL changes.

---

## 18. Outbound Identity

Outbound links should expose stable semantic labels where tracking is required.

Example:

```html
<a
  href="https://example-partner.com/"
  data-track-id="outbound-link"
  data-outbound-label="partner"
  data-outbound-location="service_detail"
>
```

The outbound destination URL may change while the semantic partner identity remains stable.

Where possible, semantic labels should be preferred over raw-domain-based identity.

---

## 19. Shared Source of Truth

The preferred architecture is:

```text
application business data
↓
component props / state
├── DOM tracking attributes
└── Data Layer event payload
```

Example:

```javascript
const service = {
  id: 'guided_trips',
  name: 'Guided Trips'
};
```

The same `service.id` should drive:

```html
data-service-id="guided_trips"
```

and:

```javascript
service_id: 'guided_trips'
```

Separate manual mappings should be avoided where the application already owns the value.

---

## 20. Localization Rule

Machine identifiers MUST NOT change across languages.

Example:

```text
English UI:
Guided Trips

Polish UI:
Wycieczki z przewodnikiem

Ukrainian UI:
Поїздки з гідом

Tracking:
service_id = guided_trips
```

The same rule applies to:

```text
booking_type
booking_item_id
form_id
phone_label
phone_location
email_label
service_id
```

---

## 21. Display Names

Human-readable names MAY exist separately from stable IDs.

Example:

```text
booking_item_id = sup_board_001
booking_item_name = Loon SUP Board
```

The ID is authoritative for identity.

The name is optional descriptive context.

A display-name change MUST NOT require changing the stable item ID.

---

## 22. Identifier Stability

Once a business identifier is used in production tracking, it SHOULD remain stable.

Do not rename identifiers simply because:

* marketing copy changed;
* route changed;
* UI title changed;
* component was redesigned;
* CSS changed.

Identifier changes may fragment historical analytics data and break GTM rules.

---

## 23. Identifier Deprecation

If a business entity is permanently removed, its identifier may stop appearing in future events.

The identifier SHOULD NOT be immediately reused for a different entity.

Example:

```text
sup_board_001
```

must not later represent a completely different rental product.

---

## 24. Tracking Attribute Stability

The following attributes are tracking-interface contracts:

```text
data-track-id
data-form-id
data-form-location
data-phone-label
data-phone-location
data-email-label
data-email-location
data-booking-type
data-booking-item-id
data-service-id
data-outbound-label
data-outbound-location
```

They MUST NOT be:

* renamed;
* removed;
* reused for unrelated meaning;

without tracking review.

---

## 25. Presentation Independence

Tracking attributes MUST NOT be used as the primary styling contract.

For example, CSS should not depend on:

```css
[data-track-id="booking-cta"]
```

when a normal styling class or component API is more appropriate.

This keeps analytics metadata independent from visual implementation.

---

## 26. QA Consistency Requirement

For elements that expose both DOM metadata and Data Layer context, QA must be able to compare the values directly.

Example:

```text
DOM:
data-booking-item-id="sup_board_001"

Data Layer:
booking_item_id = sup_board_001
```

Mismatch between the two is considered a tracking defect.

---

## 27. Current Identifier Summary

| Identifier          | Purpose                      | Example                 |
| ------------------- | ---------------------------- | ----------------------- |
| `data-track-id`     | DOM tracking role            | `booking-cta`           |
| `form_id`           | stable form identity         | `contact_main`          |
| `form_location`     | semantic form placement      | `modal`                 |
| `phone_label`       | semantic phone role          | `sales`                 |
| `phone_location`    | semantic phone placement     | `product_detail`        |
| `email_label`       | semantic email role          | `support`               |
| `email_location`    | semantic email placement     | `footer`                |
| `booking_type`      | booking category             | `equipment_reservation` |
| `booking_item_id`   | specific booking entity      | `sup_board_001`         |
| `booking_id`        | confirmed booking outcome    | `BK-FLZZ7E`             |
| `submission_id`     | confirmed Contact submission | `MSG-K7SL7D`            |
| `service_id`        | specific service identity    | `guided_trips`          |
| `outbound_label`    | semantic outbound target     | `partner`               |
| `outbound_location` | semantic outbound placement  | `service_detail`        |

---

## 28. Core Rule

The general identity hierarchy is:

```text
stable business/application ID
↓
semantic Data Layer value
↓
DOM tracking metadata
↓
URL / route
↓
visible text
```

When a higher-quality source exists, lower-quality sources must not be used to reconstruct the same identity.
