# OmanSSIMPairing Documentation

This document details the automated process, **processOmanSSIMPair**, designed to establish roundabout pairings for flights originating from Oman (designated by `_IFCSClient_s eq 'Oman'`).

The utility analyzes flight data received via SSIM (Standard Schedules Information Manual) updates, specifically linking an outbound flight from Muscat (MCT) to a subsequent inbound flight back to MCT, forming a complete trip sequence. This pairing logic is essential for downstream crew and aircraft assignment systems.

## 1.1 Core Business Logic (Oman Pairing Rule)

The pairing logic is based on the following heuristic, commonly used for short-haul or regional round trips:

- **Outbound Flight (First):** Must depart from Muscat (MCT) and have an Odd Flight Number.
- **Return/Connecting Flight(s):** Must be a subsequent flight that:
  - Departs from the arrival station of the previous flight.
  - Has a departure time within 24 hours of the previous flight's arrival time.
  - Has a flight number that is either the same as the outbound flight number or the next sequential number (i.e., odd → even → odd...).
- **Pairing Conclusion (Last):** The sequence ends when the final flight leg arrives back at Muscat (MCT).

---

## 2. Technical Process Flow

The `processOmanSSIMPair` function executes a multi-step query, filtering, and pairing routine.

### 2.1 Phase 1: Data Retrieval and Filtering

| Step | Action          | Filtering Criteria                                                                                                                                                                                                | Output                                                                                                                          |
| ---- | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Flight Fetch    | - `_IFCSClient_s eq 'Oman'` (Client)<br>- `UpdateBy_sa eq 'SSIM'` (Source)<br>- `(Status_s eq null or Status_s ne 'Cancelled')` (Active)<br>- `SyncKeyWithDest_s ne null and FlightNumber_s ne null` (Valid Data) | Raw flight data                                                                                                                 |
| 2    | Initial Cleanup | Clears existing pairing data on the fetched records to ensure a fresh pairing attempt.                                                                                                                            | Clears fields: `__id2_Flight_FirstInPairOrGroup`, `PositionInPairOrGroup_i`, `PositionInPairOrGroup_s`. Modified flights array. |
| 3    | Sorting         | Sorts all flights primarily by Flight Number (numerical value) for easier sequential comparison.                                                                                                                  | Sorted flights array.                                                                                                           |

---

### 2.2 Phase 2: Segmentation and Grouping

The flights are segmented to isolate the start points of the pairings.

| Segment              | Criteria                                                     | Purpose                                                                                                                                  |
| -------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| MCT Odd Flights      | `DestinationDepart_s = 'MCT' AND FlightNumber is Odd`        | These are the potential starting flights for a roundabout pairing. Grouped by departure date.                                            |
| Non-MCT/Even Flights | All other flights (Non-MCT departure OR Even Flight Number). | These are the potential connecting/return flights. Grouped by departure date, then flattened into a single sorted array for fast lookup. |

---

### 2.3 Phase 3: Pairing Logic Execution

The utility iterates through the MCT Odd Flights chronologically (by departure date) and attempts to build a complete chain.

| Sub-Step                    | Logic Applied                                                                                                                                                                                                                                                                                                                                                        | Outcome/Field Update                                                                                                                                                        |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. First Leg Initialization | An MCT Odd Flight is selected. Its `PositionInPairOrGroup_s` is set to `'First'`. The `__id1` of this flight becomes the primary `__id2_Flight_FirstInPairOrGroup` reference key for all subsequent legs in the chain.                                                                                                                                               | Marked as First                                                                                                                                                             |
| 2. Sequential Matching      | The function loops to find the next unpaired candidate flight from the Non-MCT/Even Flights list that satisfies all pairing rules (see Section 1.1), including:<br>- **Station Match:** Candidate departure station equals the previous arrival station.<br>- **Time Constraint:** Layover time is ≤ 1440 minutes (24 hours).<br>- **Flight Number sequence check.** | If a match is found:<br>- Candidate is marked with the First Leg's ID.<br>- Candidate's `PositionInPairOrGroup_s` is set to `'Middle'`.                                     |
| 3. Iteration and Completion | The loop continues, with the previous flight's arrival station becoming the current departure station, and the required flight number sequence is checked.                                                                                                                                                                                                           | The chain is complete when a flight is found that arrives back at MCT. Its `PositionInPairOrGroup_s` is set to `'Last'`. The return flight is removed from the search list. |

---

## 3. Data Output and Database Update

### 3.1 Output Fields

The core purpose of the utility is to populate the following fields in the `EFMTable.FLIGHT` record:

| Field                             | Value                                                                                                          | Purpose                                                                           |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `__id2_Flight_FirstInPairOrGroup` | The unique primary key (`__id1`) of the First flight in the paired sequence.                                   | Links all legs in the sequence back to the single flight that initiated the trip. |
| `PositionInPairOrGroup_s`         | Enum: `'First'`, `'Middle'`, or `'Last'`.                                                                      | Defines the role of the leg in the context of the complete pairing.               |
| `PositionInPairOrGroup_i`         | Not explicitly set in the provided code, but typically used for numerical order (1, 2, 3...) within the chain. | (Reserved/Ignored in current implementation)                                      |

### 3.2 Database Update

The paired flight records are updated in the database with the new pairing fields.

### 3.3 Unpaired Flights Log

Any flight that could not be matched according to the defined logic is logged as **Unpaired** (`unpairedFlights` array) and remains without pairing data, allowing for manual review or subsequent pairing attempts via other logic utilities.
