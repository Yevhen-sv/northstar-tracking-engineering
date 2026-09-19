# NORTHSTAR — Inherited-Site Tracking QA Report

## 1. QA Scope

This QA report validates the final inherited-site GTM + GA4 implementation for the NORTHSTAR React/SPA website.

The tested measurement areas were:

### Contact Tracking

* `form_view`
* `form_start`
* `form_submit`

### Phone Tracking

* `phone_view`
* `phone_click`

### Booking Tracking

* `booking_cta_click`
* `form_start`
* `form_submit`

### Service Navigation

* `service_link_click`

The purpose of QA was not only to verify that events fired, but also to validate:

* positive trigger conditions;
* negative trigger conditions;
* required parameters;
* event grain;
* duplicate prevention;
* legitimate repeated outcomes;
* SPA navigation behavior;
* browser lifecycle behavior;
* localization behavior;
* final GA4 delivery.

---

## 2. QA Method

Validation was performed using:

* GTM Preview / Tag Assistant;
* GA4 DebugView;
* browser DevTools;
* controlled positive test cases;
* controlled negative test cases;
* semantic deduplication tests;
* SPA navigation tests;
* hard reload tests;
* localization tests;
* regression testing after fixes.

The general QA workflow was:

```text
Tracking Specification
↓
Expected behavior
↓
Controlled user action
↓
GTM Preview validation
↓
GA4 validation
↓
PASS / FAIL
↓
Root-cause analysis
↓
Fix
↓
Regression test
```

A test was considered successful only when the implementation matched the semantic rules defined in the Tracking Specification.

---

## 3. QA Result Summary

| Tracking Area            | Final Status         |
| ------------------------ | -------------------- |
| Contact tracking         | PASS                 |
| Phone tracking           | PASS after QA-01 fix |
| Booking tracking         | PASS                 |
| Service-link tracking    | PASS                 |
| GTM Preview validation   | PASS                 |
| GA4 DebugView validation | PASS                 |

### Overall Result

**PASS**

One real tracking defect was discovered during QA:

```text
phone_view
→ DUPLICATE / WRONG_GRAIN
```

The issue was fixed and regression-tested successfully.

---

## 4. Contact Tracking QA

### 4.1 Contact `form_view`

| Test                                                               | Expected        | Result |
| ------------------------------------------------------------------ | --------------- | ------ |
| Form remains below 75% viewport visibility                         | No event        | PASS   |
| Form reaches ≥75% but remains visible for less than 1 second       | No event        | PASS   |
| Form remains ≥75% visible for at least 1 second                    | One `form_view` | PASS   |
| Same form becomes visible again in the same browser-page lifecycle | No duplicate    | PASS   |

Validated parameters:

```text
form_id = contact_main
form_location = contact_page
```

---

### 4.2 Contact `form_start`

| Test                                                 | Expected           | Result |
| ---------------------------------------------------- | ------------------ | ------ |
| User only focuses a field                            | No event           | PASS   |
| User enters whitespace only                          | No event           | PASS   |
| First non-whitespace character is entered            | One `form_start`   | PASS   |
| Subject changes away from default `order`            | One `form_start`   | PASS   |
| Consent checkbox is the only interaction             | No event           | PASS   |
| Additional meaningful interactions after first start | No duplicate       | PASS   |
| Optional phone field receives meaningful input       | Valid `form_start` | PASS   |

Input validity was intentionally not required.

For example, an incomplete email can still represent real form engagement.

Validated parameters:

```text
form_id = contact_main
form_location = contact_page
```

---

### 4.3 Contact `form_submit`

| Test                                                | Expected            | Result |
| --------------------------------------------------- | ------------------- | ------ |
| Invalid form submission                             | No `form_submit`    | PASS   |
| Valid submission followed by confirmed success page | One `form_submit`   | PASS   |
| Reload success page                                 | No duplicate        | PASS   |
| Open old success URL directly                       | No false conversion | PASS   |
| Submit another legitimate Contact request           | New `form_submit`   | PASS   |

The success event was validated only after correlation between:

```text
valid pending submission
+
/contact/success
+
valid MSG-* reference
```

This confirmed that a browser submit attempt alone was not treated as a successful conversion.

---

## 5. Phone Tracking QA

### 5.1 `phone_view`

The initial implementation exposed a real architecture defect.

The Tracking Specification required:

```text
one phone_view
per unique
phone_label + phone_location
per browser document lifecycle
```

However, the initial implementation relied on GTM Element Visibility:

```text
Once per element
```

This deduplicated by physical DOM-element identity rather than semantic phone identity.

The defect is documented as `QA-01`.

After the fix, the following regression tests were performed:

