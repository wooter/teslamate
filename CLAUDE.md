# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TeslaMate is a self-hosted data logger for Tesla vehicles written in Elixir. It logs vehicle data to PostgreSQL and provides visualization through Grafana dashboards. Vehicle data is also published to MQTT for integration with home automation systems.

## Development Commands

### Initial Setup
```bash
mix setup                                  # Install deps, setup DB, install npm packages
```

### Database Management
```bash
mix ecto.create                            # Create database
mix ecto.migrate                           # Run migrations
mix ecto.reset                             # Drop and recreate database with migrations
```

### Running the Application
```bash
iex -S mix phx.server                      # Start interactive server
mix phx.server                             # Start server (non-interactive)
```
Application runs on http://localhost:4000

### Testing
```bash
mix test                                   # Run all tests (creates test DB, runs migrations, tests)
mix test test/path/to/test.exs             # Run specific test file
mix test test/path/to/test.exs:42          # Run specific test at line 42
mix test --trace                           # Run with detailed output
mix test --only focus                      # Run only tests tagged with @tag :focus
mix ci                                     # CI pipeline (format check, unused deps check, test with warnings as errors)
```

### Code Quality
```bash
mix format                                 # Format Elixir code
mix format --check-formatted               # Check if code is formatted
mix credo                                  # Run static analysis
mix credo --strict                         # Run strict analysis
mix dialyzer                               # Run type checking (slow on first run)
treefmt                                    # Format entire codebase (Elixir, JS, shell, etc)
```

### Frontend Assets
```bash
cd assets && npm ci                        # Install frontend dependencies
mix assets.deploy                          # Build production assets (runs npm deploy and phx.digest)
```

### Docker
```bash
make teslamate                             # Build teslamate Docker image
make grafana                               # Build teslamate-grafana Docker image
```

## Architecture

### Core Components

**Application Supervision Tree** (`lib/teslamate/application.ex`):
- Standard mode: Repo → Vault → HTTP → Api → Updater → PubSub → Endpoint → Terrain → Vehicles → MQTT → Repair
- Import mode: Skips Vehicles and MQTT, starts Import supervisor instead

**Vehicle State Management** (`lib/teslamate/vehicles/`):
- `TeslaMate.Vehicles` - Supervisor managing individual vehicle processes
- `TeslaMate.Vehicles.Vehicle` - GenStateMachine (not GenServer) for each vehicle
- Vehicle states:
  - `:start` - Initial state, fetching vehicle data
  - `:asleep` - Vehicle is asleep, minimal polling
  - `:online` - Vehicle is online but idle
  - `:driving` - Active driving, high-frequency polling
  - `:charging` - Vehicle is charging
  - `:updating` - Vehicle is receiving software update
- Polling intervals per state (all configurable via env vars):
  - Asleep: 30s default (`POLLING_ASLEEP_INTERVAL`)
  - Driving: 2.5s default (`POLLING_DRIVING_INTERVAL`)
  - Charging: 5s default (`POLLING_CHARGING_INTERVAL`)
  - Online: 60s default (`POLLING_ONLINE_INTERVAL`)
  - Default: 15s default (`POLLING_DEFAULT_INTERVAL`)
- Circuit breakers (`:fuse` library) protect against API errors and missing vehicles

**Data Logging** (`lib/teslamate/log/`):
- Schema modules: `Car`, `Drive`, `Charge`, `ChargingProcess`, `Position`, `State`, `Update`
- `TeslaMate.Log` context provides functions for creating and querying logged data
- All driving and charging sessions are logged with positions, energy consumption, etc.

**Tesla API Client** (`lib/tesla_api/`):
- `TeslaApi.Auth` - OAuth authentication and token management
- `TeslaApi.Vehicle` - Vehicle data fetching
- `TeslaApi.Stream` - WebSocket streaming for real-time vehicle data
- Built on the `Tesla` HTTP client library with custom middleware

**MQTT Integration** (`lib/teslamate/mqtt/`):
- Publishes vehicle state and data to MQTT broker using Tortoise311 client
- Supports namespacing for multi-vehicle setups
- `TeslaMate.Mqtt.Publisher` publishes vehicle data
- `TeslaMate.Mqtt.PubSub` subscribes to Phoenix.PubSub and forwards to MQTT

**Web Interface** (`lib/teslamate_web/`):
- Phoenix LiveView-based UI for vehicle monitoring and settings
- Live views:
  - `CarLive` - Main vehicle dashboard
  - `ChargeLive` - Charging session details
  - `GeoFenceLive` - Geofence management
  - `SettingsLive` - Application settings
  - `ImportLive` - Data import from TeslaFi/tesla-apiscraper
  - `SigninLive` - Tesla account authentication

