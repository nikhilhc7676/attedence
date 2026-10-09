# Low Attendance Alert System

A beginner-friendly college attendance dashboard built with React, Flask, Pandas, and SQLite. It recalculates every attendance percentage on import and on record changes; incoming `Atn_%` and `Alert_Status` values are never trusted.

> **Dataset note:** No attendance CSV was present in the supplied workspace. To make the project immediately runnable, `data/attendance.csv` contains a small, clearly replaceable demonstration dataset. Replace it with the college-provided file (same column names) before presenting real attendance. The seeded SQLite data is imported from that CSV, not hardcoded dashboard data.

## Features

- Responsive dashboard with API-backed student counts, safe/alert totals, average, and Recharts visualizations.
- Search, subject/status filters, attendance sorting, pagination, student details, add/edit/delete.
- Dedicated low-attendance alerts, with **Critical** (<50%) and **Warning** (50%–74.99%) severity.
- CSV import with validation/cleaning summary; importing replaces the current attendance dataset.
- SQLite persistence and indexes for student ID, subject, attendance, and status.
- REST API validation and JSON errors; automatic calculation of the 75% threshold.

## Project architecture

```text
low-attendance-alert-system/
├── backend/
│   ├── app.py                 Flask application, schema bootstrap, one-time CSV seed
│   ├── database.py            SQLite connection and schema/index creation
│   ├── models.py              Attendance calculations and row helpers
│   ├── routes.py              REST API and validation
│   ├── data_cleaning.py       Reusable Pandas import/cleaning pipeline
│   ├── requirements.txt
│   └── cleaned_attendance.csv Generated normalized CSV after seed/import
├── data/
│   └── attendance.csv         Initial dataset (replace with provided CSV)
├── frontend/
│   ├── src/App.jsx            Dashboard, student table, alert and analytics views
│   ├── src/services/api.js    Axios REST client
│   ├── src/styles.css         Responsive application styling
│   └── package.json
└── attendance.db              Created automatically at first backend start
```

SQLite is accessed with Python's built-in `sqlite3` module, so no ORM setup is required. The single attendance table stores one record per student ID. Attendance is calculated as `round(attended_classes / total_classes * 100, 2)`; values below 75 are `Alert`, otherwise `Safe`.

## Requirements

- Python 3.10 or newer
- Node.js 18 or newer and npm

## Install and run

### 1. Backend environment and dependencies

From the project root, create and activate a virtual environment, then install the backend requirements:

**Windows PowerShell**

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r backend\requirements.txt
```

**macOS/Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r backend/requirements.txt
```

Start Flask from the project root:

```bash
python backend/app.py
```

The first startup creates `attendance.db`, creates indexes, cleans `data/attendance.csv`, writes `backend/cleaned_attendance.csv`, and seeds the database only when it is empty. The API is at `http://localhost:5000/api`; a health check is available at `/api/health`.

### 2. Frontend dependencies and development server

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the Vite URL printed in the terminal (normally `http://localhost:5173`). The browser frontend calls the Flask API at `http://localhost:5000/api`. To change this, copy `frontend/.env.example` to `frontend/.env` and set `VITE_API_URL`.

### 3. Import a CSV

Use **Students → Import CSV** in the UI, choose a `.csv` file, and confirm replacement of existing records. You can also call `POST /api/import` with a multipart form field named `file`.

Required fields can use either the project names or source dataset headers:

- `S_ID` / `student_id`
- `Name` / `name`
- `Sub` / `subject`
- `Total_Cls` / `total_classes`
- `Atn_Cls` / `attended_classes`

`Atn_%` and `Alert_Status` are optional and ignored. The cleaner trims text, removes exact duplicates and duplicate student IDs (keeps the first), validates missing/invalid rows and class counts, recalculates values, and writes `backend/cleaned_attendance.csv`. The response includes original, duplicate, invalid, missing, and final record counts. On successful import the database contents are replaced atomically within the SQLite transaction.

To re-seed from the default CSV after a demo, stop Flask, delete `attendance.db`, then start Flask again. (The cleaner output is regenerated.)

## API endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/health` | Backend health check |
| GET | `/api/students` | All student records |
| GET | `/api/students/<student_id>` | One student's attendance details |
| GET | `/api/alerts` | Records below 75%, with `Critical`/`Warning` severity |
| GET | `/api/dashboard` | Totals, average, subject averages/counts, distribution, safe-vs-alert |
| GET | `/api/subjects` | Unique subject names |
| POST | `/api/students` | Create; JSON includes student ID, name, subject, total and attended classes |
| PUT | `/api/students/<student_id>` | Update record (student ID stays the path ID) |
| DELETE | `/api/students/<student_id>` | Delete record |
| POST | `/api/import` | Import and replace records from multipart CSV field `file` |

Successful creation returns HTTP 201. Invalid fields return 400, missing IDs return 404, duplicate IDs return 409. API errors use `{"error": "..."}`.

## Data cleaning summary

The cleaning module returns `original_records`, `removed_duplicate_records`, `removed_exact_duplicates`, `removed_duplicate_student_ids`, `invalid_records`, `missing_values`, and `final_records`. Invalid rows include missing required fields, non-numeric class counts, total classes <= 0, negative attendance, and attended classes greater than total classes.

## Demonstration checklist

1. Start Flask and confirm `/api/health` returns `{"status":"ok"}`.
2. Open the frontend; confirm dashboard totals and charts load from the API.
3. Search by name/ID, filter by subject/status, and sort the student list.
4. Open student details and verify calculation/progress display.
5. Add a student under and over 75%; confirm alert and Safe status update.
6. Edit the record and delete it; verify totals refresh.
7. Open Low attendance and filter Warning/Critical.
8. Import a CSV and check its returned cleaning summary and dashboard refresh.
9. Open Analytics and verify the distribution, subject averages, and subject enrollment charts.

Run the backend API/import smoke tests from the project root (with backend dependencies installed):

```powershell
python -m unittest backend.test_app -v
```

Build-check the frontend from `frontend/` with `npm run build`.

For a production deployment, disable Flask debug mode and configure CORS to allow only the deployed frontend origin.
