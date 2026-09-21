# NORTHSTAR — State and Lifecycle Specification

## 1. Purpose

This document defines lifecycle and state-management requirements for the NORTHSTAR frontend tracking interface.

The purpose is to ensure that tracking context remains correct across:

* initial document load;
* React SPA navigation;
* component mount/unmount;
* modal open/close cycles;
* form instances;
* repeated real user interactions;
* successful business transactions.

The frontend must preserve real business context while preventing stale state and technical duplicate events.

---

## 2. Lifecycle Model

The tracking implementation must distinguish between different lifecycle levels.

These concepts are not interchangeable:

```text
browser document lifecycle
SPA route lifecycle
component lifecycle
form instance lifecycle
modal lifecycle
business transaction lifecycle
```

Each canonical event must follow the lifecycle that matches the underlying application fact.

---

## 3. Browser Document Lifecycle

A browser document lifecycle begins when a new HTML document is loaded.

Examples:

```text
hard reload
direct navigation
full document navigation
```

A normal React SPA route change does not create a new browser document.

Frontend tracking state must not assume that route navigation automatically resets document-level state.

GTM may maintain document-level analytical state separately.

---

## 4. SPA Route Lifecycle

A route lifecycle begins when the application enters a route through:

* initial page load;
* SPA navigation;
* browser back/forward navigation handled by the application.

Route navigation may:

* mount new components;
* unmount existing components;
* preserve shared layout components;
* recreate business entities in the DOM.

Canonical frontend events must use current application state after each route transition.

Old route context must not leak into events generated on the new route.

---

## 5. Application Context After Route Change

After a route transition, tracking values must reflect the newly active application context.

Example:

```text
/product/item-a
↓
SPA navigation
↓
/product/item-b
```

A later event for item B must not reuse:

```text
booking_item_id = item_a
```

The correct event must use:

```text
booking_item_id = item_b
```

Stale route or entity context is considered a tracking defect.

---

## 6. Shared Components

Components such as:

```text
Header
Footer
PhoneLink
EmailLink
Navigation
```

may remain mounted across SPA navigation.

Their semantic tracking context must remain correct even if the component is reused across multiple routes.

A shared component must not derive current business context from stale route state when the relevant semantic value can be provided directly through props or application data.

---

## 7. Component Remounting

React may destroy and recreate a component while the browser document remains the same.

A component remount must not automatically be interpreted as a new analytical occurrence.

Frontend canonical events should represent real application facts, not mount events, unless the specification explicitly defines mounting as the source fact.

Example:

```text
phone-link component unmounted
↓
same semantic phone-link mounted again
```

must not itself create a `phone_click`.

Visibility measurement remains a GTM policy and may separately observe the new DOM element.

---

# 8. Contact Form Lifecycle

The Contact form has a form-instance lifecycle.

A form instance begins when the Contact form component is created and ready for user interaction.

The form remains the same instance while the user:

```text
focuses fields
types
clears values
changes fields
continues editing
```

These actions do not create a new form instance.

---

## 9. Contact `form_start` State

The Contact form instance begins in:

```text
pristine
```

After the first qualifying meaningful interaction, it transitions to:

```text
started
```

State transition:

```text
pristine
↓
meaningful interaction
↓
started
```

Only the transition from `pristine` to `started` may generate:

```text
contact_form_start
```

Further qualifying interactions in the same form instance must not create additional frontend `contact_form_start` events.

---

## 10. Contact Form Reset

Clearing field values after the form has started must not reset the current form instance back to `pristine` for tracking purposes.

Example:

```text
user types name
→ contact_form_start

user deletes name
→ no reset

user types again
→ no second contact_form_start
```

A new form component instance may become eligible for a new canonical `contact_form_start`.

---

## 11. Contact Submission Lifecycle

A successful Contact submission creates a unique business outcome identified by:

```text
submission_id
```

Example:

```text
MSG-K7SL7D
```

The success lifecycle is transaction-based rather than page-based.

One confirmed submission:

```text
submission_id = MSG-001
```

must produce one:

```text
contact_form_submit_success
```

A second confirmed submission:

```text
submission_id = MSG-002
```

represents a new business occurrence and must be observable separately.

---

## 12. Contact Success State

The frontend must not generate repeated success events merely because:

* success UI rerenders;
* success component remounts;
* state is read again;
* an effect executes again.

The same confirmed `submission_id` represents the same business outcome.

---

# 13. Booking Flow Lifecycle

Booking tracking requires persistent business identity across multiple application steps.

The core flow is:

```text
booking CTA
↓
selected booking context
↓
booking modal
↓
booking form start
↓
confirmed booking success
```

The frontend must preserve the same booking identity throughout this flow.

---

## 14. Booking Context

When the user activates a booking CTA, the application must establish the current booking context.

Minimum context:

```text
booking_type
booking_item_id
```

Example:

```text
booking_type = equipment_reservation
booking_item_id = sup_board_001
```