**Other Key Modules**:
- `TeslaMate.Locations` - Geofencing and address lookup
- `TeslaMate.Terrain` - Elevation data fetching using SRTM
- `TeslaMate.Vault` - Encryption for sensitive data using Cloak
- `TeslaMate.Settings` - Global and per-car settings
- `TeslaMate.Import` - Import data from TeslaFi and tesla-apiscraper
- `TeslaMate.Repair` - Data repair and cleanup utilities

### Data Flow

1. **Vehicle Supervisor** (`TeslaMate.Vehicles`) spawns one process per car
2. **Vehicle Process** polls Tesla API based on current state
3. **Log Context** persists data (positions, charges, drives, states) to PostgreSQL
4. **MQTT Publisher** broadcasts real-time data to MQTT broker
5. **LiveView** subscribes to PubSub for real-time UI updates
6. **Grafana** queries PostgreSQL for visualization dashboards

### Configuration

Runtime configuration is in `config/runtime.exs`. It reads environment variables for:
- Database connection (DATABASE_URL)
- MQTT settings (MQTT_HOST, MQTT_USERNAME, etc.)
- Tesla API authentication
- Application features (geofencing, address lookup, etc.)

Development config in `config/dev.exs`, test config in `config/test.exs`, and production config in `config/prod.exs`.

### Database

Uses PostgreSQL with Ecto as the ORM. Migrations are in `priv/repo/migrations/`. The schema tracks:
- Cars and their configurations
- Driving sessions with GPS positions
- Charging sessions with power/energy data
- Vehicle states (online, asleep, etc.)
- Software updates
- Settings (global and per-car)

### Frontend

Assets in `assets/` built with:
- esbuild for JavaScript bundling
- Sass for CSS
- Bulma CSS framework
- Leaflet for maps
- Phoenix LiveView for reactivity

Build script: `assets/scripts/build.js`

## Development Patterns

### Adding a New Feature to Vehicle Logging

1. Add fields to relevant schema in `lib/teslamate/log/` (e.g., `car.ex`, `drive.ex`)
2. Create migration: `mix ecto.gen.migration migration_name`
3. Update `TeslaMate.Vehicles.Vehicle` state machine to capture new data
4. Update `TeslaMate.Log` functions to persist data
5. Add MQTT publishing in `lib/teslamate/mqtt/pubsub.ex` if needed
6. Update LiveView in `lib/teslamate_web/live/` for UI
7. Write tests in `test/teslamate/` mirroring `lib/` structure

### Dependency Injection for Testing

Vehicle processes accept dependencies via opts for testability:
```elixir
deps = %{
  log: Keyword.get(opts, :deps_log, Log),
  api: Keyword.get(opts, :deps_api, Api),
  # ...
}
```
Use `Core.Dependency.call/2` or `call/3` to invoke dependency functions in tests.

## Testing Patterns

Tests use ExUnit with:
- Async tests where possible (`use TeslaMateTest.Case, async: true`)
- Sandbox mode for database isolation
- Mock library for external API calls
- Test fixtures in `test/fixtures/`
- Helpers in `test/support/`

Common test patterns in vehicle tests (`test/teslamate/vehicles/`):
- Start a vehicle process for each test
- Send mock API responses
- Assert state transitions and logged data
- Use `assert_receive` for async message handling

## Code Style

- Follow standard Elixir conventions
- Max line length: 120 characters
- Use `mix format` for consistent formatting (config in `.formatter.exs`)
- Run `mix credo` for style and complexity checks (config in `.credo.exs`)
- Use pipelines for data transformation
- Pattern match in function heads
- Use `with` for multiple operations that can fail
- Prefer explicit over implicit
- Document public functions with `@doc`
- Use `@moduledoc` for module-level documentation
- Dialyzer is configured but runs slowly - use judiciously

## Important Notes

- The application can run in import mode (set IMPORT_DIR env var) which disables vehicle logging and starts the import process
- Vehicle processes use GenStateMachine for complex state management (not GenServer)
- All timestamps should be in UTC
- Encryption is handled by Cloak for sensitive fields (tokens, passwords)
- The app is designed to minimize vampire drain by allowing vehicles to sleep
- MQTT namespace must not contain "/" character
- Database migrations should never modify existing Grafana dashboards' schema assumptions

## Common Issues

### Database Connection
Ensure PostgreSQL is running and environment variables are set:
- `DATABASE_USER`
- `DATABASE_PASS`
- `DATABASE_NAME`
- `DATABASE_HOST`

### MQTT Testing
MQTT features are optional. Tests should handle cases where MQTT is disabled (config set to `nil`).

### Asset Compilation
If CSS/JS changes don't appear, ensure the watcher is running (automatic with `mix phx.server` in dev mode).
