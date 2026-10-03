# LinkEdge

A local hub that collects sensor data (LoRaWAN or plain HTTP), checks that the values make sense, stores them, and forwards them to your cloud via webhooks.

## How it works

```
LoRaWAN sensors → Gateway Bridge → MQTT → ChirpStack ─┐
                                                      ├→ Hub (FastAPI) → TimescaleDB → webhooks
Other sources (HTTP) ─────────────────────────────────┘
```

## Services

| Service | Port | What it does |
|---|---|---|
| `postgres` (TimescaleDB) | 5432 | Stores telemetry |
| `redis` | 6379 | Cache and retry buffer |
| `mosquitto` | 1883, 9001 | MQTT broker |
| `chirpstack-gateway-bridge` | 1700/udp | Converts Semtech UDP to MQTT |
| `chirpstack` | 8080 | LoRaWAN network server (web UI) |
| `hub` | 8000 | LinkEdge core: REST API and WebSocket |

## Quick start

You need Docker with Docker Compose.

```sh
cp .env.example .env     # then edit .env and set your own passwords and secret key
docker compose up --build -d
curl http://localhost:8000/health
```

Interactive API docs: http://localhost:8000/docs

## Send a reading

```sh
curl -X POST http://localhost:8000/api/v1/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "dev_eui": "sensor-01",
    "device_profile_id": "soil_sensor_v1",
    "data": {"ph": 6.5, "temperature_c": 21.3, "moisture_pct": 40}
  }'
```

The response tells you whether the reading is valid and why not, if it isn't.

## API

| Endpoint | Description |
|---|---|
| `POST /api/v1/ingest` | Send a reading over HTTP (for non-LoRaWAN sources) |
| `GET /api/v1/telemetry` | Query history (filter by device, time, validity) |
| `GET /api/v1/telemetry/{id}` | Get a single reading |
| `WS /api/v1/telemetry/ws/stream` | Live WebSocket feed |
| `/api/v1/devices` | Register devices and assign validation profiles (CRUD) |
| `GET /api/v1/devices/profiles` | List loaded device profiles |
| `/api/v1/export-targets` | Manage webhook destinations (CRUD) |
| `POST /api/v1/export-targets/{id}/test` | Send a test payload to a webhook |
| `GET /health` | Health check |

## Device profiles

A profile defines how a device's data is decoded and which values are allowed. Profiles are JSON files in [hub/device_profiles/](hub/device_profiles/).

- `generic`: pass-through with no rules. Used for unknown devices.
- `soil_sensor_v1`: JSON decoder that checks physical ranges for pH, temperature, moisture, EC, CO2, O2, CH4, N2O, vibroacoustic frequency and amplitude.

To add your own, drop a new JSON file in that folder and restart the hub.

## Configuration

All settings are in `.env` (see [.env.example](.env.example)):

| Variable | Description |
|---|---|
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | Database credentials |
| `MQTT_USERNAME`, `MQTT_PASSWORD` | MQTT login (empty = anonymous, dev only) |
| `MQTT_UPLINK_TOPIC` | Topic where ChirpStack publishes uplinks |
| `HUB_SECRET_KEY` | **Change this** before exposing the hub to a network |
| `EXPORT_BATCH_SIZE`, `EXPORT_RETRY_INTERVAL` | Webhook batch size and retry delay (seconds) |
| `LOG_LEVEL` | `DEBUG`, `INFO`, `WARNING` or `ERROR` |

## License

MIT. See [LICENSE](LICENSE).