This context must remain associated with the active booking flow until that flow ends or is replaced by a new one.

---

## 15. Booking CTA Activation

Every real booking CTA activation is a new application interaction.

Example:

```text
user activates item A
→ booking_cta_click

user closes modal

user activates item A again
→ another booking_cta_click
```

The frontend must preserve both real interactions.

Whether analytics later counts both belongs to GTM policy.

---

## 16. Modal Open Lifecycle

Opening the booking modal creates an active booking-flow instance.

The active flow must contain the selected:

```text
booking_type
booking_item_id
```

The booking modal must not infer this identity from:

* visible heading;
* translated text;
* current URL;
* DOM position.

It should receive or consume the identity directly from application state.

---

## 17. Modal Close

Closing the booking modal ends the current visible modal instance.

The implementation must prevent stale booking state from leaking into the next booking flow.

The application may either:

* clear the active booking context on close; or
* replace it deterministically when a new booking CTA is activated.

In either implementation, the next booking interaction must use fresh current context.

---

## 18. Reopening the Same Booking

Example:

```text
open sup_board_001
↓
close modal
↓
open sup_board_001 again
```

This represents:

```text
two booking CTA activations
two modal instances
potentially two form instances
```

The frontend must not suppress the second real interaction merely because the business item is the same.

GTM may later choose to analytically deduplicate those events.

---

## 19. Switching Booking Items

Example:

```text
open item A
↓
close
↓
open item B
```

The second flow must use:

```text
booking_item_id = B
```

for all subsequent events.

The following is a defect:

```text
booking_cta_click = item B
booking_form_start = item A
```

Identity must remain coherent within each flow.

---

## 20. Booking Form Instance

Each new booking form presented for an active booking flow is a form instance.

The booking form begins in:

```text
pristine
```

and transitions to:

```text
started
```

after the first meaningful interaction.

Only this transition may generate:

```text
booking_form_start
```

for that form instance.

---

## 21. Booking Form Context

At the moment `booking_form_start` is generated, the frontend must still have access to:

```text
form_id
form_location
booking_type
booking_item_id
```

These values must describe the current active booking flow.

They must not be reconstructed by GTM.

---

## 22. Booking Form Close Before Submit

If the user:

```text
opens booking
starts form
closes modal
```

the flow ends without success.

The frontend must not generate:

```text
booking_form_submit_success
```

Closing the modal does not represent a successful business outcome.

---

## 23. Booking Success Lifecycle

A successful booking creates a transaction-like business outcome identified by:

```text
booking_id
```

Example:

```text
BK-FLZZ7E
```

The canonical success event must contain:

```text
booking_id
booking_type
booking_item_id
```

from the same booking flow.

---

## 24. Booking Success Identity Consistency

For one booking flow:

```text
booking_cta_click
booking_form_start
booking_form_submit_success
```

must preserve the same:

```text
booking_type
booking_item_id
```

Example:

```text
booking_cta_click
booking_item_id = sup_board_001

booking_form_start
booking_item_id = sup_board_001

booking_form_submit_success
booking_item_id = sup_board_001
```

A mismatch is considered a tracking defect.

---

## 25. Booking Success Technical Deduplication

The same confirmed:

```text
booking_id
```

must produce one canonical success event.

Example:

```text
BK-1001
```

must not generate repeated `booking_form_submit_success` events because of:

* rerender;
* repeated success-state observation;
* component remount;
* duplicate callback;
* repeated effect execution.

---

## 26. Multiple Legitimate Bookings

Different booking IDs represent different business outcomes.

Example:

```text
BK-1001
BK-1002
```

must remain observable as two successful bookings even if they occur within the same browser document lifecycle.

Frontend must not apply session-level or document-level suppression to real transactions.

---

# 27. Phone Lifecycle

Phone click tracking is interaction-based.

Every real eligible activation should produce:

```text
phone_click
```

The frontend must not persist an application-level suppression state such as:

```text
phone already clicked
```

for analytics purposes.

Analytical deduplication belongs to GTM.

---

## 28. Phone Visibility

Phone visibility is not a frontend event lifecycle.

Frontend only provides stable DOM metadata:

```text
phone_label
phone_location
```

GTM owns visibility lifecycle and analytical deduplication.

A React remount may create a new physical DOM element while the semantic phone identity remains unchanged.

GTM must handle that distinction according to measurement policy.

---

# 29. Service-Link Lifecycle

Every real specific-service link activation should produce:

```text
service_link_click
```

Frontend must not suppress repeat activation based on previously clicked `service_id`.

Example:

```text
guided_trips click
guided_trips click again
```

represents two real application interactions.

GTM may later choose a different analytical grain.

---

# 30. Email and Outbound Lifecycle

The same principle applies to:

```text
email_click
outbound_click
```

Every real eligible activation should remain observable.

Frontend must prevent technical duplicates but must not perform analytics-level suppression.

---

# 31. State Reset Rules

State must reset according to the underlying application concept.

### Form-start state

Reset when a genuinely new form instance is created.

