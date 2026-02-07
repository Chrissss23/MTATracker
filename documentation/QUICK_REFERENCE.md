# MTA Tracker - Quick Reference Guide

## Quick Start

```bash
# 1. Setup
python3 -m venv mta_env
source mta_env/bin/activate <-- Run this one to start 
pip install requests protobuf pandas

# 2. Run notebook
jupyter notebook MTA_Tracker.ipynb

# 3. Execute cells in order (1 through 25)
# 4. Check results
ls -la logs/*/
```

---

## Cell Execution Order

| Cell | Name | What It Does |
|------|------|------------|
| 1-4 | Setup | Markdown explanations |
| 5 | Imports | Load libraries (requests, json, logging, pandas) |
| 6 | MTATracker Class | Define class to fetch data from API |
| 7 | Initialize | `tracker = MTATracker()` |
| 8 | Fetch | `tracker.fetch_data()` - Gets ~200KB protobuf bytes |
| 9 | Status | `tracker.get_status()` - Show what we have |
| 10 | Install Packages | Ensures protobuf is available |
| 11 | First Parser | Simple protobuf parser (basic) |
| 12 | Protobuf Examples | Shows how varint encoding works |
| 13 | Another Parser | Intermediate version |
| 14 | **BetterProtobufParser** | **Main parser - use this one!** |
| 15 | Test Parser | Run it on data and show first 10 entities |
| 16 | **Export Data** | **Save to JSON/CSV/TXT in logs/** |
| 17 | Markdown Explanation | Understanding the code (NEW!) |
| 18-19 | Misc | Old code (can skip) |
| 20 | Debug Alerts 1 | Show first alert data |
| 21 | Debug Alerts 2 | Deep dive into protobuf structure |
| 22 | Debug Alerts 3 | Show pattern across multiple alerts |
| 23-24 | Misc | Utilities |
| 25 | GTFS-RT Class | Complete decoder class |

---

## Classes & Methods

### MTATracker Class

```python
tracker = MTATracker()

# Method: fetch data from API
tracker.fetch_data() → bytes (raw protobuf data)

# Method: get info about current state
tracker.get_status() → dict with last_update, data_available, data_size

# Method: placeholder (not used)
tracker.parse_data() → dict with basic info

# Attribute: the actual raw data
tracker.data → bytes (raw API response)
```

### BetterProtobufParser Class

```python
# Static methods (don't need to create instance)

# Main entry point - parse entire feed
BetterProtobufParser.parse_feed(raw_bytes) → feed_dict

# Parse individual pieces
BetterProtobufParser.parse_header(bytes) → header_dict
BetterProtobufParser.parse_entity(bytes) → entity_dict

# Extract specific data from nested messages
BetterProtobufParser.extract_vehicle_info(bytes, entity)
BetterProtobufParser.extract_trip_info(bytes, entity)
BetterProtobufParser.extract_alert_info(bytes, entity)
BetterProtobufParser.extract_trip_update_info(bytes, entity)

# Low-level utility
BetterProtobufParser.decode_varint(bytes, position) → (value, new_position)
```

---

## Data Structures

### Feed Dictionary (returned by parse_feed)

```python
feed = {
    "header": {
        "version": "1.0",
        "timestamp": 1706900891,
        "incrementality": 0
    },
    "entities": [
        {
            "id": "000001",
            "type": "vehicle",
            "trip_id": "106550_1..S03R",
            "route_id": "20260202",
            "delay_seconds": "N/A",
            "alert_message": "N/A",
            "affected_routes": "N/A"
        },
        ...
    ]
}
```

### Entity Dictionary (one item in entities list)

```python
entity = {
    "id": "000001",                    # Unique identifier
    "type": "vehicle",                 # vehicle | trip_update | alert
    "trip_id": "106550_1..S03R",       # Specific trip identifier
    "route_id": "20260202",            # Transit line
    "delay_seconds": "N/A",            # For trip_updates only
    "alert_message": "Service Alert",  # For alerts only
    "affected_routes": "142S"          # For alerts only
}
```

---

## Data Values Explained

### Entity Types

```python
# VEHICLE - Current location & status of a bus/train
{
    "type": "vehicle",
    "route_id": "1",      # ← Has this
    "trip_id": "xyz",     # ← Has this
    "delay_seconds": "N/A"  # ← Doesn't have this
}

# TRIP_UPDATE - Delays and schedule changes
{
    "type": "trip_update",
    "route_id": "1",      # ← Has this
    "trip_id": "xyz",     # ← Has this
    "delay_seconds": "120"  # ← Has this (in seconds)
}

# ALERT - Service advisories
{
    "type": "alert",
    "alert_message": "Service Alert",  # ← Always "Service Alert"
    "affected_routes": "142S"           # ← Route ID
    # Everything else is N/A
}
```

### Route IDs

```python
# Route IDs come in different formats:

# Subway lines (numbered)
"1", "2", "3", "4", "5", "6", "7"

# Subway lines (lettered)  
"A", "B", "C", "D", "E", "F", "G", "L", "M", "N", "Q", "R", "W", "Z"

# Bus routes
"101", "103", "104", "M15", "Q32", "BX22", "SIM5"

# In MTA's raw data, might be encoded as:
"20260202"  # (confusing internal format)
"142S"      # (clear route identifier)
```

### Why Data is "N/A"

```python
entity = {
    "id": "000001",
    "type": "vehicle",
    "trip_id": "106550_1..S03R",    # ← Has real data
    "route_id": "20260202",          # ← Has real data
    "delay_seconds": "N/A",          # ← N/A because vehicles don't have delays
                                     #    (only trip_updates have delays)
    "alert_message": "N/A",          # ← N/A because this is a vehicle, not an alert
    "affected_routes": "N/A"         # ← N/A because this is a vehicle, not an alert
}
```

---

## Output Files Explained

### File 1: JSON (mta_feed_[timestamp]_fixed.json)

**Format:** Complete structured data

```json
{
  "header": {
    "version": "1.0",
    "timestamp": 1706900891
  },
  "entities": [
    { "id": "000001", "type": "vehicle", ... },
    { "id": "000002", "type": "vehicle", ... },
    ...
  ]
}
```

**When to use:**
- Programmatic access
- Feeding data to other tools
- Archival/backup

**Size:** ~94 KB (full data)

### File 2: CSV (mta_entities_[timestamp]_fixed.csv)

**Format:** Spreadsheet-friendly

```
Entity_ID,Type,Route_ID,Trip_ID,Delay_Seconds,Alert_Message,Affected_Routes
000001,vehicle,20260202,106550_1..S03R,N/A,N/A,N/A
000002,vehicle,20260202,108950_1..S03R,N/A,N/A,N/A
000003,alert,N/A,N/A,N/A,Service Alert,142S
```

**When to use:**
- Open in Excel/Google Sheets
- Quick analysis
- Sharing data with non-technical people
- Filtering and sorting

**Size:** ~20 KB (easy to handle)

### File 3: TXT (mta_metadata_[timestamp]_fixed.txt)

**Format:** Human-readable documentation

```
======================================================================
MTA GTFS-REALTIME DATA EXPORT (IMPROVED PARSER)
======================================================================

EXPORT TIMESTAMP: 2026-02-02T19:19:31.123456

FEED HEADER INFORMATION:
----------------------------------------------------------------------
GTFS Version: 1.0
Feed Timestamp: 2026-02-02 19:19:31

ENTITY BREAKDOWN:
----------------------------------------------------------------------
Total Entities: 462
  - Trip Updates: 0
  - Vehicles: 286
  - Alerts: 175

DATA EXTRACTION STATISTICS:
----------------------------------------------------------------------
Entities with route_id: 286 (61.9%)
Entities with trip_id: 286 (61.9%)
Entities with delay: 0 (0.0%)

... (more documentation)
```

**When to use:**
- Understanding what the data contains
- Field definitions and explanations
- Usage instructions

**Size:** ~3 KB (metadata only)

---

## Protobuf Concepts (Simplified)

### What is Protobuf?

Binary format developed by Google. Much more compact than JSON.

```
JSON version: 1,000 bytes
Protobuf version: 200 bytes (same data!)

That's why MTA uses it - saves bandwidth.
```

### Wire Types

```
Wire Type 0: Varint (variable-length integer)
           - Numbers that fit in different sizes
           - e.g., 1 uses 1 byte, 1,000,000 uses 4 bytes

Wire Type 1: 64-bit fixed
           - Always 8 bytes

Wire Type 2: Length-delimited
           - Preceded by length
           - Strings, nested messages
           - e.g., [length: 5][data: hello]

Wire Type 5: 32-bit fixed
           - Always 4 bytes
```

### Tag Format

```
Each field starts with a tag (1-3 bytes):

Tag = (field_number << 3) | wire_type

Example:
Field 1, Wire Type 2 = (1 << 3) | 2 = 10 = 0x0a

To extract:
field_number = tag >> 3
wire_type = tag & 0x07
```

### Varint Example

```
Decimal 150:

Binary: 10010110

Split into 7-bit chunks:
00000001 0010110

Encode with continuation bits:
10010110 (continued, lower 7 bits = 0010110)
00000001 (final, all 7 bits = 0000001)

So: 150 = 0xb6 0x01 (two bytes)
```

---

## Troubleshooting

### Problem: "Can't connect to MTA API"

```python
# Error message:
requests.exceptions.ConnectionError: Failed to connect to MTA API

# Solutions:
1. Check internet connection
   ping google.com
   
2. Check if MTA API is up
   curl https://api-endpoint.mta.info/...
   
3. Wait a few minutes and try again
   (API might be temporarily down)
```

### Problem: "All fields show N/A"

```python
# CSV output:
000001,vehicle,N/A,N/A,N/A,N/A,N/A
000002,vehicle,N/A,N/A,N/A,N/A,N/A

# Likely cause:
Parser didn't extract nested message data

# Solution:
Make sure you're using BetterProtobufParser (Cell 14)
Old parser didn't extract nested messages
```

### Problem: "Invalid protobuf data"

```python
# Error:
Exception in parsing

# Likely causes:
1. Data is corrupted
2. Parser has a bug
3. MTA changed their format

# Solutions:
1. Check logs for error messages
2. Re-run cell to fetch fresh data
3. Try in fresh Python kernel
```

### Problem: "CSV file is empty"

```python
# CSV file exists but has only headers, no data rows

# Likely cause:
Feed has no entities

# Check:
print(len(feed['entities']))  # Should be > 0

# If 0:
Check that parse_feed() completed successfully
Make sure you're using the right 'feed' variable
```

---

## Statistics (From Latest Run)

```
Execution Date: 2026-02-02 19:19:31
API Response Size: 208,264 bytes

Entity Breakdown:
├─ Vehicles: 286 (61.9%)
│  └─ Route ID Extraction: 100%
│  └─ Trip ID Extraction: 100%
├─ Alerts: 175 (37.9%)
│  └─ Affected Routes Extraction: 100%
└─ Trip Updates: 0 (0.0%)

Total Unique Routes: Multiple (A, 1, 2, 142S, 103N, etc.)

Export Locations:
├─ logs/20260202_191931/mta_feed_20260202_191931_fixed.json (94 KB)
├─ logs/20260202_191931/mta_entities_20260202_191931_fixed.csv (20 KB)
└─ logs/20260202_191931/mta_metadata_20260202_191931_fixed.txt (3 KB)
```

---

## Next Steps / Future Features

1. **Automated Scheduling**
   ```python
   # Fetch data every 30 seconds
   import schedule
   schedule.every(30).seconds.do(tracker.fetch_data)
   ```

2. **Database Storage**
   ```python
   # Store in SQLite instead of CSV
   import sqlite3
   db.execute("INSERT INTO entities VALUES (...)")
   ```

3. **Web Dashboard**
   ```python
   # Visualize alerts
   # Show vehicle positions on map
   ```

4. **Email Alerts**
   ```python
   # Notify when specific routes have alerts
   if entity['affected_routes'] == 'A':
       send_email("A train has an alert!")
   ```

5. **Delay Analysis**
   ```python
   # Calculate average delays by route
   average_delay = sum(delays) / len(delays)
   ```

---

## Common Questions

**Q: Why do some entities have route_id but vehicles don't?**
A: Different entity types contain different information:
- Vehicles: Always have route + trip
- Trip Updates: Have route + trip + delay
- Alerts: Only have route (in affected_routes field)

**Q: Why do all alerts show "Service Alert"?**
A: The MTA protobuf feed doesn't include alert message text. Field 7 contains the route identifier instead.

**Q: Can I run this without an internet connection?**
A: No - it fetches data from the MTA API. You need internet.

**Q: How often does the data update?**
A: The MTA API updates in real-time. New data is available every few seconds.

**Q: Can I parse data from a different transit agency?**
A: Possibly - if they also use GTFS-realtime protobuf format. You'd need to adjust field numbers.

**Q: What does "N/A" mean?**
A: "Not Available" - that field doesn't have data for this entity type.

