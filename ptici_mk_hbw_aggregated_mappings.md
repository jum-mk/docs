# ptici.mk x EBP Data Export Logic (Bulk Mode)

This document outlines the logic in ptici.mk api used to generate the **European Bird Portal (EBP)** exports in **Bulk Mode** (`mode="B"`). The system is split into two distinct export streams/endpoints.

## 1. Aggregated Export (Casual & Direct)
**Endpoint**: `/api/export_partner_json/`
**Target Data**: Casual Observations and Direct Entries.
**Location Mode**: `"A"` (Aggregated 10x10km Grid)

### Aggregation Strategy
All Casual and Direct entries sharing the same **10x10km Grid Cell** (`LAEA10x10`) and **Date** are grouped into a single **Event**.

### Event-Level Logic
| **Key** | **Value / Logic** | **Notes** |
| :--- | :--- | :--- |
| `event_id` | `{LAEA10x10}_{UnixTimestamp}` | Unique identifier for location+date combo. |
| `location` | 10x10km Grid Code | e.g., `10kmE353N212`. |
| `date` | `YYYY-MM-DD` | Date of observations. |
| `time` | `""` (Empty String) | Omitted for aggregated data. |
| `location_mode` | `"A"` | Aggregated. |
| `data_type` | `"C"` | Casual/Aggregated. |
| `observer` | **Count** of unique User IDs | Number of distinct observers active in that grid/date. |
| `records` | **Count** of unique (Species + Observer) combinations | Total distinct observation instances. |
| `protocol_id` | `""` | Empty. |

### Record-Level Logic
Records are aggregated by **Species (HBW Code)** within the Event. Note that `records_of_species` and `count` aggregation differs from standard individual records.

| **Key** | **Value / Logic** | **Notes** |
| :--- | :--- | :--- |
| `record_id` | `{event_id}_{species_code}` | Unique record ID. |
| `species_code` | HBW Species Code (Integer) | Mapped from `codename_hbw`. |
| `count` | **Maximum** count recorded | Max count observed by any single observer for this species. |
| `records_of_species` | **Count** of unique observers | Number of observers who saw this species. |
| `breeding_code` | **Maximum** breeding code index | Highest EBBA2 code (0-16) recorded. |
| `flying_over` | `"Y"`, `"N"`, or `""` | `"Y"` if ALL records are overflight, `"N"` if NONE, `""` if mixed. |

---

## 2. Transect Export (Standard/Complete Lists)
**Endpoint**: `/api/export_transect_json/`
**Target Data**: Transect Sessions.
**Location Mode**: `"E"` (Exact/Event)

### Strategy
Detailed export where **1 Session = 1 Event**. No aggregation across different user sessions.

### Event-Level Logic
| **Key** | **Value / Logic** | **Notes** |
| :--- | :--- | :--- |
| `event_id` | `{Session_ID}` | Direct Database ID of the Session. |
| `location` | WKT Geometry | Point or Path (WKT format). Defaults to `POINT(0 0)` if missing. |
| `date` | `YYYY-MM-DD` | Session Date. |
| `time` | `HH:MM:SS` | Session Start Time. |
| `duration` | Hours (Float) | Calculated as `End_Time - Start_Time`. |
| `location_mode` | `"E"` | Exact/Event based. |
| `data_type` | `"L"` (Complete) or `"F"` (Fixed) | Based on `is_list_complete` flag. |
| `observer` | User ID | Specific observer ID. |
| `records` | **Count** of unique species | Number of species in the list. |
| `protocol_id` | `TRANSECT_STD` | Default protocol code (can be overridden via params). |

### Record-Level Logic
Returns individual records for the session.

| **Key** | **Value / Logic** | **Notes** |
| :--- | :--- | :--- |
| `record_id` | `{Entry_ID}` | Direct Database ID of the Entry. |
| `count` | `minimum_number` | Count observed. |
| `records_of_species` | `1` | Strictly 1 for single-list exports. |
| `flying_over` | `"Y"` or `"N"` | Direct mapping from boolean flag. |
| `breeding_code` | Numeric Code (0-16) | EBBA2 standard. |