| Test                                                        | Expected                     | Result |
| ----------------------------------------------------------- | ---------------------------- | ------ |
| Two physical footer links both represent `general + footer` | One GA4 `phone_view`         | PASS   |
| `sales + product_detail` becomes visible                    | One `phone_view`             | PASS   |
| SPA navigate away and return to the same semantic CTA       | No second semantic event     | PASS   |
| A different semantic phone key becomes visible              | New `phone_view`             | PASS   |
| Hard reload and expose the same phone CTA again             | Event becomes eligible again | PASS   |

---

### 5.2 `phone_click`

| Test                                                       | Expected                               | Result |
| ---------------------------------------------------------- | -------------------------------------- | ------ |
| First click on `general + footer`                          | One `phone_click`                      | PASS   |
| Click another physical link with the same semantic key     | No duplicate                           | PASS   |
| Click another phone label                                  | New event                              | PASS   |
| Same label in another location                             | New event                              | PASS   |
| SPA navigate away and return, then click same semantic key | No duplicate within document lifecycle | PASS   |
| Hard reload, then click same semantic key                  | Event eligible again                   | PASS   |

Validated semantic parameters:

```text
phone_label
phone_location
```

---

## 6. Booking Tracking QA

### 6.1 `booking_cta_click`

| Test                                             | Expected                               | Result |
| ------------------------------------------------ | -------------------------------------- | ------ |
| Click valid booking CTA and booking form appears | One event                              | PASS   |
| Click nested text/span inside valid CTA          | CTA still detected                     | PASS   |
| Click nested SVG/icon inside valid CTA           | CTA still detected                     | PASS   |
| Click unrelated button such as Subscribe         | No event                               | PASS   |
| Repeat CTA interaction for same `booking_type`   | No duplicate                           | PASS   |
| Trigger different booking category               | New event                              | PASS   |
| SPA navigation and return to same booking type   | No duplicate within document lifecycle | PASS   |
| Polish-language version                          | Same semantic tracking                 | PASS   |

Validated parameter:

```text
booking_type
```

Allowed values:

```text
equipment_reservation
session_booking
```

A booking CTA was confirmed only when the known booking form appeared within the configured confirmation window.

---

### 6.2 Booking `form_start`

| Test                                | Expected                                     | Result |
| ----------------------------------- | -------------------------------------------- | ------ |
| Focus field without entering data   | No event                                     | PASS   |
| Enter whitespace only               | No event                                     | PASS   |
| Enter meaningful text               | One `form_start`                             | PASS   |
| Select valid date                   | One `form_start`                             | PASS   |
| Change people away from default `2` | One `form_start`                             | PASS   |
| Consent checkbox only               | No event                                     | PASS   |
| Repeat qualifying interactions      | No duplicate                                 | PASS   |
| Polish-language modal               | Same semantic event                          | PASS   |
| `/services/equipment-rental` route  | `session_booking`, not equipment reservation | PASS   |

The `/services/equipment-rental` case was an important routing test.

Although the URL contains the word `equipment`, the actual booking flow is a session booking.

The implementation therefore follows the defined business mapping rather than relying on keyword guessing.

Validated parameters:

```text
form_id = booking_request_modal
form_location = modal
booking_type
```

---

### 6.3 Booking `form_submit`

| Test                                                      | Expected             | Result |
| --------------------------------------------------------- | -------------------- | ------ |
| Invalid submission                                        | No event             | PASS   |
| Browser submit event without confirmed success            | No event             | PASS   |
| Valid success with new `BK-*` reference                   | One `form_submit`    | PASS   |
| Same success state observed repeatedly                    | No duplicate         | PASS   |
| Second legitimate booking with different `BK-*` reference | New `form_submit`    | PASS   |
| Unrelated form activity                                   | No event             | PASS   |
| Polish-language flow                                      | Same semantic result | PASS   |

The browser `submit` event was confirmed to represent only a submission attempt.

Successful booking required:

```text
known booking dialog
+
booking form disappeared
+
valid BK-* reference appeared
```

A different valid booking reference represents a new legitimate business outcome.

---

## 7. Service-Link Tracking QA

| Test                                            | Expected                               | Result |
| ----------------------------------------------- | -------------------------------------- | ------ |
| Click `/services/sup-lessons`                   | `service_id = sup_lessons`             | PASS   |
| Click `/services/equipment-rental`              | `service_id = equipment_rental`        | PASS   |
| Click `/services/guided-trips`                  | `service_id = guided_trips`            | PASS   |
| Click same service from another placement       | No duplicate                           | PASS   |
| Click different service                         | New event                              | PASS   |
| Click generic `/services`                       | No event                               | PASS   |
| Click `/rental`                                 | No event                               | PASS   |
| Click `/campaign`                               | No event                               | PASS   |
| SPA return and click previously counted service | No duplicate within document lifecycle | PASS   |
| Hard reload and click same service              | Event eligible again                   | PASS   |
| `/services/sup-lessons?source=test`             | `service_id = sup_lessons`             | PASS   |

The query-string test confirmed that technical URL decoration does not change the semantic service identity.

