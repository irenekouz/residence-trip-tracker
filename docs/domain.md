# Residence Trip Tracker

## 1. Entities
- **User**: `userId`, `exportTemplate` (optional string). 
  - One user can have multiple legal documents and multiple trips.
  - If `exportTemplate` is `null`, the default template is used.
- **LegalDocument**: `documentId`, `userId`, `documentName`, `documentType`, `startDate`, `endDate`.
  - `userId` references `User`.
  - `documentType` is `visa` or `TRP` (temporary residence permit).
  - `documentName` is an optional free-text label entered by the user, for example "residence card 2023-2024". It is used only for display and does not affect any calculation. It is stored as `null` when the user leaves it empty.
- **Trip**: `tripId`, `userId`, `startDate`, `endDate`, `countryCode`.
  - `userId` references `User`.
  - Only trips outside Poland are stored.
  - `countryCode` is the country whose border was crossed.
  - `countryCode` is an ISO 3166-1 alpha-2 code and cannot be `PL`.

## 2. Computed relations (not stored)
- **Trip validity**: a trip is valid when some document of the same user covers `trip.startDate` and some document of the same user covers `trip.endDate`.
  The two documents can be the same or different.
  A document covers a day when `document.startDate <= day` and `day <= document.endDate`.
  Validity does not depend on the current date.
- **Trip errors**: a list of errors for a trip, computed on read, not stored. Each error has:
  - `code`: `START_NOT_COVERED` or `END_NOT_COVERED`.
  - `date`: the uncovered date.
  - A trip has error when its error list is not empty.
  - Trip errors do not depend on `ReferenceDate`.

## 3. Computed properties (not stored)
- **TripLength** (in days): counted inclusively, so both the departure day and the return day are included.
  - Formula: `endDate - startDate + 1`
  - Examples:
    - Left and returned on 10 March → 1 day.
    - Left on 10 March, returned on 11 March → 2 days.
- **TripJustification**: distinct `documentType` values of the documents of the same user that cover `trip.startDate` or `trip.endDate`.
  Only the type is returned, not the document itself. Empty when no document covers either date.
- **DisplayName**: `documentName` if it is set, otherwise `{documentType} {startDate}-{endDate}`, with dates formatted as `DD.MM.YYYY`. Example: `TRP 01.05.2024-30.04.2025`.
  Computed on read, so it updates when the dates change.
- **ReferenceDate**: the date from which the 5-year window is counted. It is the filing date if the user provided one, otherwise today's date in the user's locale, sent by the client. If the client does not send it, the server uses the current UTC date. Dates are stored without time and without time zone.
- **Window**: the period from `windowStart` to `windowEnd`, both boundary days included.
  - `windowEnd` is `ReferenceDate`.
  - `windowStart` is `ReferenceDate - 5 years`. If that date does not exist (29 February), it is 1 March.
- **TripDaysInWindow** (in days): the number of days of a trip that fall inside the window. Counted inclusively, like `TripLength`.
  - Formula: `max(0, min(trip.endDate, windowEnd) - max(trip.startDate, windowStart) + 1)`
  - A trip that is partly inside the window counts only the days inside it.
  - Days after `ReferenceDate` are not counted.
- **TotalDaysInWindow**: the number of distinct calendar days inside the window that belong to at least one trip.
  A day shared by two trips (return from one, departure on the next) is counted once.
  Only days are counted, not months.
- **Date formats**: the API sends and receives dates as `YYYY-MM-DD`. `DD.MM.YYYY` is used only for display and export (`DisplayName`, `{startDate}`, `{endDate}`).

## 4. Scenarios

Unless stated otherwise, `ReferenceDate` is 2026-11-22.

Dates in scenarios are written as `YYYY-MM-DD`, except where the expected result is an exported or displayed value.

