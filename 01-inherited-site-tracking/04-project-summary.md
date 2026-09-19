# NORTHSTAR — Inherited-Site Tracking Project Summary

## Project Overview

I designed, implemented, and QA-tested a GTM + GA4 tracking setup for a React/SPA website under inherited-site conditions.

The implementation assumed that dedicated frontend analytics instrumentation was not always available.

Instead of requiring immediate frontend changes, the first version of the tracking architecture used the reliable signals already exposed by the website:

* DOM structure;
* link URLs;
* form attributes;
* browser events;
* route context;
* dynamically rendered modal states;
* confirmed success states.

The project focused not only on making events fire, but on defining their business meaning, measurement grain, deduplication rules, failure conditions, and QA requirements.

---

## Measurement Scope

The implementation covers four main areas.

### Contact Funnel

```text
form_view
↓
form_start
↓
form_submit
```

The tracking distinguishes between:

* meaningful form visibility;
* actual form engagement;
* confirmed successful submission.

A browser submit attempt alone is not considered a conversion.

---

### Phone CTAs

```text
phone_view
phone_click
```

Phone interactions are classified using:

```text
phone_label
phone_location
```

Examples include:

```text
sales + product_detail
general + footer
rental + rental_page
```

Semantic deduplication prevents multiple physical DOM elements from automatically becoming multiple analytical events.

---

### Booking Funnel

```text
booking_cta_click
↓
form_start
↓
form_submit
```

The inherited-site implementation tracks booking intent at category level:

```text
equipment_reservation
session_booking
```

Booking success is confirmed only when the form is replaced by a valid success state containing a unique `BK-*` reference.

---

### Service Navigation

```text
service_link_click
```

Specific service identity is derived from the destination URL.

For example:

```text
/services/sup-lessons
↓
service_id = sup_lessons
```

Generic destinations such as `/services`, `/rental`, and `/campaign` are excluded.

---

## Technical Implementation

The implementation uses a combination of:

* GTM Element Visibility;
* Just Links triggers;
* Auto-Event Variables;
* Lookup Tables;
* Custom JavaScript;
* delegated Custom HTML event listeners;
* `MutationObserver`;
* `sessionStorage`;
* internal Data Layer events;
* semantic deduplication.

The general architecture is:

```text
Browser / DOM / application fact
↓
GTM detection
↓
Semantic normalization
↓
Validation
↓
Deduplication
↓
Internal event
↓
GA4
```

This separates low-level technical activity from the final analytical event.

---

## Key QA Finding

The most important defect discovered during QA involved `phone_view`.

The initial implementation relied on GTM Element Visibility:

```text
Once per element
```

This worked at DOM-node level, but the Tracking Specification required deduplication by semantic phone identity:

```text
phone_label + phone_location
```

Two different DOM elements could represent the same phone CTA.

React could also recreate the same semantic CTA after SPA navigation.

As a result:

```text
DOM identity
≠
business identity
```

The original implementation could generate duplicate analytical events even though GTM behaved exactly as configured.

The architecture was changed to:

```text
Element Visibility
↓
derive phone_label
+
derive phone_location
↓
semantic deduplication
↓
phone_view_unique
↓
GA4 phone_view
```

Regression testing confirmed that:

* duplicate physical elements were suppressed;
* SPA-recreated elements did not create duplicate semantic events;
* different phone identities remained eligible;
* a hard reload correctly reset the browser-document lifecycle.

---

## QA Approach

The implementation was tested using:

* GTM Preview / Tag Assistant;
* GA4 DebugView;
* positive test cases;
* negative test cases;
* duplicate-event tests;
* SPA navigation;
* hard reloads;
* localization testing;
* regression testing.

Validation focused on more than tag firing.

Each event was checked against:

```text
business meaning
+
trigger conditions
+
required parameters
+
semantic grain
+
negative conditions
+
lifecycle behavior
```

All final events passed QA.

---

## Architecture Decisions

One of the main goals of the project was to distinguish between logic that reasonably belongs in GTM and logic that should be exposed by the application.

Simple and stable inference was considered acceptable when based on signals such as:

* stable URLs;
* form actions;
* small semantic mappings;
* predictable route context;
* confirmed success states.

Frontend instrumentation is preferred when GTM would otherwise need to:

* reconstruct application state;
* maintain large business mappings;
* preserve item identity across multiple screens;
* depend heavily on translated UI text;
* duplicate business logic already known by the application.

The project therefore follows the principle:

> **GTM should act as an integration and measurement layer, not as a second frontend application.**

---

## Example: Booking Architecture Boundary

The current implementation tracks:

```text
booking_type
```

because two stable booking categories can be inferred with reasonable confidence.

It intentionally does not attempt full item-level attribution such as:

```text
booking_item_id
```

across:

```text
CTA click
↓
modal
↓
form_start
↓
successful booking
```

That identity should be provided and preserved by the application.

This limitation becomes the basis for the next phase of the NORTHSTAR project: a developer-facing frontend tracking specification and a structured Data Layer implementation.

---

## Final Result

The inherited-site implementation successfully tracks:

* Contact form visibility, engagement, and confirmed submissions;
* phone CTA visibility and clicks;
* booking intent, engagement, and confirmed submissions;
* navigation to specific service pages.

The project includes:

* a formal Tracking Specification;
* implementation documentation;
* semantic deduplication rules;
* GTM Preview evidence;
* GA4 DebugView evidence;
* a documented QA defect;
* root-cause analysis;
* regression testing;
* documented architecture limitations.

### Final Status

**Implementation: PASS**

**QA: PASS**

**Next phase:** Frontend Tracking Specification and Data Layer redesign.
