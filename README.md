# NORTHSTAR Tracking Engineering

End-to-end web tracking engineering case built around a React/SPA website.

The project demonstrates how the same measurement requirements can be implemented under different frontend conditions:

1. **Inherited-site tracking** — GTM and GA4 implementation using the existing DOM, URLs, browser events, application states, and limited frontend signals.
2. **Frontend tracking specification** — developer requirements for a tracking-ready frontend, including stable identifiers and structured `dataLayer` events.
3. **DataLayer-based implementation** — reimplementation of the same measurement model using frontend-provided business context instead of reconstructing application state inside GTM.

The goal is not only to configure tags, but to design a maintainable tracking architecture, validate it, identify its limitations, and decide where responsibility should belong between the application and GTM.

---

## Project Architecture

```text
Existing React / SPA website
↓
Measurement requirements
↓
Inherited-site GTM implementation
↓
QA and debugging
↓
Architecture limitations identified
↓
Frontend tracking specification
↓
Tracking-ready frontend
↓
DataLayer-based GTM implementation
↓
QA and architecture comparison
```

---

## Completed Case

### Phase 1 — Inherited-Site Tracking

The first implementation is complete and includes:

* tracking specification;
* GTM implementation notes;
* semantic deduplication rules;
* GTM Preview evidence;
* GA4 DebugView evidence;
* documented QA defect;
* root-cause analysis;
* regression testing;
* architecture limitations.

One important issue discovered during QA involved `phone_view`.

The original implementation relied on GTM `Once per element`, but the required analytical grain was based on:

```text id="8g5m82"
phone_label + phone_location
```

This caused a mismatch between DOM-element identity and business identity.

The implementation was redesigned to apply semantic deduplication before the final GA4 event.

![GA4 DebugView — Booking flow](01-inherited-site-tracking/screenshots/ga4-debugview/02-ga4-debugview-booking-form-submit.png)

**Explore the completed case:**

* [Phase 1 Overview](01-inherited-site-tracking/README.md)
* [Tracking Specification](01-inherited-site-tracking/01-tracking-spec.md)
* [Implementation Notes](01-inherited-site-tracking/02-implementation-notes.md)
* [QA Report](01-inherited-site-tracking/03-qa-report.md)
* [Project Summary](01-inherited-site-tracking/04-project-summary.md)

---

## Phase 1 — Inherited-Site Tracking

**Status: Completed**

The first phase assumes that dedicated analytics instrumentation is not available.

Tracking is implemented using the signals already exposed by the website, including:

* DOM structure;
* link URLs;
* form attributes;
* browser events;
* route context;
* success-page states;
* dynamically rendered modal states.

Main technologies and techniques:

* Google Tag Manager;
* Google Analytics 4;
* Element Visibility;
* Just Links triggers;
* Auto-Event Variables;
* Lookup Tables;
* Custom JavaScript;
* delegated Custom HTML listeners;
* `MutationObserver`;
* `sessionStorage`;
* internal Data Layer events;
* semantic deduplication.

### Measurement Areas

#### Contact

* `form_view`
* `form_start`
* `form_submit`

#### Phone

* `phone_view`
* `phone_click`

#### Booking

* `booking_cta_click`
* `form_start`
* `form_submit`

#### Service Navigation

* `service_link_click`

### Documentation

* [Inherited-Site Tracking Overview](01-inherited-site-tracking/README.md)
* [Tracking Specification](01-inherited-site-tracking/01-tracking-spec.md)
* [Implementation Notes](01-inherited-site-tracking/02-implementation-notes.md)
* [QA Report](01-inherited-site-tracking/03-qa-report.md)
* [Project Summary](01-inherited-site-tracking/04-project-summary.md)

---

## QA and Debugging

The implementation was validated using:

* GTM Preview / Tag Assistant;
* GA4 DebugView;
* positive test cases;
* negative test cases;
* semantic deduplication tests;
* SPA navigation tests;
* localization tests;
* hard-reload/reset tests;
* regression testing.

One important issue discovered during QA involved `phone_view`.

GTM Element Visibility was originally configured with `Once per element`. This prevented duplicate firing for the same DOM node, but it did not match the required analytical grain.

Two different DOM elements could represent the same semantic phone CTA, and React could recreate the same CTA after SPA navigation.

The final solution introduced semantic deduplication based on:

```text
phone_label + phone_location
```