| # | Scenario | Expected result |
|---|---|---|
| 1 | User left Poland and returned on the same day. | Trip length is 1 day. |
| 2 | User left Poland and returned on the next day. | Trip length is 2 days. |
| 3 | A visa covers both trip dates. | Justification is `visa`. |
| 4 | A TRP covers both trip dates. | Justification is `TRP`. |
| 5 | A visa and a TRP both cover both trip dates. | Justification is `visa` and `TRP`. |
| 6 | A visa covers `startDate`, a TRP covers `endDate`. | Trip is valid. Justification is `visa` and `TRP`. |
| 7 | A TRP covers `startDate`, a visa covers `endDate`. | Trip is valid. Justification is `visa` and `TRP`. |
| 8 | 2 TRPs overlap and both cover the trip dates. | Justification is `TRP`. |
| 9 | No document covers `startDate` or `endDate`. | Trip is saved. On read, it has errors for each uncovered date. Justification is empty. |
| 10 | Departure is covered by a visa, return is not covered by any document. | Trip is saved. On read, it has one error: `END_NOT_COVERED` with the return date. Justification is `visa`. |
| 11 | A TRP is the only document covering the trip's `endDate`. User deletes that TRP. | The document is deleted. On read, the trip has an `END_NOT_COVERED` error with the return date. |
| 12 | User saves a trip or a document with `endDate` earlier than `startDate`. | Record is rejected with a validation error. Nothing is saved. |
| 13 | A document's `endDate` equals the trip's `startDate` (or a document's `startDate` equals the trip's `endDate`). | The boundary day is covered. The document is included in the justification, and no error is produced for that date. |
| 14 | A document's `endDate` is one day before the trip's `startDate`, and no other document covers `startDate`. | `START_NOT_COVERED` is produced for the trip's `startDate`. |
| 15 | A trip lies fully inside the window. | `TripDaysInWindow` equals `TripLength`. |
| 16 | A trip lies fully before the window. | `TripDaysInWindow` is 0. |
| 17 | A trip is 2021-11-20 to 2021-11-24. | Only 2021-11-22 to 2021-11-24 count: `TripDaysInWindow` is 3. |
| 18 | A trip is 2026-11-20 to 2026-11-25. | Days after `ReferenceDate` do not count: `TripDaysInWindow` is 3. |
| 19 | The user provides a filing date of 2026-03-10. | The window is 2021-03-10 to 2026-03-10, and `TotalDaysInWindow` is recalculated for it. |
| 20 | The user has no trips. | `TotalDaysInWindow` is 0. |
| 21 | A trip is on 2021-11-22 only (`ReferenceDate` is 2026-11-22). | The boundary day is inside the window: `TripDaysInWindow` is 1. |
| 22 | A trip is on 2021-11-21 only (`ReferenceDate` is 2026-11-22). | The day is before the window: `TripDaysInWindow` is 0. |
| 23 | The user provides a filing date of 2027-03-01 and has a planned trip 2027-01-10 to 2027-01-15. | The window is 2022-03-01 to 2027-03-01. The planned trip counts: `TripDaysInWindow` is 6. |
| 24 | User saves a trip that shares at least one day with another trip of the same user, other than the return/departure day case. | The trip is rejected with a validation error. |
| 25 | User edits dates of an existing trip so that they do not overlap other trips. | The trip is saved. It is not compared with its own previous dates. |
| 26 | Trip A is 2026-03-05 to 2026-03-10, trip B is 2026-03-10 to 2026-03-12. | Both trips are saved. `TotalDaysInWindow` is 8, the day 2026-03-10 is counted once. |
| 27 | Trip A is 2026-03-05 to 2026-03-10, trip B is 2026-03-09 to 2026-03-12. | Trip B is rejected with a validation error. |
| 28 | `ReferenceDate` is 2024-02-29, a trip is on 2019-02-28 only. | The window starts on 2019-03-01. `TripDaysInWindow` is 0. |
| 29 | `ReferenceDate` is 2024-02-29, a trip is on 2019-03-01 only. | The boundary day is inside the window. `TripDaysInWindow` is 1. |
| 30 | `ReferenceDate` is not provided. | The server uses the current UTC date. |
| 31 | Trip A is 2026-03-10 to 2026-03-10, trip B is 2026-03-10 to 2026-03-10. | Both trips are saved. Each has `TripLength` 1. `TotalDaysInWindow` counts 2026-03-10 once. |
| 32 | Trip A is 2026-03-10 to 2026-03-10, trip B is 2026-03-10 to 2026-03-14. | Both trips are saved. `TotalDaysInWindow` is 5. |
| 33 | Trip A is 2026-03-10 to 2026-03-14, trip B is 2026-03-10 to 2026-03-14. | Trip B is rejected with a validation error. |
| 34 | Trip A is 2026-03-10 to 2026-03-10, trip B is 2026-03-10 to 2026-03-14, trip C is 2026-03-05 to 2026-03-10. | Export order is C, A, B (by `startDate`, then `endDate`). |
| 35 | Two trips have the same `startDate` and `endDate`. | Export order is by `tripId`, so repeated exports give the same result. |
| 36 | A trip is fully before the window. | The trip is not in the export. |
| 37 | A trip is 2021-11-20 to 2021-11-24 (`ReferenceDate` is 2026-11-22). | The trip is in the export with dates `20.11.2021` and `24.11.2021`. `{tripLength}` is 5. |
| 38 | Default template, trip to DE, 2026-03-10 to 2026-03-14, justification `visa` and `TRP`. | The line is `DE - 10.03.2026 - 14.03.2026 - visa, TRP`. |
| 39 | The user saves a template with an unknown placeholder. | The template is rejected with a validation error. |
| 40 | A one-day trip, no document covers that day. | Two errors on read, `START_NOT_COVERED` and `END_NOT_COVERED`, both with the same date. |

