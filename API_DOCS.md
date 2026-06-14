# API Documentation — CG State Risk Engine

> **Base URL:** `http://localhost:8001`
> **Format:** All request and response bodies are `application/json` unless noted.
> **Authentication:** The backend API itself does not enforce token-based auth. Access control is enforced on the frontend via `AuthContext`. Ensure the backend is not publicly exposed without a reverse-proxy auth layer in production.

---

## Table of Contents

- [Health](#health)
  - [GET /healthz](#get-healthz)
  - [DELETE /cache](#delete-cache)
- [Forecasting](#forecasting)
  - [POST /forecast](#post-forecast)
  - [POST /forecast/compare](#post-forecastcompare)
  - [POST /forecast/run](#post-forecastrun)
  - [GET /forecast/stream/{runId}](#get-forecaststreamrunid)
- [Facilities](#facilities)
  - [GET /facilities](#get-facilities)
  - [GET /facilities/{id}](#get-facilitiesid)
  - [POST /facilities](#post-facilities)
  - [POST /facilities/bulk](#post-facilitiesbulk)
  - [PUT /facilities/{id}](#put-facilitiesid)
  - [DELETE /facilities/{id}](#delete-facilitiesid)
- [Error Responses](#error-responses)

---

## Health

### `GET /healthz`

Returns the current health status of the service and cache statistics.

**Authentication:** None required.

**Request Parameters:** None.

**Response — `200 OK`**

```json
{
  "status": "ok",
  "cache": {
    "size": 12,
    "keys": ["forecast:district_A:prophet", "forecast:district_B:random_forest"]
  }
}
```

| Field | Type | Description |
|---|---|---|
| `status` | `string` | Always `"ok"` when the service is running. |
| `cache.size` | `integer` | Number of entries currently in the prediction cache. |
| `cache.keys` | `string[]` | List of cache key identifiers. |

**cURL Example**
```bash
curl http://localhost:8001/healthz
```

---

### `DELETE /cache`

Clears the entire in-memory prediction cache. Useful when you want to force fresh model runs.

**Authentication:** None required.

**Request Parameters:** None.

**Response — `200 OK`**

```json
{
  "status": "cleared",
  "message": "Prediction cache has been cleared."
}
```

**cURL Example**
```bash
curl -X DELETE http://localhost:8001/cache
```

---

## Forecasting

### `POST /forecast`

Runs a single risk forecast using either the **Prophet** or **Random Forest** model for a given time series.

**Authentication:** None required.

**Request Body**

```json
{
  "data": [
    { "date": "2024-01-01", "value": 34.5 },
    { "date": "2024-01-02", "value": 36.1 }
  ],
  "horizon": 30,
  "model": "prophet",
  "metric_name": "temperature"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `data` | `array` | ✅ | Historical time-series input. Each item must have `date` (ISO 8601 string) and `value` (number). Minimum ~10 data points recommended. |
| `horizon` | `integer` | ✅ | Forecast horizon in days. Accepted values: `30`, `60`, `90`. |
| `model` | `string` | ✅ | Model to use. Accepted values: `"prophet"`, `"random_forest"`. |
| `metric_name` | `string` | ✅ | Human-readable label for the metric being forecast (e.g., `"rainfall"`, `"temperature"`). |

**Response — `200 OK`**

```json
{
  "metric_name": "temperature",
  "model": "prophet",
  "horizon": 30,
  "forecast": [
    {
      "date": "2024-02-01",
      "predicted": 38.2,
      "lower": 35.0,
      "upper": 41.4
    }
  ],
  "accuracy": {
    "mape": 4.72
  }
}
```

| Field | Type | Description |
|---|---|---|
| `metric_name` | `string` | Echoed from the request. |
| `model` | `string` | Model used. |
| `horizon` | `integer` | Forecast horizon in days. |
| `forecast` | `array` | Array of predicted data points. Each item has `date`, `predicted`, `lower` (confidence bound), and `upper` (confidence bound). |
| `accuracy.mape` | `float` | Mean Absolute Percentage Error (%) from backtesting. Lower is better. |

**Status Codes**

| Code | Description |
|---|---|
| `200` | Forecast generated successfully. |
| `422` | Validation error — malformed request body. |
| `500` | Internal server error during model execution. |

**cURL Example**
```bash
curl -X POST http://localhost:8001/forecast \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      {"date": "2024-01-01", "value": 34.5},
      {"date": "2024-01-02", "value": 36.1},
      {"date": "2024-01-03", "value": 33.8}
    ],
    "horizon": 30,
    "model": "prophet",
    "metric_name": "temperature"
  }'
```

---

### `POST /forecast/compare`

Runs the same dataset through **both** Prophet and Random Forest models simultaneously and returns a side-by-side comparison.

**Authentication:** None required.

**Request Body**

Same schema as [`POST /forecast`](#post-forecast), except `model` is ignored — both models always run.

```json
{
  "data": [
    { "date": "2024-01-01", "value": 34.5 },
    { "date": "2024-01-02", "value": 36.1 }
  ],
  "horizon": 60,
  "metric_name": "rainfall"
}
```

**Response — `200 OK`**

```json
{
  "metric_name": "rainfall",
  "horizon": 60,
  "prophet": {
    "forecast": [...],
    "accuracy": { "mape": 5.1 }
  },
  "random_forest": {
    "forecast": [...],
    "accuracy": { "mape": 6.3 }
  }
}
```

| Field | Type | Description |
|---|---|---|
| `prophet` | `object` | Forecast result from the Prophet model. Same shape as the `forecast` + `accuracy` fields in `POST /forecast`. |
| `random_forest` | `object` | Forecast result from the Random Forest model. |

**Status Codes**

| Code | Description |
|---|---|
| `200` | Both forecasts generated successfully. |
| `422` | Validation error. |
| `500` | Model execution error. |

**cURL Example**
```bash
curl -X POST http://localhost:8001/forecast/compare \
  -H "Content-Type: application/json" \
  -d '{
    "data": [{"date": "2024-01-01", "value": 34.5}],
    "horizon": 60,
    "metric_name": "rainfall"
  }'
```

---

### `POST /forecast/run`

Starts a **bulk background forecasting job** across multiple facilities. Returns a `runId` immediately; use the streaming endpoint to receive results.

**Authentication:** None required.

**Request Body**

```json
{
  "facilities": [
    {
      "id": "FAC001",
      "name": "District Hospital Raipur",
      "data": [
        { "date": "2024-01-01", "value": 34.5 }
      ]
    }
  ],
  "horizon": 30,
  "model": "prophet",
  "metric_name": "flood_risk_index"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `facilities` | `array` | ✅ | List of facility objects, each with `id`, `name`, and `data` (same time-series schema as `/forecast`). |
| `horizon` | `integer` | ✅ | Forecast horizon: `30`, `60`, or `90`. |
| `model` | `string` | ✅ | `"prophet"` or `"random_forest"`. |
| `metric_name` | `string` | ✅ | Metric label. |

**Response — `202 Accepted`**

```json
{
  "runId": "a3f9c12e-4b77-4d2a-b8e1-0c1234567890",
  "status": "started",
  "total": 5
}
```

| Field | Type | Description |
|---|---|---|
| `runId` | `string` | UUID identifying this bulk run. Pass to `/forecast/stream/{runId}`. |
| `status` | `string` | Always `"started"` on success. |
| `total` | `integer` | Total number of facilities queued for processing. |

**Status Codes**

| Code | Description |
|---|---|
| `202` | Bulk run started. |
| `422` | Validation error. |

**cURL Example**
```bash
curl -X POST http://localhost:8001/forecast/run \
  -H "Content-Type: application/json" \
  -d '{
    "facilities": [
      {
        "id": "FAC001",
        "name": "District Hospital Raipur",
        "data": [{"date": "2024-01-01", "value": 34.5}]
      }
    ],
    "horizon": 30,
    "model": "prophet",
    "metric_name": "flood_risk_index"
  }'
```

---

### `GET /forecast/stream/{runId}`

**Server-Sent Events (SSE)** endpoint. Streams real-time progress updates and results for a bulk forecast run started via `POST /forecast/run`.

**Authentication:** None required.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `runId` | `string` | UUID returned by `POST /forecast/run`. |

**Response — `200 OK`**

Content-Type: `text/event-stream`

Events are emitted as the backend processes each facility. There are three event types:

**Progress event:**
```
data: {"type": "progress", "completed": 2, "total": 5, "facilityId": "FAC002"}
```

**Result event (per facility):**
```
data: {
  "type": "result",
  "facilityId": "FAC002",
  "facilityName": "CHC Durg",
  "forecast": [...],
  "accuracy": {"mape": 4.2}
}
```

**Completion event:**
```
data: {"type": "done", "completed": 5, "total": 5}
```

**Status Codes**

| Code | Description |
|---|---|
| `200` | Stream opened successfully. |
| `404` | `runId` not found or already expired. |

**JavaScript Example**
```js
const source = new EventSource(`http://localhost:8001/forecast/stream/${runId}`);

source.onmessage = (event) => {
  const payload = JSON.parse(event.data);
  if (payload.type === "done") source.close();
  console.log(payload);
};
```

---

## Facilities

### `GET /facilities`

Returns a list of all facilities stored in the SQLite database.

**Authentication:** None required.

**Query Parameters:** None.

**Response — `200 OK`**

```json
[
  {
    "id": "FAC001",
    "name": "District Hospital Raipur",
    "type": "hospital",
    "district": "Raipur",
    "lat": 21.2514,
    "lng": 81.6296,
    "capacity": 300,
    "risk_score": 72.4
  }
]
```

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique facility identifier. |
| `name` | `string` | Facility name. |
| `type` | `string` | `"hospital"` or `"school"`. |
| `district` | `string` | Chhattisgarh district name. |
| `lat` | `float` | Latitude coordinate. |
| `lng` | `float` | Longitude coordinate. |
| `capacity` | `integer` | Facility capacity (beds / students). |
| `risk_score` | `float` | Latest computed risk score (0–100). |

**cURL Example**
```bash
curl http://localhost:8001/facilities
```

---

### `GET /facilities/{id}`

Returns details for a single facility.

**Authentication:** None required.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | `string` | Facility ID. |

**Response — `200 OK`**

```json
{
  "id": "FAC001",
  "name": "District Hospital Raipur",
  "type": "hospital",
  "district": "Raipur",
  "lat": 21.2514,
  "lng": 81.6296,
  "capacity": 300,
  "risk_score": 72.4
}
```

**Status Codes**

| Code | Description |
|---|---|
| `200` | Facility found. |
| `404` | Facility not found. |

**cURL Example**
```bash
curl http://localhost:8001/facilities/FAC001
```

---

### `POST /facilities`

Creates a new facility or updates an existing one (upsert by `id`).

**Authentication:** None required.

**Request Body**

```json
{
  "id": "FAC002",
  "name": "Government School Bilaspur",
  "type": "school",
  "district": "Bilaspur",
  "lat": 22.0796,
  "lng": 82.1391,
  "capacity": 450,
  "risk_score": 55.0
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | `string` | ✅ | Unique facility ID. |
| `name` | `string` | ✅ | Facility name. |
| `type` | `string` | ✅ | `"hospital"` or `"school"`. |
| `district` | `string` | ✅ | District name. |
| `lat` | `float` | ✅ | Latitude. |
| `lng` | `float` | ✅ | Longitude. |
| `capacity` | `integer` | ❌ | Facility capacity. |
| `risk_score` | `float` | ❌ | Initial risk score. |

**Response — `201 Created`**

```json
{
  "id": "FAC002",
  "name": "Government School Bilaspur",
  "type": "school",
  "district": "Bilaspur",
  "lat": 22.0796,
  "lng": 82.1391,
  "capacity": 450,
  "risk_score": 55.0
}
```

**Status Codes**

| Code | Description |
|---|---|
| `201` | Facility created. |
| `200` | Facility updated (upsert). |
| `422` | Validation error. |

**cURL Example**
```bash
curl -X POST http://localhost:8001/facilities \
  -H "Content-Type: application/json" \
  -d '{
    "id": "FAC002",
    "name": "Government School Bilaspur",
    "type": "school",
    "district": "Bilaspur",
    "lat": 22.0796,
    "lng": 82.1391,
    "capacity": 450,
    "risk_score": 55.0
  }'
```

---

### `POST /facilities/bulk`

Bulk upsert multiple facilities in a single request. Existing records are updated; new records are inserted.

**Authentication:** None required.

**Request Body**

```json
{
  "facilities": [
    {
      "id": "FAC001",
      "name": "District Hospital Raipur",
      "type": "hospital",
      "district": "Raipur",
      "lat": 21.2514,
      "lng": 81.6296
    },
    {
      "id": "FAC002",
      "name": "Government School Bilaspur",
      "type": "school",
      "district": "Bilaspur",
      "lat": 22.0796,
      "lng": 82.1391
    }
  ]
}
```

**Response — `200 OK`**

```json
{
  "inserted": 1,
  "updated": 1,
  "total": 2
}
```

| Field | Type | Description |
|---|---|---|
| `inserted` | `integer` | Number of new records created. |
| `updated` | `integer` | Number of existing records updated. |
| `total` | `integer` | Total records processed. |

**Status Codes**

| Code | Description |
|---|---|
| `200` | Bulk operation completed. |
| `422` | Validation error. |

**cURL Example**
```bash
curl -X POST http://localhost:8001/facilities/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "facilities": [
      {"id": "FAC001", "name": "District Hospital Raipur", "type": "hospital", "district": "Raipur", "lat": 21.2514, "lng": 81.6296}
    ]
  }'
```

---

### `PUT /facilities/{id}`

Updates an existing facility by ID. Only provided fields are updated.

**Authentication:** None required.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | `string` | ID of the facility to update. |

**Request Body**

Partial update — include only the fields you want to change:

```json
{
  "risk_score": 88.5,
  "capacity": 320
}
```

**Response — `200 OK`**

```json
{
  "id": "FAC001",
  "name": "District Hospital Raipur",
  "type": "hospital",
  "district": "Raipur",
  "lat": 21.2514,
  "lng": 81.6296,
  "capacity": 320,
  "risk_score": 88.5
}
```

**Status Codes**

| Code | Description |
|---|---|
| `200` | Facility updated successfully. |
| `404` | Facility not found. |
| `422` | Validation error. |

**cURL Example**
```bash
curl -X PUT http://localhost:8001/facilities/FAC001 \
  -H "Content-Type: application/json" \
  -d '{"risk_score": 88.5, "capacity": 320}'
```

---

### `DELETE /facilities/{id}`

Permanently removes a facility from the database.

**Authentication:** None required.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `id` | `string` | ID of the facility to delete. |

**Response — `200 OK`**

```json
{
  "status": "deleted",
  "id": "FAC001"
}
```

**Status Codes**

| Code | Description |
|---|---|
| `200` | Facility deleted. |
| `404` | Facility not found. |

**cURL Example**
```bash
curl -X DELETE http://localhost:8001/facilities/FAC001
```

---

## Error Responses

All error responses follow a consistent structure:

```json
{
  "detail": "Human-readable error description."
}
```

### Standard HTTP Error Codes

| Code | Meaning | Common Causes |
|---|---|---|
| `400` | Bad Request | Malformed JSON body. |
| `404` | Not Found | Resource (facility, runId) does not exist. |
| `422` | Unprocessable Entity | Request body fails schema validation. FastAPI returns a detailed `detail` array listing each field error. |
| `500` | Internal Server Error | Model execution failure, database error, or unhandled exception. Check server logs. |

### `422` Validation Error Example

FastAPI returns structured validation errors for `422` responses:

```json
{
  "detail": [
    {
      "loc": ["body", "horizon"],
      "msg": "value is not a valid integer",
      "type": "type_error.integer"
    }
  ]
}
```

### SSE Stream Error

If a `runId` is invalid or has expired, the stream endpoint returns:

```json
{
  "detail": "Run ID not found."
}
```

---

## Notes

- **CORS:** FastAPI is configured to accept requests from the Vite dev server (`localhost:5173`). Update `allow_origins` in `main.py` for production deployments.
- **Concurrency:** The bulk `/forecast/run` endpoint executes facility forecasts sequentially in a background task. For large batches (50+ facilities), expect proportionally longer processing times.
- **Cache invalidation:** Forecast results are cached in memory keyed by district + model. Call `DELETE /cache` to invalidate after uploading new historical data.
- **Database location:** SQLite database file is at `forecast-service/facilities.db`. Back this file up before running bulk destructive operations.
