# ptici.mk x EBP Data Aggregation Logic (Bulk Mode)

This document outlines the validation and aggregation logic used to generate the **European Bird Portal (EBP)** export in **Bulk Mode** (`mode="B"`), specifically for `location_mode="A"` (Aggregated).

## 1. Aggregation Strategy
All bird observations (Casual, Transect, and Direct entries) sharing the same **10x10km Grid Cell** (`LAEA10x10`) and **Date** are grouped into a single **Event**.

- **Location Mode**: `"A"` (Aggregated)
- **Data Type**: `"C"` (Casual/Aggregated)
- **Protocol**: Omitted (Data from multiple sources/protocols is merged)

## 2. Event-Level Aggregation Rules
Each aggregated event defines the context for the observations.

| **Key**         | **Value / Logic**                           | **EBP Compliance Note**                                          |
|:----------------|:--------------------------------------------|:-----------------------------------------------------------------|
| `event_id`      | `{LAEA10x10}_{UnixTimestamp}`               | Unique identifier for the location+date combo.                   |
| `location`      | 10x10km Grid Code (e.g., `10kmE353N212`)    | Standard ETRS89-LAEA grid code.                                  |
| `date`          | `YYYY-MM-DD`                                | Date of observations.                                            |
| `time`          | `""` (Empty String)                         | **Removed** for `location_mode="A"` (covers whole day).          |
| `observer`      | **Count** of unique User IDs (as string)    | "Number of different observers" (e.g., `"3"`).                   |
| `records`       | **Count** of unique Species + Observer pairs| "Number of different combinations of observer and species".       |
| `protocol_id`   | `""` (Empty String)                         | Blank as data is aggregated from multiple protocols.              |
| `data_type`     | `"C"`                                       | Aggregated data is classified as Casual.                         |

## 3. Record-Level Aggregation Rules
Within each event, data is further grouped by **Species (HBW Code)**.

| **Key**              | **Value / Logic**                       | **EBP Compliance Note**                                      |
|:---------------------|:----------------------------------------|:-------------------------------------------------------------|
| `record_id`          | `{event_id}_{species_code}`             | Unique record identifier.                                    |
| `species_code`       | HBW Species Code (Integer)              | Mapped from internal species list. Entries without HBW codes are skipped. |
| `count`              | **Maximum** count recorded              | "Maximum count of all records with counts".                  |
| `records_of_species` | **Count** of unique observers           | "Number of different observers that have recorded the species". |
| `breeding_code`      | **Maximum** breeding code index         | Highest EBBA2 code (0-16) recorded for this species in the event. |
| `flying_over`        | `"Y"`, `"N"`, or `""`                   | `"Y"` if all flying, `"N"` if none flying. **Empty** if mixed behaviors (unclear) or unknown. |

## 4. Aggregation Example

**Input Data (Same Day, Same Grid `10kmE100N100`)**:
1. **User A**: Sees 2 Robins (Breeding: 0)
2. **User B**: Sees 1 Robin (Breeding: 2)
3. **User A**: Sees 5 Blackbirds

**Aggregated Output**:

**Event**:
- `location`: `10kmE100N100`
- `observer`: `"2"` (User A, User B)
- `records`: `2` (Robin+UserA + Robin+UserB + Blackbird+UserA -> Wait! Robin+UserA is 1, Robin+UserB is 1, Blackbird+UserA is 1. Total = 3 combinations)
    - *Correction*: The logic is unique (Species, Observer) tuples.
        - (Robin, User A)
        - (Robin, User B)
        - (Blackbird, User A)
    - Total `records` = 3.

**Records**:
1. **Robin (`species_code`: 1234)**
    - `count`: `2` (Max of 2 and 1)
    - `records_of_species`: `2` (User A and User B)
    - `breeding_code`: `2` (Max of 0 and 2)
2. **Blackbird (`species_code`: 5678)**
    - `count`: `5`
    - `records_of_species`: `1` (User A)
