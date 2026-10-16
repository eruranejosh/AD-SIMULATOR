# Qarba Ad Simulator

Qarba Ad Simulator is a web application designed to help the Qarba marketing team estimate the potential cost and performance of Facebook and Instagram advertising campaigns.

The application provides configurable campaign budgets, durations, target audiences, and scenario-based forecasts to support marketing decisions.

## Features

- Configure campaign names, budgets, and durations.
- Select the target audience for property buyers and tenants.
- Estimate impressions, clicks, enquiries, cost per click, and cost per enquiry.
- Compare conservative, expected, and optimistic performance scenarios.
- Save and retrieve campaigns.
- Display budgets and estimated advertising costs in Nigerian naira (NGN).

**Note:** Forecasts are estimates based on configurable assumptions, not guaranteed results or live Meta advertising data.

## Technology Stack

### Frontend
- Next.js
- React
- TypeScript

### Backend
- Python
- FastAPI
- SQLite

## Project Structure

```text
qarba-ad-simulator/
├── backend/
│   └── main.py
├── frontend/
│   ├── app/
│   │   └── page.tsx
│   ├── package.json
│   └── ...
├── .gitignore
└── README.md
```

## Getting Started

### Prerequisites

Install the following:

- Python 3
- Node.js and npm

### 1. Start the backend

Open a terminal and navigate to the backend directory:

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install fastapi "uvicorn[standard]" sqlalchemy
uvicorn main:app --reload --port 8000
```

The API should be available at:

`http://127.0.0.1:8000`

Interactive API documentation:

`http://127.0.0.1:8000/docs`

### 2. Start the frontend

Open a second terminal:

```bash
cd ~/qarba-ad-simulator/frontend
npm install
npm run dev
```

Open the application at:

`http://localhost:3000`

Ensure the frontend's API configuration points to the running backend at `http://127.0.0.1:8000`.

## Configuration

Adjust campaign assumptions and forecasting parameters within the application as needed. Keep credentials, private configuration files, and environment secrets out of version control.

## Project Status

Under development.

## Disclaimer

This tool supports campaign planning and budgeting. Actual advertising performance depends on audience targeting, creative quality, competition, platform delivery, and other factors. Estimates should not be treated as guaranteed outcomes.
