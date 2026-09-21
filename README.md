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

**Status: Planned**

The second phase will convert the measurement requirements and limitations discovered during the inherited-site implementation into a developer-facing tracking specification.

The specification will define:

* stable tracking identifiers;
* `dataLayer` event names;
* event parameters;
* allowed parameter values;
* trigger conditions;
* negative conditions;
* event grain;
* deduplication responsibilities;
* state persistence requirements;
* SPA lifecycle behavior;
* success confirmation requirements;
* QA acceptance criteria.

The purpose is to move business context closer to the application instead of reconstructing it inside GTM.

---

## Phase 3 — DataLayer-Based Implementation

**Status: Planned**

After the frontend is instrumented, the same measurement model will be implemented again using structured frontend events.

The expected architecture will move from patterns such as:

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
Application knows business event
↓
dataLayer.push(...)
↓
GTM validation / transformation
↓
GA4
```

This phase will make it possible to compare:

* implementation complexity;
* reliability;
* maintainability;
* dependency on DOM structure;
* SPA behavior;
* semantic accuracy;
* debugging effort.

---

## Architecture Principle

The project follows one central rule:

> **GTM should act as an integration and measurement layer, not as a second frontend application.**

Small and stable transformations inside GTM are reasonable.

For example:

* URL normalization;
* small lookup tables;
* semantic deduplication;
* extracting context from a stable DOM structure.

Frontend instrumentation becomes preferable when GTM would need to:

* reconstruct application state;
* maintain large business mappings;
* carry item identity across multiple screens or steps;
* depend heavily on translated UI text;
* reproduce logic already known by the application.

---

## Example: Booking Identity

The inherited-site implementation tracks booking intent at category level:

```text
booking_type =
equipment_reservation
session_booking
```

This can be derived with reasonable confidence from stable routes and modal structure.

Item-level attribution, such as:

```text
booking_item_id = guided_trip_001
```

would require the selected item identity to survive across:

```text
CTA click
↓
modal opening
↓
form_start
↓
successful booking
```

That state should be owned and exposed by the frontend rather than reconstructed inside GTM.

---

## Repository Structure

```text
northstar-tracking-engineering/
│
├── README.md
│
├── 01-inherited-site-tracking/
│   ├── README.md
│   ├── 01-tracking-spec.md
│   ├── 02-implementation-notes.md
│   ├── 03-qa-report.md
│   ├── 04-project-summary.md
│   └── screenshots/
│
├── 02-frontend-tracking-spec/
│   └── planned
│
└── 03-data-layer-implementation/
    └── planned
```

---

## Tools

* Google Tag Manager
* Google Analytics 4
* JavaScript
* Data Layer
* Browser DevTools
* React / SPA environment

---

## Current Status

The inherited-site tracking implementation and QA are complete.

The next stage is to convert the identified tracking requirements and architecture limitations into a structured frontend tracking specification.
