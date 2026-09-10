# Agriminds

Smart farming dashboard with a lightweight Node.js backend. No package installation is required.

## Run locally

1. Open a terminal in this folder.
2. Run `npm start`.
3. Open `http://localhost:3000` in a browser.

The dashboard loads live values from the backend when served through this address. Opening `index.html` directly still shows the demo dashboard, but pump controls will require the server.

## API routes

| Method | Route | Purpose |
| --- | --- | --- |
| GET | `/api/health` | API health check |
| GET | `/api/dashboard` | Farm, sensor, pump, and alert data |
| GET / POST | `/api/sensors` | Read or send sensor measurements |
| GET / POST | `/api/irrigation` | Read or switch the irrigation pump |
| GET | `/api/alerts` | Current climate-risk alerts |

Example to stop the pump:

```json
POST /api/irrigation
{ "running": false }
```

Sensor readings are stored in `data.json`. A critically low water level (10% or lower) automatically stops the pump to protect the motor.