### Booking context

Reset or replace when the active booking flow ends or a new booking entity is selected.

### Booking success dedupe

Keyed to confirmed `booking_id`.

### Contact success dedupe

Keyed to confirmed `submission_id`.

### Analytical session/document dedupe

Not frontend responsibility.

---

# 32. Technical State vs Analytics State

Frontend may maintain technical state such as:

```text
has_this_form_instance_started
current_booking_item
reported_booking_ids
reported_submission_ids
```

where needed to preserve application correctness.

Frontend must not maintain analytics-policy state such as:

```text
user_already_clicked_phone_this_session
service_already_counted_this_document
booking_cta_already_counted_for_analytics
```

Those rules belong downstream.

---

# 33. State Persistence Scope

Technical state should use the smallest scope necessary for the application fact.

Examples:

```text
form started state
→ component/form-instance scope

current booking context
→ active booking-flow/application state

reported booking success ID
→ enough scope to prevent duplicate source pushes

analytics session dedupe
→ GTM
```

State must not be made globally persistent without a clear requirement.

---

# 34. Avoid Unnecessary `sessionStorage`

The redesigned frontend should not use `sessionStorage` simply because the inherited GTM implementation used it.

The old Contact implementation required `sessionStorage` because GTM had to correlate a submit attempt across a document navigation.

If the application itself owns confirmed success and can push the canonical event at the correct source-of-truth moment, this workaround is no longer necessary.

Frontend persistence mechanisms should be chosen from application needs, not copied from inherited analytics workarounds.

---

# 35. Avoid DOM Observation for Application Truth

The redesigned frontend should not use `MutationObserver` to determine facts that the application already knows directly.

Example:

```text
booking created successfully
```

should come from application/backend success handling.

It should not be rediscovered by observing that:

```text
form disappeared
+
BK-* text appeared
```

DOM observation remains a fallback technique, not the preferred source of application truth.

---

# 36. Event Ordering

Typical expected booking sequence:

```text
booking_cta_click
↓
booking_form_start
↓
booking_form_submit_success
```

Typical Contact sequence:

```text
contact_form_start
↓
contact_form_submit_success
```

However, each canonical event payload must remain independently interpretable.

GTM must not need to retrieve required context from an earlier event in the sequence.

---

# 37. No Cross-Event Context Dependency

Incorrect pattern:

```text
booking_cta_click
contains booking_item_id

booking_form_start
contains no booking_item_id

GTM remembers previous item
```

Correct pattern:

```text
booking_cta_click
contains booking_item_id

booking_form_start
contains booking_item_id

booking_form_submit_success
contains booking_item_id
```

Required business context must travel with each relevant canonical event.

---

# 38. Page Reload Behavior

A hard reload creates a new browser document and new component instances.

Frontend form-instance state resets naturally with the application.

Confirmed business outcomes must still not be duplicated simply because a success view is reloaded.

Transaction identity remains authoritative:

```text
same booking_id
→ same booking outcome
```

---

# 39. Browser Back / Forward

SPA or browser history navigation must not cause stale business context to be reused.

If the user returns to a previously visited route:

* current page/entity context must be recalculated from current application state;
* no canonical click event should fire solely because history navigation occurred;
* form and booking events should only fire when their actual source conditions occur.

---

# 40. Localization Lifecycle

Language changes must not alter business identity.

If the active language changes while the application remains loaded:

```text
EN → PL → UA
```

tracking values such as:

```text
booking_type
booking_item_id
service_id
phone_label
form_id
```

must remain unchanged.

A language switch alone must not create business-interaction events.

---

# 41. Lifecycle QA Principle

For every tracked flow, QA must verify both:

```text
identity continuity
```

and:

```text
correct reset behavior
```

Examples:

```text
same booking flow
→ same booking_item_id across events

new booking flow
→ fresh current context

same confirmed booking_id
→ no technical duplicate

different booking_id
→ new success event
```

---

# 42. Lifecycle Summary

| State / Entity        | Lifecycle Owner            | Reset / New Occurrence           |
| --------------------- | -------------------------- | -------------------------------- |
| Browser document      | Browser                    | full document load               |
| SPA route             | Application router         | route transition                 |
| Contact form instance | Frontend component         | new form instance                |
| Contact started state | Frontend                   | new form instance                |
| Contact success       | Business transaction       | new `submission_id`              |
| Booking context       | Frontend application state | new booking flow / selected item |
| Booking form instance | Frontend component         | new booking-form instance        |
| Booking success       | Business transaction       | new `booking_id`                 |
| Phone click           | User interaction           | every real activation            |
| Service click         | User interaction           | every real activation            |
| Analytics dedupe      | GTM                        | according to measurement policy  |

---

# 43. Core Lifecycle Rule

The frontend must preserve enough state to accurately describe real application facts, but it must not encode downstream analytics policy.

The intended boundary is:

```text
Frontend state
→ what is true in the application

GTM state
→ how analytics chooses to count it
```
