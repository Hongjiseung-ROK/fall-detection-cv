# ArUco Fall Detection

A computer vision system that detects fall events for cylindrical objects using ArUco markers, with Azure SQL logging and Logic App alerts.

## Demo

<video src="demo.mp4" controls width="720"></video>

> Real-time fall detection pipeline: webcam → ArUco pose estimation → fall confirmation (2s threshold) → alert dispatch.

## Key Features

*   **Cylindrical Object Tracking:** Uses a strip of ArUco markers to detect tilt from any angle.
*   **Robust Fall Detection Logic:**
    *   **Angle Smoothing:** Uses a moving average filter to reduce sensor noise.
    *   **Strict 2-Second Verification:** Requires a continuous "fallen" state for 2.0 seconds for *both* high-tilt angles and bottom marker detection to prevent false alarms.
*   **Dual Logging System:**
    *   **Azure SQL Database:** Stores structured event data (`FallEvents` table) with `ExperimentID` and `VerificationStatus`.
    *   **Local Fallback Log:** Writes to `local_fall_log.txt` for offline debugging.
*   **Real-time Alerts with Smart Intervals:**
    *   **Stage-Based Notifications:** Sends alerts at **2s, 1m, 10m, and 1h** of continuous fall duration. Alerts stop after 1 hour for the same event to prevent spam.
    *   **Clean Data:** Risk Angle is rounded to the nearest integer for consistent reporting.
    *   **Logic App Webhook:** Asynchronously sends HTTP POST requests to Azure Logic Apps.

## Responsible AI Verification
    *   **Database Schema:** Includes `VerificationStatus` (Pending/Verified/FalsePositive) and `VerifySubject` columns.
    *   **Experiment Tracking:** Generates unique `ExperimentID` per run for session management.
*   **Performance Monitoring:** Real-time RAM usage display.

## System Architecture

1.  **Detection Layer:** Python + OpenCV (ArUco).
2.  **Logic Layer:** Angle calculation, Smoothing, Thresholding (45 deg).
3.  **Data Layer:** Azure SQL Database (via ODBC).
4.  **Notification Layer:** Azure Logic Apps (HTTP Webhook).

## Installation

1.  **Clone the repository:**
    ```bash
    git clone <repository_url>
    cd marker_dev
    ```

2.  **Install Dependencies:**
    ```bash
    pip install -r requirements.txt
    ```

3.  **Environment Setup:**
    *   Ensure an Azure SQL Database is set up.
    *   Create a `sql.env` file in the root directory (not committed to git) with the following content:
        ```
        sql_id=your_username
        sql_pw=your_password
        sql_ocbc=Driver={ODBC Driver 18 for SQL Server};Server=tcp:your_server.database.windows.net,1433;Database=your_database;Encrypt=yes;TrustServerCertificate=no;Connection Timeout=30;
        logic_app_url="https://prod-xx.region.logic.azure.com..."
        ```

## Usage

Run the main detection script:

```bash
python src/cylinder_fall_detection.py
```

### Operational Workflow
1.  **Startup:** The system generates a unique **Experiment ID**.
2.  **Monitoring:** The webcam feed monitors the object.
3.  **Fall Event:**
    *   If tilt > 45 degrees OR Bottom Marker (ID 99) is seen:
        *   Timer starts.
    *   If condition holds for **2.0 seconds**:
        *   **Smart Alert Start:** First notification sent (DB + Webhook).
        *   **Follow-up Alerts:** If fall persists, alerts repeat at 1m, 10m, and 1h intervals.
        *   **Log to DB:** Status `FALL_CONFIRMED`, VerificationStatus `0` (Pending), RiskAngle (Integer).
        *   **Webhook:** Sends JSON payload to Logic App (Timeout 30s, Async).

## Project Structure

*   `src/cylinder_fall_detection.py`: Main application entry point.
*   `src/db_logger.py`: Database interaction class.
*   `requirements.txt`: Python package dependencies.
*   `SQL_AGENT_TOOLS.md`: Documentation for SQL Agent verification tools.
