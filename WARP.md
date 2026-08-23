# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

TeslaMate is a self-hosted data logger for Tesla vehicles written in Elixir with Phoenix LiveView. It logs vehicle data to PostgreSQL, publishes to MQTT, and visualizes data through Grafana dashboards.

### Core Technologies
- **Language**: Elixir 1.12+
- **Web Framework**: Phoenix 1.7 with LiveView 0.20
- **Database**: PostgreSQL with Ecto
- **Frontend**: Phoenix LiveView with Bulma CSS, Leaflet.js for maps
- **MQTT**: Tortoise311 library for vehicle data publishing
- **Build**: esbuild for asset compilation

## Development Commands

### Initial Setup
```bash
mix setup              # Install dependencies, create/migrate DB, install npm packages
```

### Database Management
```bash
mix ecto.create        # Create the database
mix ecto.migrate       # Run migrations
mix ecto.reset         # Drop, create, and migrate database
mix ecto.drop          # Drop the database
```

### Running the Application
```bash
mix phx.server         # Start Phoenix server with code reloading (dev mode)
iex -S mix phx.server  # Start with interactive shell
```
The application runs on http://localhost:4000

### Testing
```bash
mix test                           # Run all tests
mix test test/path/to/file_test.exs # Run a specific test file
mix test test/path/to/file_test.exs:42 # Run a specific test at line 42
```
Tests automatically create and migrate a test database. The test alias in mix.exs runs tests with `--no-start` flag.

### Code Quality
```bash
mix format             # Format code according to .formatter.exs
mix format --check-formatted # Check if code is formatted (CI)
mix credo              # Run Credo linter
mix credo --strict     # Run with strict checks (max line length: 120)
mix dialyzer           # Run Dialyzer type checker
mix ci                 # Run all CI checks: format, deps check, tests
```

### Assets
```bash
cd assets && npm ci    # Install frontend dependencies
mix assets.deploy      # Build and digest assets for production
```

### Docker
```bash
make teslamate         # Build teslamate Docker image
make grafana           # Build teslamate-grafana Docker image
```

## Architecture

### Application Structure

TeslaMate follows a context-based Phoenix architecture with these key domains:

#### Core Contexts
- **TeslaMate.Log**: Database operations for cars, drives, charging, positions, states
- **TeslaMate.Vehicles**: Supervisor managing individual vehicle processes
- **TeslaMate.Api**: Tesla API client wrapper
- **TeslaMate.Mqtt**: MQTT publisher for real-time vehicle data
- **TeslaMate.Locations**: Geo-fencing and address lookup
- **TeslaMate.Settings**: Per-car and global settings management
- **TeslaMate.Terrain**: Elevation data lookup using SRTM
- **TeslaMate.Import**: Data import from TeslaFi and tesla-apiscraper
- **TeslaMate.Repair**: Background data repair and maintenance

#### Vehicle State Machine

Each vehicle runs as a GenStateMachine process (`TeslaMate.Vehicles.Vehicle`) with states:
- **:start** - Initial state, fetching vehicle data
- **:asleep** - Vehicle is asleep, minimal polling
- **:online** - Vehicle is online but idle
- **:driving** - Active driving, high-frequency polling
- **:charging** - Vehicle is charging
- **:updating** - Vehicle is receiving software update

The state machine uses different polling intervals per state (configurable via env vars):
- Asleep: 30s default
- Driving: 2.5s default
- Charging: 5s default
- Online: 60s default
- Default: 15s default

Circuit breakers (via `:fuse`) protect against API errors and missing vehicles.

#### Web Interface

Phoenix LiveView-based UI in `lib/teslamate_web/`:
- **Live views** in `live/` - Real-time vehicle monitoring, settings, import, geo-fencing
- **Controllers** in `controllers/` - Traditional HTTP endpoints
- **Templates** in `templates/` - HTML templates
- **Views** in `views/` - View helpers

Frontend assets built with esbuild:
- `assets/js/` - JavaScript modules
- `assets/css/` - SASS/SCSS stylesheets
- Uses Bulma CSS framework and Leaflet.js for maps

#### Data Flow

1. **Vehicle Supervisor** (`TeslaMate.Vehicles`) spawns one process per car
2. **Vehicle Process** polls Tesla API based on current state
3. **Log Context** persists data (positions, charges, drives, states) to PostgreSQL
4. **MQTT Publisher** broadcasts real-time data to MQTT broker
5. **LiveView** subscribes to PubSub for real-time UI updates
6. **Grafana** queries PostgreSQL for visualization dashboards

