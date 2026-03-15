# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Project

```bash
# Install dependencies
pip install -r requirements.txt

# Run main fall detection (from repo root)
cd src && python cylinder_fall_detection.py

# Generate ArUco marker strip for a cylinder
python src/generate_cylinder_marker.py

# Generate bottom detection marker
python src/generate_bottom_marker.py
```

## Required Configuration

The project requires a `sql.env` file (gitignored) in the repo root:

```
sql_id=your_username
sql_pw=your_password
sql_ocbc=Driver={ODBC Driver 18 for SQL Server};Server=tcp:your_server.database.windows.net,1433;Database=your_database;Encrypt=yes;TrustServerCertificate=no;Connection Timeout=30;
logic_app_url="https://prod-xx.region.logic.azure.com..."
```

The ODBC driver must be installed separately (system-level). `db_logger.py` auto-selects among available ODBC drivers as a fallback.

## Architecture

```
Detection Layer (OpenCV + ArUco pose estimation)
       ↓
Logic Layer (angle smoothing → fall state machine → alert scheduler)
       ↓
Event Logging
  ├── Local file: local_fall_log.txt
  ├── Azure SQL: FallEvents table (via db_logger.py)
  └── Azure Logic Apps: async webhook POST
```

### Key Files

- **`src/cylinder_fall_detection.py`** — Main loop. Captures webcam frames, detects ArUco markers, computes 3D tilt angle via Rodrigues rotation, applies 5-frame moving average smoothing, drives a fall state machine, and schedules multi-stage alerts (2s / 1m / 10m / 1h thresholds). Also renders live status overlay and logs CSV debug data every 0.1s.
- **`src/db_logger.py`** — `DBLogger` class wrapping Azure SQL via pyodbc. Reads credentials from `sql.env`. Each run gets a unique `ExperimentID` for session tracking.
- **`src/generate_cylinder_marker.py`** — Interactive utility: given cylinder diameter and desired marker size, computes circumference and generates a printable ArUco strip PNG.
- **`src/generate_bottom_marker.py`** — Generates ArUco marker ID 99 (DICT_4X4_100) for bottom-face detection.

### Fall Detection Logic

- ArUco marker IDs 0–98 are the cylinder-side markers; ID 99 is the bottom marker.
- A fall is confirmed when tilt angle > 45° persists for ≥ 2.0 seconds **or** the bottom marker becomes visible.
- Alerts fire asynchronously (separate thread) to avoid blocking the video loop.
- The `FallEvents` table schema: `ExperimentID`, `Timestamp`, `Duration`, `MaxAngle`, `AlertLevel`.
