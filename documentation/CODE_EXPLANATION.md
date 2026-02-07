# MTA Tracker - Complete Code Explanation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Setup & Libraries (Cells 1-7)](#setup--libraries)
4. [MTATracker Class (Cell 6)](#mtatracker-class)
5. [Protobuf Parsing (Cells 11-14)](#protobuf-parsing)
6. [BetterProtobufParser Class (Cell 14)](#betterprotobufparser-class)
7. [Data Export (Cell 16)](#data-export)
8. [Debugging & Testing Cells (Cells 20-22)](#debugging--testing)

---

## Overview

This notebook fetches real-time transit data from the MTA (Metropolitan Transportation Authority) and extracts useful information from it. The entire process has three main steps:

1. **FETCH**: Get data from the MTA API (raw binary protobuf format)
2. **PARSE**: Convert binary protobuf data into readable Python dictionaries  
3. **EXPORT**: Save the parsed data to JSON, CSV, and TXT files

### What is the MTA?
The MTA operates buses and subways in New York City. They provide real-time data about:
- **Vehicles**: Current position and status of buses and trains
- **Trip Updates**: Delays and schedule changes
- **Alerts**: Service disruptions and announcements

### What is Protobuf?
Protocol Buffers (protobuf) is a compact binary format developed by Google. It's much smaller than JSON or XML, which is important for sending large amounts of data over the internet. However, it's not human-readable - we need to parse it.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    MTA API                          │
│   Returns: Binary Protobuf Data (~200KB)           │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│          BetterProtobufParser                       │
│   (Decodes binary protobuf → Python dictionaries)  │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
        ┌─────────┴──────────┬──────────┐
        ▼                    ▼          ▼
    ┌──────┐           ┌──────┐    ┌──────┐
    │ JSON │           │ CSV  │    │ TXT  │
    └──────┘           └──────┘    └──────┘
     94 KB           20 KB (spreadsheet-ready)  3 KB
                                            (metadata)
```

---

## Setup & Libraries

### Cell 5: Import Libraries
```python
import requests           # Makes HTTP requests to APIs
import json             # Works with JSON format
from datetime import datetime  # Timestamps
from typing import Optional, Dict, Any  # Type hints for clarity
import logging          # Records what the program is doing
import pandas as pd     # Data analysis (imported but not used yet)
```

**What these do:**
- `requests`: Makes the HTTP call to fetch data from the MTA API
- `json`: Converts Python objects to/from JSON format
- `datetime`: Records when we fetched data
- `logging`: Prints informational messages to track progress
- `pandas`: Optional - could be used for data analysis

---

## MTATracker Class

### What is it?
A helper class that manages fetching data from the MTA API. Think of it as a "wrapper" around the API - it handles all the communication details so we don't have to.

### Class Variables (Configuration)

```python
BASE_URL = "https://api-endpoint.mta.info/Dataservice/mtagtfsfeeds/nyct%2Fgtfs"
TIMEOUT = 30
```

- **BASE_URL**: The exact web address where MTA publishes their data
  - "nyct" = New York City Transit
  - "%2F" = URL-encoded forward slash (/)
  
- **TIMEOUT**: Wait maximum 30 seconds for the API to respond, then give up

### Method 1: `__init__()` - Initialize

```python
def __init__(self):
    self.session = requests.Session()  # Reusable HTTP connection
    self.last_update = None            # When did we last fetch?
    self.data = None                   # Stores raw API response
```

**What happens when you create `tracker = MTATracker()`:**
1. Sets up an HTTP session (connection to the internet)
2. Initializes empty variables to store data and timestamps

**Why use `requests.Session()`?**
- More efficient than creating a new connection each time
- Reuses the same TCP connection for multiple requests

### Method 2: `fetch_data()` - Get Data From API

```python
def fetch_data(self) -> Optional[bytes]:
    # Makes HTTP GET request
    response = self.session.get(
        self.BASE_URL,
        timeout=self.TIMEOUT
    )
    response.raise_for_status()  # Raise error if status is 404, 500, etc.
    
    # Store the result
    self.last_update = datetime.now()
    self.data = response.content  # Raw bytes
    
    return self.data
```

**What it does:**
1. Makes an HTTP GET request to the MTA API
2. Waits max 30 seconds for response
3. If successful, stores the data and timestamp
4. Returns the raw binary data

**Error handling:**
```python
except requests.exceptions.Timeout:
    # API took too long to respond
    
except requests.exceptions.ConnectionError:
    # Can't reach the API (network issue)
    
except requests.exceptions.HTTPError as e:
    # API returned an error (404 = not found, 500 = server error)
```

### Method 3: `parse_data()` - Process Data

```python
def parse_data(self) -> Optional[Dict[str, Any]]:
    # Placeholder - actual parsing is in BetterProtobufParser
    parsed_data = {
        "raw_bytes": len(self.data),
        "timestamp": self.last_update.isoformat()
        "status": "fetched"
    }
    return parsed_data
```

This is a simple info method. The real parsing happens in `BetterProtobufParser`.

### Method 4: `get_status()` - Check Status

```python
def get_status(self) -> Dict[str, Any]:
    return {
        "last_update": self.last_update.isoformat() if self.last_update else None,
        "data_available": self.data is not None,
        "data_size": len(self.data) if self.data else 0
    }
```

Returns current state: When did we fetch? Do we have data? How much?

---

## Protobuf Parsing

### What is Protobuf?

Protobuf (Protocol Buffers) is a binary format developed by Google. Here's why the MTA uses it:

**Advantages:**
- ✅ Very compact - data is ~200KB instead of ~1MB as JSON
- ✅ Fast to parse
- ✅ Version-compatible (can add new fields without breaking old parsers)

**Disadvantages:**
- ❌ Not human-readable (you need a schema/parser)
- ❌ Need external library or manual parser

### Protobuf Wire Format

Protobuf encodes data using "field tags" and "wire types":

```
Wire Type 0: Varint (variable-length integer)
Wire Type 1: 64-bit fixed (8 bytes)
Wire Type 2: Length-delimited (string, nested message, etc.)
Wire Type 5: 32-bit fixed (4 bytes)
```

**Example: A simple message**
```
Field 1 (route_id) = "A" (string, wire type 2)
Field 2 (vehicle_id) = 12345 (int, wire type 0)

Binary representation:
0x0a = tag for field 1, wire type 2 (length-delimited)
0x01 = length = 1 byte
0x41 = 'A'
0x10 = tag for field 2, wire type 0 (varint)
0xb9 0x60 = varint for 12345
```

### Variable-Length Integers (Varint)

Numbers in protobuf aren't always the same size. Varint encoding uses fewer bytes for small numbers:

```python
def decode_varint(data, pos):
    """
    Extract a variable-length integer from protobuf data.
    
    Algorithm:
    1. Each byte has 7 bits of data + 1 bit that says "more bytes coming"
    2. Keep reading bytes until we find one with the 0x80 bit clear
    3. Combine all the 7-bit chunks to get the final number
    
    Example:
    Byte: 0b10000001 (bit 7 is set = more bytes)
    Byte: 0b00000001 (bit 7 is clear = last byte)
    This represents: 0000001 0000001 = 129
    """
    value = 0
    shift = 0
    while pos < len(data):
        byte = data[pos]
        pos += 1
        value |= (byte & 0x7f) << shift  # Add lower 7 bits
        if (byte & 0x80) == 0:            # If high bit is 0, we're done
            break
        shift += 7  # Next byte contributes to higher positions
    return value, pos
```

---

## BetterProtobufParser Class

### Overview

This is the workhorse of the entire notebook. It takes raw binary protobuf data and converts it into Python dictionaries that we can understand.

**Architecture:**
```
parse_feed()
├─ parse_header() → {"version": "1.0", "timestamp": 1234567890}
└─ parse_entity() ← for each entity in feed
    ├─ extract_trip_update_info() → {"trip_id": "xyz", "delay_seconds": "120"}
    ├─ extract_vehicle_info()     → {"route_id": "A", "trip_id": "xyz"}
    └─ extract_alert_info()       → {"alert_message": "...", "affected_routes": "A,B"}
```

### Method: `decode_varint()` (Static)

```python
@staticmethod
def decode_varint(data, pos):
    """Read one variable-length integer from binary data"""
    value = 0
    shift = 0
    while pos < len(data):
        byte = data[pos]
        pos += 1
        value |= (byte & 0x7f) << shift
        if (byte & 0x80) == 0:
            break
        shift += 7
    return value, pos
```

**Returns:** `(the_number_we_extracted, new_position_in_data)`

### Method: `parse_feed()` (Static)

```python
@staticmethod
def parse_feed(data):
    """
    Parse the entire FeedMessage from raw protobuf bytes.
    
    Returns: {
        "header": {...},
        "entities": [
            {"id": "000001", "type": "vehicle", ...},
            {"id": "000002", "type": "alert", ...},
            ...
        ]
    }
    """
    feed = {"header": {}, "entities": []}
    pos = 0
    
    while pos < len(data):
        tag, pos = decode_varint(data, pos)
        field_num = tag >> 3      # Extract field number (bits 3-7)
        wire_type = tag & 0x07    # Extract wire type (bits 0-2)
        
        if wire_type == 2:  # Length-delimited (string/message)
            length, pos = decode_varint(data, pos)
            field_data = data[pos:pos+length]
            pos += length
            
            if field_num == 1:  # Field 1 = header
                feed["header"] = parse_header(field_data)
            elif field_num == 2:  # Field 2 = entities
                entity = parse_entity(field_data)
                feed["entities"].append(entity)
    
    return feed
```

**What's happening:**
1. Create empty feed with header and entities list
2. Loop through binary data byte by byte
3. Read tag (field number + wire type)
4. Based on field number, call appropriate parser
5. Collect all results

### Method: `parse_entity()` (Static)

```python
@staticmethod
def parse_entity(data):
    """
    Parse one FeedEntity and extract its data.
    
    Returns a dictionary like:
    {
        "id": "000001",
        "type": "vehicle",  # or "trip_update" or "alert"
        "trip_id": "106550_1..S03R",
        "route_id": "20260202",
        "delay_seconds": "N/A",
        "alert_message": "N/A",
        "affected_routes": "N/A"
    }
    """
    entity = {
        "id": "",
        "type": "unknown",
        "trip_id": "N/A",
        "route_id": "N/A",
        "delay_seconds": "N/A",
        "alert_message": "N/A",
        "affected_routes": "N/A"
    }
    
    # Parse fields within the entity
    while pos < len(data):
        tag, pos = decode_varint(data, pos)
        field_num = tag >> 3
        wire_type = tag & 0x07
        
        if wire_type == 2:
            length, pos = decode_varint(data, pos)
            field_data = data[pos:pos+length]
            pos += length
            
            if field_num == 1:  # id field
                entity["id"] = field_data.decode('utf-8')
                
            elif field_num == 2:  # trip_update field
                entity["type"] = "trip_update"
                extract_trip_update_info(field_data, entity)
                
            elif field_num == 3:  # vehicle field
                entity["type"] = "vehicle"
                extract_vehicle_info(field_data, entity)
                
            elif field_num == 4:  # alert field
                entity["type"] = "alert"
                extract_alert_info(field_data, entity)
    
    return entity
```

**Key insight:** Each entity can only have ONE of {trip_update, vehicle, alert}. The field_num tells us which type.

### Method: `extract_vehicle_info()` (Static)

```python
@staticmethod
def extract_vehicle_info(data, entity):
    """
    Extract vehicle-specific data (route, trip, position).
    
    For each vehicle in the feed:
    Field 1 = Trip descriptor (nested message)
    Field 2 = Position (GPS coordinates - we skip this)
    Field 3+ = Other metadata (we skip)
    """
    # Look for field 1 (trip descriptor)
    if field_num == 1:
        extract_trip_info(field_data, entity)  # Get route_id and trip_id
```

**What data does a vehicle have?**
- ✅ route_id: Which line is this bus on? (e.g., "A", "1", "M15")
- ✅ trip_id: Which specific trip? (unique identifier)
- ❌ delay_seconds: Not available for vehicles (only for trip_updates)

### Method: `extract_trip_info()` (Static)

```python
@staticmethod
def extract_trip_info(data, entity):
    """
    Extract trip_id and route_id from a TripDescriptor message.
    
    TripDescriptor contains:
    Field 1 = trip_id
    Field 3 = route_id (note: skips field 2)
    """
    while pos < len(data):
        tag, pos = decode_varint(data, pos)
        field_num = tag >> 3
        
        if field_num == 1:
            value = decode_string(data, pos, length)
            entity["trip_id"] = value
            
        elif field_num == 3:
            value = decode_string(data, pos, length)
            entity["route_id"] = value
```

**Example output for a vehicle:**
```python
{
    "id": "000001",
    "type": "vehicle",
    "trip_id": "106550_1..S03R",
    "route_id": "20260202",
    "delay_seconds": "N/A",
    "alert_message": "N/A",
    "affected_routes": "N/A"
}
```

### Method: `extract_alert_info()` (Static)

```python
@staticmethod
def extract_alert_info(data, entity):
    """
    Extract alert-specific data.
    
    IMPORTANT: MTA alerts don't have text descriptions!
    Field 7 actually contains the affected route identifier.
    
    Alert structure:
    Field 1 = active_period (time window - we skip)
    Field 2 = informed_entity (which routes affected - we skip)
    Field 3-6 = Various metadata (we skip)
    Field 7 = Route identifier (e.g., "142S", "103N")
    """
    route_identifier = "N/A"
    
    while pos < len(data):
        tag, pos = decode_varint(data, pos)
        field_num = tag >> 3
        
        if field_num == 7:
            route_identifier = decode_string(data, pos, length)
    
    # Populate entity
    if route_identifier != "N/A":
        entity["affected_routes"] = route_identifier
        entity["alert_message"] = "Service Alert"
```

**Key finding:**
- ❌ MTA doesn't provide alert message text
- ✅ Field 7 contains route ID (e.g., "142S", "103N")
- ✅ We use generic "Service Alert" for all alerts

**Example output for an alert:**
```python
{
    "id": "000003",
    "type": "alert",
    "trip_id": "N/A",
    "route_id": "N/A",
    "delay_seconds": "N/A",
    "alert_message": "Service Alert",
    "affected_routes": "142S"
}
```

---

## Data Export

### Cell 16: Export to Logs

```python
# Create logs directory structure
logs_dir = Path("logs")
logs_dir.mkdir(exist_ok=True)

timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")  # e.g., 20260202_191931
run_dir = logs_dir / timestamp  # e.g., logs/20260202_191931/
run_dir.mkdir(exist_ok=True)
```

**Why this structure?**
- `logs/` = Main directory holding all historical runs
- `logs/20260202_191931/` = Each run in its own timestamped folder
- This way you can compare data from different times

### File 1: JSON Export

```python
json_file = run_dir / f"mta_feed_{timestamp}_fixed.json"
with open(json_file, 'w') as f:
    json.dump(feed, f, indent=2, default=str)
```

**Output format:**
```json
{
  "header": {
    "version": "1.0",
    "timestamp": 1706900891
  },
  "entities": [
    {
      "id": "000001",
      "type": "vehicle",
      "trip_id": "106550_1..S03R",
      "route_id": "20260202",
      ...
    },
    ...
  ]
}
```

**Use case:** Programmatic access to all data

### File 2: CSV Export

```python
csv_file = run_dir / f"mta_entities_{timestamp}_fixed.csv"
with open(csv_file, 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(["Entity_ID", "Type", "Route_ID", "Trip_ID", "Delay_Seconds", "Alert_Message", "Affected_Routes"])
    
    for entity in feed["entities"]:
        entity_id = entity.get("id", "N/A")[:50]
        ent_type = entity.get("type", "unknown")
        route = entity.get("route_id", "N/A")
        trip = entity.get("trip_id", "N/A")
        delay = entity.get("delay_seconds", "N/A")
        alert_msg = entity.get("alert_message", "N/A")
        affected_routes = entity.get("affected_routes", "N/A")
        
        writer.writerow([entity_id, ent_type, route, trip, delay, alert_msg, affected_routes])
```

**Before (broken):**
```
000009,alert,N/A,N/A,N/A,137S,N/A
       ↑ Route data in wrong column!
```

**After (fixed):**
```
000009,alert,N/A,N/A,N/A,Service Alert,137S
                                        ↑ Route data in correct column
```

**Output format:**
```
Entity_ID,Type,Route_ID,Trip_ID,Delay_Seconds,Alert_Message,Affected_Routes
000001,vehicle,20260202,106550_1..S03R,N/A,N/A,N/A
000002,vehicle,20260202,108950_1..S03R,N/A,N/A,N/A
000003,alert,N/A,N/A,N/A,Service Alert,142S
000004,vehicle,20260202,109150_1..N03R,N/A,N/A,N/A
000005,alert,N/A,N/A,N/A,Service Alert,103N
```

**Use case:** Open in Excel/Google Sheets, filter/sort data

### File 3: Metadata/Legend

```python
meta_file = run_dir / f"mta_metadata_{timestamp}_fixed.txt"
with open(meta_file, 'w') as f:
    f.write("=" * 70 + "\n")
    f.write("MTA GTFS-REALTIME DATA EXPORT (IMPROVED PARSER)\n")
    f.write("=" * 70 + "\n\n")
    
    # Write statistics, field definitions, usage instructions
```

**Contents:**
- When was this data exported?
- How many entities? (vehicles, trip_updates, alerts)
- What percentage have route data extracted?
- Field definitions and explanations
- How to use the CSV file

---

## Debugging & Testing Cells

### Cell 20: Debug Alert Structure

```python
# Find first alert and display its fields
first_alert = None
for entity in feed["entities"]:
    if entity["type"] == "alert":
        first_alert = entity
        break

print(f"First Alert Entity:")
print(f"  ID: {first_alert['id']}")
print(f"  Type: {first_alert['type']}")
print(f"  Alert Message: {first_alert['alert_message']}")
print(f"  Affected Routes: {first_alert['affected_routes']}")
```

**Purpose:** Verify that alert data is being extracted correctly

### Cell 21: Deep Dive into Protobuf

```python
# This cell manually reconstructs protobuf parsing to show
# exactly what fields exist in the raw binary data

# For first alert, it shows:
# Field 1: [binary data] (internal schedule info)
# Field 3: 38 (enum value)
# Field 5: 1770077231 (timestamp)
# Field 7: '142S' (route identifier)
```

**Purpose:** Understand the actual protobuf structure in MTA data

### Cell 22: Check Multiple Alerts

```python
# Display first 3 alerts to show the pattern
# All show:
# - Field 7 contains route IDs: 142S, 103N, 139S
# - No fields contain readable alert message text
```

**Purpose:** Confirm the pattern holds across all alerts

---

## Important Data Findings

### Before (Broken)
```
000009,alert,N/A,N/A,N/A,137S,N/A
                                ↑
                    Route ID in wrong column!
```

### After (Fixed)
```
000009,alert,N/A,N/A,N/A,Service Alert,137S
                             ↑         ↑
                    Generic message  Correct column
```

### Statistics
- Total entities: 462
- Vehicles: 286 (100% have route_id)
- Trip updates: 0 (none available today)
- Alerts: 175 (100% have affected_routes)

---

## How to Use This Code

### Step 1: Setup Environment
```bash
python3 -m venv mta_env
source mta_env/bin/activate
pip install requests protobuf pandas
```

### Step 2: Run the Notebook
```bash
jupyter notebook MTA_Tracker.ipynb
```

### Step 3: Execute Cells in Order
1. Cells 1-7: Setup and imports
2. Cell 8: Initialize MTATracker
3. Cell 9: Fetch data from API
4. Cell 10: Check status
5. Cell 14: Load BetterProtobufParser
6. Cell 15: Test parser on data
7. Cell 16: Export to CSV/JSON/TXT
8. Cells 20-22: Debug and verify

### Step 4: Analyze Results
```bash
# View latest export
ls -la logs/*/

# Open CSV in spreadsheet app
open logs/20260202_191931/mta_entities_20260202_191931_fixed.csv
```

---

## Key Concepts Summary

| Concept | Meaning | Example |
|---------|---------|---------|
| **Protobuf** | Compact binary format | 200KB vs 1MB as JSON |
| **Field Tag** | Identifier for data field | Field 1 = ID, Field 2 = trip_update |
| **Wire Type** | How data is encoded | 0=varint, 2=length-delimited |
| **Varint** | Variable-length integer | Small numbers = fewer bytes |
| **Entity** | One item in the feed | One vehicle, or one alert |
| **Entity Type** | Category of entity | vehicle, trip_update, or alert |
| **Route ID** | Which transit line | "1", "A", "M15" |
| **Trip ID** | Unique trip identifier | "106550_1..S03R" |

---

## Troubleshooting

### Q: All fields showing N/A
**Cause:** Parser not extracting nested message data
**Fix:** Use `BetterProtobufParser` (Cell 14)

### Q: Alerts have route IDs in wrong column
**Cause:** Was putting field 7 (route ID) in alert_message column
**Fix:** Now correctly maps field 7 to affected_routes

### Q: Can't connect to MTA API
**Cause:** Network issue or API down
**Fix:** Check internet, try again later

### Q: CSV file is empty
**Cause:** No entities parsed
**Fix:** Verify parse_feed() completed successfully

---

## Future Improvements

1. **Database Storage**: Store data in SQLite instead of CSV
2. **Real-time Updates**: Fetch data every 30 seconds automatically
3. **Web Dashboard**: Visualize alerts and vehicle positions
4. **Delay Analysis**: Calculate average delays by route
5. **Email Alerts**: Notify when specific routes have alerts
6. **GPS Mapping**: Plot vehicle positions on a map