## 5. Invariants
- A user can read, create, update and delete only their own documents and trips.
- A user cannot have two documents of the same `documentType` with the same `startDate`.
- For both `LegalDocument` and `Trip`, `endDate` must be on or after `startDate`.
- Trips of the same user cannot overlap, with one exception: the return day of one trip may be the departure day of another.
  Trips `a` and `b` are rejected when they share at least one day
  (`a.startDate <= b.endDate` and `b.startDate <= a.endDate`)
  and neither `a.endDate == b.startDate` nor `b.endDate == a.startDate`.
  The check runs on create and on update. When a trip is updated, it is not compared with itself.
- Coverage of trip dates by documents is not checked on input.
- A record that violates any invariant is rejected with a validation error and is not saved.
- Rationale for the return/departure exception: the legal requirement concerns passport stamps and passport checks (for example at hotels), so leaving and returning on the same day is a normal event. Two one-day trips on the same date are allowed.
- `documentName` that is empty or contains only whitespace is stored as `null`.


## 6. Export
- The user defines the export format as a template string with placeholders. The template is stored per user.
- Available placeholders:
  - `{countryCode}`: `trip.countryCode`
  - `{startDate}`, `{endDate}`: trip dates, always formatted as `DD.MM.YYYY`
  - `{tripLength}`: `TripLength`
  - `{justification}`: `TripJustification` types joined with `, ` in a fixed order (`visa`, then `TRP`)
- Default template: `{countryCode} - {startDate} - {endDate} - {justification}`
- Only trips that have at least one day inside the window are exported (`TripDaysInWindow > 0`). The window is defined by `ReferenceDate`.
- Dates are never clipped to the window. A trip that is partly inside the window is exported with its actual `startDate` and `endDate`, and `{tripLength}` is the full `TripLength`.
- A trip with an empty justification is exported with an empty value for `{justification}`.
- Unknown placeholders are rejected when the template is saved.
- Ordering: trips are sorted by `startDate`, then by `endDate`, then by `tripId`, all ascending.
  - `tripId` is only a tie-breaker to keep the order stable. It does not reflect the order of entry.