before sending the final event to GA4.

This distinction between **DOM identity** and **business identity** became one of the main architecture lessons from the inherited-site implementation.

---

## Phase 2 — Frontend Tracking Specification

**Status: Completed**

The second phase converts the measurement requirements, QA findings, and architecture limitations identified during the inherited-site implementation into a structured developer-facing tracking specification.

The objective is to move business context closer to the application and reduce the amount of application-state reconstruction required inside GTM.

The target responsibility model is:

```text
Application
→ owns business truth and application state

Data Layer
→ exposes a stable tracking interface

GTM
→ owns analytics policy, transformations, deduplication, and destinations
```

The specification defines:

* canonical frontend tracking events;
* required and optional event parameters;
* parameter types and allowed values;
* stable business identifiers;
* tracking-related DOM attributes;
* form and booking lifecycle behavior;
* SPA state and identity propagation;
* technical duplicate prevention;
* analytical deduplication boundaries;
* confirmed success semantics;
* PII boundaries;
* legacy tracking migration rules;
* developer-facing QA acceptance criteria.

### Canonical Application Events

The frontend tracking contract defines application-level events such as:

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

can later be mapped by GTM to:

```text
GA4
Google Ads
Meta
other analytics or advertising destinations
```

without requiring the frontend tracking contract to change.

### Architecture Principle

The specification follows one central rule:

> **Frontend owns application truth. GTM owns analytics policy.**

The frontend is responsible for facts it already knows directly, including:

* selected business entity;
* current booking identity;
* real user interactions;
* meaningful form engagement;
* confirmed application success;
* stable machine-readable values;
* technical duplicate prevention.

GTM remains responsible for:

* visibility thresholds;
* analytical deduplication;
* destination-specific event mapping;
* GA4 event naming;
* advertising-platform routing;
* reporting-specific transformations.

### Data Layer Design

Required event context must be provided atomically in the same `dataLayer.push()`.

Example:

```javascript
dataLayer.push({
  event: 'booking_cta_click',
  booking_type: 'equipment_reservation',
  booking_item_id: 'sup_board_001'
});
```

Required context must not depend on stale values from previous Data Layer pushes.

Business identifiers must remain stable across:

```text
initial page load
SPA navigation
localization
component rerenders
multi-step application flows
```

The specification also defines how booking identity must persist across:

```text
booking_cta_click
↓
booking_form_start
↓
booking_form_submit_success
```

without GTM reconstructing that identity from URLs, translated text, modal content, or DOM hierarchy.

### QA and Acceptance Criteria

The specification includes developer-facing QA requirements covering:

* required event parameters;
* current application state;
* language-independent identifiers;
* technical duplicate prevention;
* preservation of legitimate repeated interactions;
* confirmed form-success semantics;
* booking identity continuity;
* SPA navigation;
* localization;
* stale-state prevention;
* legacy tracking migration;
* PII boundaries;
* regression protection for existing site behavior.

The frontend tracking implementation should only be accepted when the canonical event contract, lifecycle behavior, identity propagation, and QA requirements all pass.

### Documentation

* [Frontend Tracking Specification Overview](02-frontend-tracking-spec/README.md)
* [Developer Requirements](02-frontend-tracking-spec/01-developer-requirements.md)
* [Data Layer Event Specification](02-frontend-tracking-spec/02-data-layer-event-spec.md)
* [Tracking Identifiers](02-frontend-tracking-spec/03-tracking-identifiers.md)
* [State and Lifecycle](02-frontend-tracking-spec/04-state-and-lifecycle.md)
* [QA Acceptance Criteria](02-frontend-tracking-spec/05-qa-acceptance-criteria.md)

---

## Phase 3 — DataLayer-Based Implementation

**Status: Planned**

The next phase will implement the same measurement model using the structured frontend tracking contract defined in Phase 2.

The architecture will move from:

```text
DOM / route / UI inference
↓
Custom GTM logic
↓
semantic reconstruction
↓
GA4
```

toward:

```text
Application business fact
↓
canonical dataLayer event
↓
GTM validation / transformation
↓
GA4 / Ads / other destinations
```

This will allow the two implementations to be compared in terms of:

* reliability;
* maintainability;
* dependency on DOM structure;
* SPA behavior;
* semantic accuracy;
* implementation complexity;
* debugging effort;
* responsibility split between frontend and GTM.