### Key Modules to Know

- `TeslaMate.Application` - Application supervisor tree
- `TeslaMate.Vehicles.Vehicle` - Vehicle state machine (GenStateMachine)
- `TeslaMate.Log` - Primary database context
- `TeslaMate.Api` - Tesla API client
- `TeslaApi.Vehicle` - Tesla API vehicle structs (external dep)
- `TeslaMateWeb.Router` - Route definitions
- `TeslaMate.Repo` - Ecto repository

## Development Patterns

### Database Migrations

Migrations are in `priv/repo/migrations/`. Follow Ecto conventions:
```bash
mix ecto.gen.migration migration_name
```

### Adding a New Feature to Vehicle Logging

1. Add fields to relevant schema in `lib/teslamate/log/` (e.g., `car.ex`, `drive.ex`)
2. Create migration in `priv/repo/migrations/`
3. Update `TeslaMate.Vehicles.Vehicle` state machine to capture new data
4. Update `TeslaMate.Log` functions to persist data
5. Add MQTT publishing in `lib/teslamate/mqtt/pubsub.ex` if needed
6. Update LiveView in `lib/teslamate_web/live/` for UI
7. Write tests in `test/teslamate/` mirroring lib structure

### Testing Patterns

- Unit tests mirror `lib/` structure in `test/`
- Use `TeslaMate.Factory` (if present) or fixtures in `test/fixtures/`
- Mock external APIs using `Mock` library (`:mock` dependency)
- Tests run with `--no-start` to avoid starting the full application
- Database uses `Ecto.Adapters.SQL.Sandbox` for test isolation

### Code Style

- Follow `mix format` rules in `.formatter.exs`
- Credo configuration in `.credo.exs` enforces style
- Max line length: 120 characters
- Use pipelines for data transformation
- Pattern match in function heads
- Use `with` for multiple operations that can fail

### Dependency Injection

Vehicle processes accept dependencies via opts for testability:
```elixir
deps = %{
  log: Keyword.get(opts, :deps_log, Log),
  api: Keyword.get(opts, :deps_api, Api),
  # ...
}
```

Use `Core.Dependency.call/2` or `call/3` to invoke dependency functions in tests.

## Configuration

Environment-specific config in `config/`:
- `config.exs` - Base configuration
- `dev.exs` - Development (code reloading, debug errors)
- `test.exs` - Test environment
- `prod.exs` - Production settings
- `runtime.exs` - Runtime configuration (for releases)

Database connection configured via environment variables in production.

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
If CSS/JS changes don't appear, ensure the watcher is running (automatic with `mix phx.server` in dev).

## Deployment to Test Server

### Testing Deployment on fra1.armlab.com

The test instance runs at https://teslamate.armlab.com using Docker Compose.

**IMPORTANT**: Always build Docker images on the server (AMD64 architecture), not on macOS (ARM64). Cross-architecture images cause performance issues and compatibility warnings.

#### Deployment Process

1. **Push your changes to GitHub**:
   ```bash
   git add .
   git commit -m "Your changes"
   git push
   ```

2. **SSH to the server and pull latest changes**:
   ```bash
   ssh wouter@fra1.armlab.com
   cd ~/teslamate-build
   git fetch origin
   git checkout feature/dark-mode  # or your branch
   git reset --hard origin/feature/dark-mode
   ```

3. **Build the Docker image on the server**:
   ```bash
   DOCKER_BUILDKIT=1 docker build -t woooter/teslamate:dark-mode .
   ```
   Note: BuildKit is required for modern Dockerfile features.

4. **Restart the container**:
   ```bash
   cd ~/teslamate
   docker compose up -d teslamate
   ```

5. **Verify the deployment**:
   - Visit https://teslamate.armlab.com
   - Check logs if needed: `docker compose logs -f teslamate`

#### Server Configuration

- Docker Compose file: `~/teslamate/docker-compose.yml`
- Build directory: `~/teslamate-build/` (clone of the repository)
- Image tag: `woooter/teslamate:dark-mode`
- Database and volumes persist between deployments

## External Documentation

- Main documentation: https://docs.teslamate.org
- Development guide: https://docs.teslamate.org/docs/development/
- Tesla API: Third-party `tesla_api` library (Elixir package)