---

## 8. QA Issue — QA-01

### Duplicate `phone_view` / Wrong Measurement Grain

| Field             | Details                   |
| ----------------- | ------------------------- |
| Issue ID          | `QA-01`                   |
| Event             | `phone_view`              |
| Issue Type        | `DUPLICATE / WRONG_GRAIN` |
| Severity          | Medium                    |
| Status            | FIXED                     |
| Regression Status | PASS                      |

### Expected Behavior

One `phone_view` should be recorded per unique:

```text
phone_label + phone_location
```

per browser document lifecycle.

Example:

```text
general + footer
```

should represent one analytical occurrence even if multiple physical links represent the same phone contact.

---

### Actual Behavior

Two duplicate scenarios were found.

#### Scenario A — Multiple Physical Elements

Two separate footer links represented:

```text
phone_label = general
phone_location = footer
```

GTM Element Visibility treated them as different elements and allowed both to trigger.

Technical behavior:

```text
DOM element #1
→ phone_view

DOM element #2
→ phone_view
```

Required semantic behavior:

```text
general|footer
→ one phone_view
```

---

#### Scenario B — React SPA Element Recreation

The `sales + product_detail` phone CTA was viewed once.

After SPA navigation away from the route and back, React recreated the DOM element.

GTM therefore saw:

```text
new DOM element
```

even though the business identity was still:

```text
sales|product_detail
```

This created another `phone_view`.

---

### Root Cause

The implementation used:

```text
Once per element
```

as if it were the analytical deduplication rule.

But:

```text
DOM identity
≠
business identity
```

GTM correctly followed its technical rule, but the technical rule did not match the measurement specification.

---

### Fix

The architecture was changed from:

```text
Element Visibility
↓
GA4 phone_view
```

to:

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

The semantic key became:

```text
phone_label + phone_location
```

Examples:

```text
general|footer
sales|product_detail
rental|rental_page
```

Native `Once per element` remains useful as a technical optimization, but it is no longer responsible for analytical event grain.

---

### Regression Result

All targeted regression tests passed:

```text
duplicate physical elements
→ suppressed

SPA-recreated same semantic CTA
→ suppressed

different semantic key
→ allowed

hard reload
→ same key eligible again
```

Final status:

**FIXED / PASS**

---

## 9. GA4 DebugView Validation

Selected final events were validated in GA4 DebugView after GTM implementation testing.

### Contact Submission

Confirmed event:

```text
form_submit
```

Confirmed semantic parameter:

```text
form_id = contact_main
```

---

### Booking Flow

DebugView confirmed the event sequence:

```text
booking_cta_click
↓
form_start
↓
form_submit
```

Confirmed booking parameter:

```text
booking_type = session_booking
```

---

### Phone Click

Confirmed event:

```text
phone_click
```

Confirmed semantic parameter:

```text
phone_label = sales
```

---

### Service Navigation

Confirmed event:

```text
service_link_click
```

Confirmed semantic parameter:

```text
service_id = guided_trips
```

GTM Preview was used as the primary implementation-debugging environment because it exposes trigger behavior, variables, Data Layer state, and complete tag parameters.

GA4 DebugView was used to confirm that selected final events and semantic parameters successfully reached GA4.

---

## 10. QA Evidence

### GTM Preview

Supporting screenshots:

```text
screenshots/gtm-preview/
├── 01-gtm-preview-contact-form-start.png
├── 02-gtm-preview-contact-form-submit.png
├── 03-gtm-preview-phone-view.png
├── 04-gtm-preview-phone-click.png
├── 05-gtm-preview-booking-cta-click.png
├── 06-gtm-preview-booking-form-start.png
├── 07-gtm-preview-booking-form-submit.png
└── 08-gtm-preview-service-link-click.png
```

### GA4 DebugView

Supporting screenshots:

```text
screenshots/ga4-debugview/
├── 01-ga4-debugview-contact-form-submit.png
├── 02-ga4-debugview-booking-form-submit.png
├── 03-ga4-debugview-phone-click.png
└── 04-ga4-debugview-service-link-click.png
```

The original pre-fix `phone_view` implementation was not preserved as a screenshot.

The QA issue is therefore documented through:

* the reproduced behavior;
* expected vs actual result;
* root-cause analysis;
* final architecture;
* regression-test evidence.

No missing evidence was recreated or fabricated.

---

## 11. Final QA Status

All events defined in the inherited-site Tracking Specification passed final validation.

The QA process demonstrated that technical firing success alone is not sufficient to prove correct tracking.

The `phone_view` defect showed that an event can fire exactly as configured in GTM while still violating the intended analytical grain.

The final implementation was therefore validated against:

```text
business meaning
+
trigger conditions
+
parameters
+
semantic grain
+
negative conditions
+
lifecycle behavior
```

rather than only checking whether a GA4 tag fired.

**Final project QA status: PASS**
