# Go Boilerplate

A production-ready, full-stack monorepo boilerplate built with **Go** (backend) and **TypeScript** (API contracts & tooling). It comes pre-configured with authentication, observability, background jobs, transactional emails, database migrations, and a comprehensive testing framework — so you can focus on building your product instead of setting up infrastructure.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Install Dependencies](#2-install-dependencies)
  - [3. Set Up the Database](#3-set-up-the-database)
  - [4. Configure Environment Variables](#4-configure-environment-variables)
  - [5. Run the Application](#5-run-the-application)
- [Backend Deep Dive](#backend-deep-dive)
  - [Entry Point](#entry-point)
  - [Configuration System](#configuration-system)
  - [Server Initialization](#server-initialization)
  - [Routing & Middleware Pipeline](#routing--middleware-pipeline)
  - [Handler System (Go Generics)](#handler-system-go-generics)
  - [Request Validation](#request-validation)
  - [Error Handling System](#error-handling-system)
  - [Database Layer](#database-layer)
  - [Database Migrations](#database-migrations)
  - [Authentication (Clerk)](#authentication-clerk)
  - [Background Jobs (Asynq)](#background-jobs-asynq)
  - [Email System (Resend)](#email-system-resend)
  - [Logging (Zerolog)](#logging-zerolog)
  - [Observability (New Relic)](#observability-new-relic)
  - [Repository & Service Layers](#repository--service-layers)
- [TypeScript Packages](#typescript-packages)
  - [packages/zod — Zod Schemas](#packageszod--zod-schemas)
  - [packages/openai — OpenAPI Generation](#packagesopenai--openapi-generation)
  - [packages/emails — Email Templates](#packagesemails--email-templates)
- [Testing](#testing)
  - [Test Infrastructure (Testcontainers)](#test-infrastructure-testcontainers)
  - [Test Utilities](#test-utilities)
  - [Running Tests](#running-tests)
- [Available Commands](#available-commands)
  - [Root-Level (Turborepo)](#root-level-turborepo)
  - [Backend (Taskfile)](#backend-taskfile)
- [Adding a New Feature (Step-by-Step)](#adding-a-new-feature-step-by-step)
- [Tech Stack Summary](#tech-stack-summary)
- [License](#license)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         MONOREPO (Turborepo + Bun)              │
├───────────────────────────┬─────────────────────────────────────┤
│        apps/backend       │          packages/                  │
│        (Go + Echo)        │  ┌─────────┬─────────┬──────────┐  │
│                           │  │  zod    │ openai  │  emails  │  │
│  ┌─────────────────────┐  │  │ (Zod   │(OpenAPI │ (React   │  │
│  │   HTTP Server        │  │  │schemas)│  gen)   │  Email)  │  │
│  │   (Echo Framework)   │  │  └─────────┴─────────┴──────────┘  │
│  ├─────────────────────┤  │                                     │
│  │   Middleware Stack   │  │  TypeScript packages that generate  │
│  │  • Rate Limiting     │  │  API contracts and email templates  │
│  │  • CORS              │  │  consumed by the Go backend.        │
│  │  • Auth (Clerk)      │  │                                     │
│  │  • Request ID        │  │                                     │
│  │  • Tracing (NR)      │  │                                     │
│  │  • Context Enhancer  │  │                                     │
│  │  • Request Logger    │  │                                     │
│  │  • Panic Recovery    │  │                                     │
│  ├─────────────────────┤  │                                     │
│  │   Handlers           │  │                                     │
│  │  (Go Generics)       │  │                                     │
│  ├─────────────────────┤  │                                     │
│  │   Services           │  │                                     │
│  ├─────────────────────┤  │                                     │
│  │   Repositories       │  │                                     │
│  ├─────────────────────┤  │                                     │
│  │   PostgreSQL (pgx)   │  │                                     │
│  │   Redis (go-redis)   │  │                                     │
│  │   Asynq (Job Queue)  │  │                                     │
│  └─────────────────────┘  │                                     │
└───────────────────────────┴─────────────────────────────────────┘
```

**How it works at a glance:**

1. An HTTP request arrives at the **Echo** web server.
2. It passes through a **middleware pipeline** (rate limiter → CORS → request ID → New Relic tracing → context enrichment → request logger → panic recovery).
3. The **router** dispatches it to the correct **handler**.
4. The handler uses the **generic `Handle` function** to automatically bind the request, validate it, call business logic, and return a typed response — all with built-in logging and tracing.
5. Business logic lives in the **service layer**, which talks to **repositories** for database access.
6. Any database errors are automatically mapped to user-friendly HTTP errors by the **sqlerr** package.
7. Background work (like sending emails) is queued via **Asynq** (Redis-backed) and processed asynchronously.

---

## Project Structure

```
go-boilerplate/
├── apps/
│   └── backend/                    # Go backend application
│       ├── cmd/
│       │   └── go-boilerplate/
│       │       └── main.go         # Entry point
│       ├── internal/
│       │   ├── config/             # Configuration loading & validation
│       │   ├── database/           # PostgreSQL connection & migrations
│       │   ├── errs/               # Structured HTTP error types
│       │   ├── handler/            # HTTP handlers (Go generics)
│       │   ├── lib/
│       │   │   ├── email/          # Email client (Resend)
│       │   │   ├── job/            # Background job queue (Asynq)
│       │   │   └── utils/          # Shared utilities
│       │   ├── logger/             # Logging setup (Zerolog + New Relic)
│       │   ├── middleware/         # HTTP middleware stack
│       │   ├── model/              # Data models (empty, add yours)
│       │   ├── repository/         # Database access layer
│       │   ├── router/             # Route definitions
│       │   ├── server/             # Server struct & lifecycle
│       │   ├── service/            # Business logic layer
│       │   ├── sqlerr/             # SQL error → HTTP error mapping
│       │   ├── testing/            # Test infrastructure
│       │   └── validation/         # Request validation framework
│       ├── static/                 # Static files (OpenAPI spec & UI)
│       ├── template/               # Email HTML templates
│       ├── .env.sample             # Environment variable template
│       ├── .golangci.yml           # Linter configuration
│       ├── Taskfile.yml            # Task runner commands
│       ├── go.mod                  # Go module definition
│       └── go.sum                  # Go dependency checksums
│
├── packages/
│   ├── zod/                        # Zod validation schemas
│   ├── openai/                     # OpenAPI spec generator (ts-rest)
│   └── emails/                     # React Email templates
│
├── package.json                    # Root monorepo config
├── turbo.json                      # Turborepo task definitions
├── bun.lock                        # Bun lockfile
└── .gitignore
```

---

## Prerequisites

Before you begin, make sure you have the following installed on your machine:

| Tool              | Version | Purpose             | Installation                                                     |
| ----------------- | ------- | ------------------- | ---------------------------------------------------------------- |
| **Go**            | ≥ 1.21  | Backend runtime     | [go.dev/dl](https://go.dev/dl/)                                  |
| **Node.js**       | ≥ 22    | TypeScript tooling  | [nodejs.org](https://nodejs.org/)                                |
| **Bun**           | ≥ 1.3   | Package manager     | `curl -fsSL https://bun.sh/install \| bash`                      |
| **PostgreSQL**    | ≥ 15    | Primary database    | `brew install postgresql` or [pgAdmin](https://www.pgadmin.org/) |
| **Redis**         | ≥ 7     | Job queue & caching | `brew install redis`                                             |
| **Task**          | ≥ 3     | Go task runner      | `brew install go-task`                                           |
| **Docker**        | Latest  | Test containers     | [docker.com](https://www.docker.com/) (needed for tests)         |
| **golangci-lint** | Latest  | Go linting          | `brew install golangci-lint`                                     |

---

## Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/yaswanth-byk/go-boilerplate.git
cd go-boilerplate
```

### 2. Install Dependencies

Install all TypeScript dependencies (monorepo-wide):

```bash
bun install
```

Install Go dependencies:

```bash
cd apps/backend
go mod download
```

### 3. Set Up the Database

Create a PostgreSQL database named `boilerplate`:

```sql
-- Using psql or pgAdmin:
CREATE DATABASE boilerplate;
```

Or via the command line:

```bash
createdb boilerplate
```

### 4. Configure Environment Variables

Copy the sample environment file and fill in your values:

```bash
cd apps/backend
cp .env.sample .env
```

Open `.env` and configure the following critical values:

```env
# Required — your environment
BOILERPLATE_PRIMARY.ENV="local"

# Server
BOILERPLATE_SERVER.PORT="8080"

# Database — update host/port/password for your setup
BOILERPLATE_DATABASE.HOST="localhost"
BOILERPLATE_DATABASE.PORT="5432"
BOILERPLATE_DATABASE.USER="postgres"
BOILERPLATE_DATABASE.PASSWORD="your_password_here"
BOILERPLATE_DATABASE.NAME="boilerplate"

# Authentication — your Clerk secret key
BOILERPLATE_AUTH.SECRET_KEY="sk_test_..."

# Email — your Resend API key
BOILERPLATE_INTEGRATION.RESEND_API_KEY="re_..."

# Redis
BOILERPLATE_REDIS.ADDRESS="redis://localhost:6379"

# Observability — your New Relic license key (optional)
BOILERPLATE_OBSERVABILITY.NEW_RELIC.LICENSE_KEY="your_nr_key"
```

> **Note:** When `BOILERPLATE_PRIMARY.ENV` is set to `"local"`, database migrations do **not** run automatically on startup. You must run them manually (see [Database Migrations](#database-migrations)).

### 5. Run the Application

Start the backend:

```bash
cd apps/backend
task run
```

Start TypeScript packages in dev mode (for schema/email development):

```bash
# From the project root
bun run dev
```

The server will start at `http://localhost:8080`. You can verify it's running:

```bash
curl http://localhost:8080/status
```

You should see:

```json
{
  "status": "healthy",
  "timestamp": "2026-03-06T...",
  "environment": "local",
  "checks": {
    "database": { "status": "healthy", "response_time": "..." },
    "redis": { "status": "healthy", "response_time": "..." }
  }
}
```

Visit `http://localhost:8080/docs` for the interactive OpenAPI documentation.

---

## Backend Deep Dive

### Entry Point

**File:** `apps/backend/cmd/go-boilerplate/main.go`

The `main()` function orchestrates the entire application startup in this order:

1. **Load configuration** — reads environment variables with the `BOILERPLATE_` prefix
2. **Initialize logger** — sets up Zerolog with optional New Relic log forwarding
3. **Run migrations** — automatically applies database migrations (skipped in `local` environment)
4. **Create server** — initializes database pool, Redis client, and job queue
5. **Wire dependencies** — creates repositories → services → handlers (dependency injection)
6. **Start HTTP server** — begins listening for requests
7. **Graceful shutdown** — waits for `SIGINT` then shuts down with a 30-second timeout

```go
cfg, _ := config.LoadConfig()
loggerService := logger.NewLoggerService(cfg.Observability)
log := logger.NewLoggerWithService(cfg.Observability, loggerService)

srv, _ := server.New(cfg, &log, loggerService)
repos := repository.NewRepositories(srv)
services, _ := service.NewServices(srv, repos)
handlers := handler.NewHandlers(srv, services)
r := router.NewRouter(srv, handlers, services)

srv.SetupHTTPServer(r)
srv.Start()
```

### Configuration System

**Files:** `internal/config/config.go`, `internal/config/observability.go`

Configuration is loaded from environment variables using [koanf](https://github.com/knadh/koanf). Every variable is prefixed with `BOILERPLATE_` and mapped to a nested Go struct using dot notation.

**How it works:**

1. `.env` file is automatically loaded via `godotenv/autoload`
2. All `BOILERPLATE_*` env vars are read, the prefix is stripped, and they're lowercased
3. The flat key-value pairs are unmarshalled into a typed `Config` struct
4. The struct is validated using `go-playground/validator`
5. Observability config gets sensible defaults if not explicitly provided

**Config structure:**

| Section         | Env Prefix                    | Purpose                                                 |
| --------------- | ----------------------------- | ------------------------------------------------------- |
| `Primary`       | `BOILERPLATE_PRIMARY.*`       | Environment name (`local`, `development`, `production`) |
| `Server`        | `BOILERPLATE_SERVER.*`        | Port, timeouts, CORS origins                            |
| `Database`      | `BOILERPLATE_DATABASE.*`      | PostgreSQL connection parameters                        |
| `Auth`          | `BOILERPLATE_AUTH.*`          | Clerk secret key                                        |
| `Redis`         | `BOILERPLATE_REDIS.*`         | Redis connection address                                |
| `Integration`   | `BOILERPLATE_INTEGRATION.*`   | Third-party API keys (Resend)                           |
| `Observability` | `BOILERPLATE_OBSERVABILITY.*` | Logging, New Relic, health checks                       |

### Server Initialization

**File:** `internal/server/server.go`

The `Server` struct is the central hub that holds all shared dependencies:

```go
type Server struct {
    Config        *config.Config
    Logger        *zerolog.Logger
    LoggerService *loggerPkg.LoggerService
    DB            *database.Database
    Redis         *redis.Client
    httpServer    *http.Server
    Job           *job.JobService
}
```

During initialization (`server.New()`):

1. **PostgreSQL** connection pool is created with New Relic instrumentation
2. **Redis** client is created with New Relic hooks (non-fatal if Redis is unavailable)
3. **Job service** (Asynq) is started with priority queues

Shutdown is graceful: the HTTP server stops accepting connections, the database pool closes, and the job server drains its queue.

### Routing & Middleware Pipeline

**Files:** `internal/router/router.go`, `internal/router/system.go`

Routes are defined using the [Echo](https://echo.labstack.com/) framework. Every request passes through a middleware stack in this exact order:

```
Request → Rate Limiter → CORS → Security Headers → Request ID
        → New Relic Tracing → Enhanced Tracing → Context Enrichment
        → Request Logger → Panic Recovery → Handler
```

| #   | Middleware               | What It Does                                                                                                                              |
| --- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Rate Limiter**         | Limits to 20 requests/second per client. Returns `429 Too Many Requests` when exceeded.                                                   |
| 2   | **CORS**                 | Allows cross-origin requests from configured origins (`BOILERPLATE_SERVER.CORS_ALLOWED_ORIGINS`).                                         |
| 3   | **Secure**               | Adds security headers (X-Frame-Options, X-Content-Type-Options, etc.).                                                                    |
| 4   | **Request ID**           | Generates a unique UUID for each request (or uses `X-Request-ID` header if provided). Stored in context and returned in response headers. |
| 5   | **New Relic Middleware** | Starts a New Relic transaction for the request.                                                                                           |
| 6   | **Enhanced Tracing**     | Adds custom attributes (IP, user agent, request ID, user ID) to the New Relic transaction.                                                |
| 7   | **Context Enrichment**   | Creates a per-request logger enriched with request ID, method, path, IP, trace context, and user info.                                    |
| 8   | **Request Logger**       | Logs every request with status, latency, method, URI, and IP. Uses appropriate log level (Info for 2xx, Warn for 4xx, Error for 5xx).     |
| 9   | **Panic Recovery**       | Catches panics and converts them to 500 errors instead of crashing the server.                                                            |

**Currently registered routes:**

| Method | Path        | Handler                         | Description                                 |
| ------ | ----------- | ------------------------------- | ------------------------------------------- |
| `GET`  | `/status`   | `HealthHandler.CheckHealth`     | Health check (DB + Redis)                   |
| `GET`  | `/docs`     | `OpenAPIHandler.ServeOpenAPIUI` | Interactive OpenAPI documentation           |
| `GET`  | `/static/*` | Static file server              | Serves static assets (OpenAPI JSON)         |
| —      | `/api/v1`   | (route group)                   | Versioned API routes — add your routes here |

### Handler System (Go Generics)

**File:** `internal/handler/base.go`

This is one of the most powerful parts of the boilerplate. Instead of writing repetitive handler code, you use **Go generics** to get automatic:

- ✅ Request binding (JSON, query params, path params)
- ✅ Request validation
- ✅ Error handling with structured responses
- ✅ Logging (start, validation, execution, completion)
- ✅ New Relic tracing attributes
- ✅ Duration metrics for each phase

**Three handler types:**

```go
// Returns JSON response
Handle[Req, Res](handler, handlerFunc, statusCode, requestStruct)

// Returns no content (204, etc.)
HandleNoContent[Req](handler, handlerFunc, statusCode, requestStruct)

// Returns file download
HandleFile[Req](handler, handlerFunc, statusCode, requestStruct, filename, contentType)
```

**Example — adding a new endpoint:**

```go
// 1. Define your request struct with validation tags
type CreateUserRequest struct {
    Name  string `json:"name" validate:"required,min=2"`
    Email string `json:"email" validate:"required,email"`
}

func (r *CreateUserRequest) Validate() error {
    return validator.New().Struct(r)
}

// 2. Define your response struct
type CreateUserResponse struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// 3. Write your handler function
func (h *UserHandler) CreateUser(c echo.Context, req *CreateUserRequest) (*CreateUserResponse, error) {
    // Your business logic here
    user, err := h.services.User.Create(req.Name, req.Email)
    if err != nil {
        return nil, err
    }
    return &CreateUserResponse{ID: user.ID, Name: user.Name, Email: user.Email}, nil
}

// 4. Register the route — Handle does everything else for you
router.POST("/api/v1/users", handler.Handle(h.Handler, h.CreateUser, http.StatusCreated, &CreateUserRequest{}))
```

That's it. The `Handle` function automatically:

- Binds the JSON body to `CreateUserRequest`
- Calls `Validate()` and returns a `400` with field errors if validation fails
- Logs "handling request" → "validation successful" → "request completed"
- Records timing for validation and handler execution separately
- Reports errors to New Relic with stack traces

### Request Validation

**File:** `internal/validation/utils.go`

The validation system uses [go-playground/validator](https://github.com/go-playground/validator) under the hood. Every request struct must implement the `Validatable` interface:

```go
type Validatable interface {
    Validate() error
}
```

The `BindAndValidate` function:

1. Binds the request body/params to the struct
2. Calls the struct's `Validate()` method
3. Returns user-friendly field-level errors:

```json
{
  "code": "BAD_REQUEST",
  "message": "Validation failed",
  "status": 400,
  "errors": [
    { "field": "name", "error": "is required" },
    { "field": "email", "error": "must be a valid email address" }
  ]
}
```

**Supported validation tags:** `required`, `min`, `max`, `oneof`, `email`, `e164`, `uuid`, `dive`, and all standard go-playground/validator tags.

### Error Handling System

**Files:** `internal/errs/http.go`, `internal/errs/type.go`, `internal/sqlerr/`

Errors are handled at three levels:

**1. Application errors** (`internal/errs/`):

```go
errs.NewBadRequestError("Invalid input", true, nil, fieldErrors, nil)
errs.NewUnauthorizedError("Unauthorized", false)
errs.NewForbiddenError("Forbidden", false)
errs.NewNotFoundError("User not found", true, nil)
errs.NewInternalServerError()
```

Every HTTP error returns a consistent JSON structure:

```json
{
  "code": "BAD_REQUEST",
  "message": "Invalid input",
  "status": 400,
  "override": true,
  "errors": [],
  "action": null
}
```

The `override` field tells the frontend whether to show the server's message directly (`true`) or use its own default message (`false`).

The optional `action` field can instruct the frontend to redirect:

```json
{
  "action": {
    "type": "redirect",
    "message": "Session expired",
    "value": "/login"
  }
}
```

**2. Database errors** (`internal/sqlerr/`):

PostgreSQL errors are automatically converted to user-friendly HTTP errors:

| PostgreSQL Error              | HTTP Response             | Example Message                               |
| ----------------------------- | ------------------------- | --------------------------------------------- |
| Unique violation (23505)      | 400 Bad Request           | "A User with this Email already exists"       |
| Foreign key violation (23503) | 400 Bad Request           | "The referenced Organization does not exist"  |
| Not null violation (23502)    | 400 Bad Request           | "The Name is required"                        |
| Check violation (23514)       | 400 Bad Request           | "The value does not meet required conditions" |
| No rows found                 | 404 Not Found             | "User not found"                              |
| All others                    | 500 Internal Server Error | "Internal Server Error"                       |

This means you **never** need to manually catch database constraint errors. Just let them bubble up, and the `sqlerr` package produces clean, user-friendly messages automatically.

**3. Global error handler** (`internal/middleware/global.go`):

The global error handler is the last line of defense. It catches all errors, including Echo errors (like route-not-found), and converts them to the standard error format. It also logs every error with full context (request ID, user ID, trace ID, stack trace).

### Database Layer

**File:** `internal/database/database.go`

The database layer uses [pgx v5](https://github.com/jackc/pgx) with a connection pool (`pgxpool`). Key features:

- **Connection pooling** with configurable max connections, idle connections, and timeouts
- **New Relic instrumentation** via `nrpgx5` — every query is automatically traced
- **Local query logging** via `pgx-zerolog` — SQL queries are printed to the console in development
- **Multi-tracer support** — both New Relic and local logging can run simultaneously
- **Password URL encoding** — special characters in passwords are handled automatically

Connection pool settings from `.env`:

```env
BOILERPLATE_DATABASE.MAX_OPEN_CONNS="25"
BOILERPLATE_DATABASE.MAX_IDLE_CONNS="25"
BOILERPLATE_DATABASE.CONN_MAX_LIFETIME="300"   # seconds
BOILERPLATE_DATABASE.CONN_MAX_IDLE_TIME="300"  # seconds
```

### Database Migrations

**File:** `internal/database/migrator.go`

Migrations use [tern](https://github.com/jackc/tern) and are embedded in the binary using Go's `embed` package. This means migration files are compiled into the Go binary — no external files needed in production.

**Creating a new migration:**

```bash
cd apps/backend
task migrations:new name=create_users_table
```

This creates a file like `internal/database/migrations/002_create_users_table.sql`:

```sql
-- Write your migrate up statements here
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

---- create above / drop below ----

-- Write your migrate down statements here
DROP TABLE IF EXISTS users;
```

**Running migrations:**

```bash
# Set the DSN first
export BOILERPLATE_DB_DSN="postgres://postgres:password@localhost:5432/boilerplate?sslmode=disable"

# Apply all pending migrations
task migrations:up
```

> **Important:** Migrations run automatically on startup in all environments **except** `local`. In local development, always run them manually.

### Authentication (Clerk)

**Files:** `internal/middleware/auth.go`, `internal/service/auth.go`

Authentication uses [Clerk](https://clerk.com) for session management. The `RequireAuth` middleware:

1. Extracts the JWT from the `Authorization` header
2. Validates it against Clerk's public keys
3. Extracts session claims (user ID, organization role, permissions)
4. Stores them in the Echo context for downstream use

**Using auth in your routes:**

```go
// Apply the auth middleware to protected routes
authGroup := router.Group("/api/v1", middlewares.Auth.RequireAuth)
authGroup.GET("/profile", handler.Handle(...))
```

**Accessing user info in handlers:**

```go
func (h *UserHandler) GetProfile(c echo.Context, req *GetProfileRequest) (*ProfileResponse, error) {
    userID := c.Get("user_id").(string)         // Clerk user ID
    role := c.Get("user_role").(string)           // Organization role
    // permissions := c.Get("permissions")        // Organization permissions
    // ...
}
```

### Background Jobs (Asynq)

**Files:** `internal/lib/job/`

Background job processing uses [Asynq](https://github.com/hibiken/asynq), a Redis-backed task queue. It supports:

- **Priority queues:** `critical` (weight 6), `default` (weight 3), `low` (weight 1)
- **Concurrency:** 10 workers by default
- **Retries:** configurable per task (default 3 retries)
- **Timeouts:** configurable per task (default 30 seconds)

**Enqueueing a job:**

```go
task, err := job.NewWelcomeEmailTask("user@example.com", "John")
if err != nil {
    return err
}
_, err = server.Job.Client.Enqueue(task)
```

**Adding a new job type:**

1. Define the task name and payload in `email_tasks.go`:

```go
const TaskPasswordReset = "email:password_reset"

type PasswordResetPayload struct {
    To        string `json:"to"`
    ResetLink string `json:"reset_link"`
}
```

2. Create a task constructor:

```go
func NewPasswordResetTask(to, resetLink string) (*asynq.Task, error) {
    payload, err := json.Marshal(PasswordResetPayload{To: to, ResetLink: resetLink})
    return asynq.NewTask(TaskPasswordReset, payload, asynq.MaxRetry(3), asynq.Queue("critical")), err
}
```

3. Add the handler in `handlers.go` and register it in `job.go`.

### Email System (Resend)

**Files:** `internal/lib/email/`

Emails are sent asynchronously using [Resend](https://resend.com) via the job queue. The flow is:

1. **Template** — HTML templates live in `template/emails/*.html` using Go's `html/template` syntax
2. **Client** — `email.Client` renders the template with data and sends via Resend API
3. **Job** — email sending is queued as a background job (non-blocking)

Template variables use Go template syntax:

```html
<p>Hi {{.UserFirstName}},</p>
<p>Thank you for joining!</p>
```

### Logging (Zerolog)

**File:** `internal/logger/logger.go`

Logging uses [Zerolog](https://github.com/rs/zerolog) with environment-aware formatting:

| Environment             | Format                                    | Output            |
| ----------------------- | ----------------------------------------- | ----------------- |
| `local` / `development` | Pretty console with colors and timestamps | Human-readable    |
| `production`            | Structured JSON                           | Machine-parseable |

Every log entry automatically includes:

- `service` — service name
- `environment` — current environment
- `timestamp` — ISO 8601 format

**Per-request logs** (added by the Context Enhancer middleware) also include:

- `request_id` — unique request identifier
- `method` — HTTP method
- `path` — request path
- `ip` — client IP
- `user_id` — authenticated user ID (if available)
- `trace.id` / `span.id` — New Relic trace context

### Observability (New Relic)

The boilerplate has deep New Relic integration across every layer:

| Layer    | Integration         | What's Tracked                                            |
| -------- | ------------------- | --------------------------------------------------------- |
| HTTP     | `nrecho-v4`         | Every request as a transaction with method, route, status |
| Database | `nrpgx5`            | Every SQL query with duration and statement               |
| Redis    | `nrredis-v9`        | Every Redis command                                       |
| Logging  | `zerologWriter`     | All structured logs forwarded to New Relic Logs           |
| Errors   | `nrpkgerrors`       | Error stack traces in transactions                        |
| Custom   | `RecordCustomEvent` | Rate limit hits, health check failures                    |

**To disable New Relic:** Leave `BOILERPLATE_OBSERVABILITY.NEW_RELIC.LICENSE_KEY` empty. All New Relic middleware becomes a no-op automatically — no code changes needed.

### Repository & Service Layers

**Files:** `internal/repository/`, `internal/service/`

The boilerplate follows a **clean architecture** pattern:

```
Handler → Service → Repository → Database
```

- **Handlers** — Parse HTTP requests, call services, return HTTP responses
- **Services** — Business logic, orchestration, external API calls
- **Repositories** — Database queries (SQL only, no business logic)

Both layers are currently scaffolded with empty structs. Add your own as you build features.

---

## TypeScript Packages

The monorepo includes three TypeScript packages managed by [Turborepo](https://turbo.build/) and [Bun](https://bun.sh/).

### packages/zod — Zod Schemas

Defines API request/response schemas using [Zod](https://zod.dev/). These schemas serve as the **single source of truth** for API contracts — they can be shared between frontend and backend.

Example: `packages/zod/src/health.ts` defines the health check response schema.

### packages/openai — OpenAPI Generation

Uses [ts-rest](https://ts-rest.com/) with `@anatine/zod-openapi` to generate an OpenAPI 3.0.2 specification from the Zod schemas. The generated spec:

- Includes security schemes (Bearer JWT + API key)
- Maps operation IDs for SDK generation
- Is exported as both `openapi.json` and `openai.json`

**Regenerate the OpenAPI spec:**

```bash
cd packages/openai
bun run build    # or: bun gen
```

This outputs `openapi.json` which is copied to `apps/backend/static/openapi.json` for serving.

### packages/emails — Email Templates

Uses [React Email](https://react.email/) to design email templates in JSX/TSX. Templates are compiled to HTML and placed in `apps/backend/template/emails/` for the Go email client to use.

---

## Testing

### Test Infrastructure (Testcontainers)

**File:** `internal/testing/container.go`

Tests use [Testcontainers](https://testcontainers.com/) to spin up a real PostgreSQL database in Docker for each test run. This means:

- ✅ **No mocks** — tests run against a real PostgreSQL instance
- ✅ **Isolation** — each test gets a fresh database with a random name
- ✅ **Migrations** — all migrations are applied automatically
- ✅ **Cleanup** — containers are terminated after tests complete

**How it works:**

```go
func TestMyFeature(t *testing.T) {
    testDB, server, cleanup := testing.SetupTest(t)
    defer cleanup()

    // testDB.Pool is a real PostgreSQL connection pool
    // server is a fully configured *server.Server
    // cleanup closes the pool and terminates the Docker container
}
```

### Test Utilities

| Utility                     | File             | Purpose                                             |
| --------------------------- | ---------------- | --------------------------------------------------- |
| `SetupTest()`               | `helpers.go`     | Sets up DB + server + cleanup in one call           |
| `WithTransaction()`         | `transaction.go` | Runs code inside a transaction (committed)          |
| `WithRollbackTransaction()` | `transaction.go` | Runs code inside a transaction (always rolled back) |
| `AssertTimestampsValid()`   | `assertions.go`  | Checks `created_at` / `updated_at` are set          |
| `AssertValidUUID()`         | `assertions.go`  | Checks a UUID is not nil                            |
| `AssertEqualExceptTime()`   | `assertions.go`  | Compares structs ignoring time fields               |
| `MustMarshalJSON()`         | `helpers.go`     | Marshals to JSON or fails the test                  |
| `Ptr[T]()`                  | `helpers.go`     | Creates a pointer to any value (generic)            |
| `ProjectRoot()`             | `helpers.go`     | Finds the project root (`go.mod` location)          |

### Running Tests

```bash
cd apps/backend

# Run all tests
task test

# Run tests with coverage report
task test:cover

# Run a specific test
go test ./internal/handler/... -run TestHealthCheck -v
```

> **Note:** Docker must be running for tests that use Testcontainers.

---

## Available Commands

### Root-Level (Turborepo)

Run from the project root (`go-boilerplate/`):

```bash
bun run build          # Build all packages
bun run dev            # Start all packages in dev mode
bun run lint           # Lint all packages
bun run lint:fix       # Lint and auto-fix
bun run typecheck      # Type-check all packages
bun run test           # Run all tests
bun run clean          # Clean all build artifacts
```

### Backend (Taskfile)

Run from `apps/backend/`:

```bash
task help              # List all commands
task run               # Start the server
task build             # Build the binary to ./bin/
task test              # Run all Go tests
task test:cover        # Run tests with HTML coverage report
task tidy              # Format code, tidy and verify modules
task lint              # Run golangci-lint
task lint:fix          # Lint with auto-fix
task migrations:new name=xxx  # Create a new migration file
task migrations:up     # Apply all pending migrations
```

---

## Adding a New Feature (Step-by-Step)

Here's a complete walkthrough for adding a new feature (e.g., a "Users" resource):

### 1. Define the Schema (TypeScript)

Add Zod schemas in `packages/zod/src/users.ts`:

```typescript
import { z } from "zod";
export const CreateUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});
export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.string(),
});
```

### 2. Create a Database Migration

```bash
cd apps/backend
task migrations:new name=create_users
```

Edit the generated SQL file.

### 3. Create the Model

Create `apps/backend/internal/model/user.go`:

```go
type User struct {
    ID        uuid.UUID `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

### 4. Create the Repository

Create `apps/backend/internal/repository/user.go` with CRUD operations using `pgx`.

### 5. Create the Service

Create `apps/backend/internal/service/user.go` with business logic, and add it to `Services` struct.

### 6. Create the Handler

Create `apps/backend/internal/handler/user.go`:

```go
type CreateUserRequest struct {
    Name  string `json:"name" validate:"required,min=2"`
    Email string `json:"email" validate:"required,email"`
}
func (r *CreateUserRequest) Validate() error {
    return validator.New().Struct(r)
}

func (h *UserHandler) Create(c echo.Context, req *CreateUserRequest) (*model.User, error) {
    return h.services.User.Create(req.Name, req.Email)
}
```

### 7. Register Routes

In `internal/router/router.go`:

```go
v1 := router.Group("/api/v1")
v1.POST("/users", handler.Handle(h.User.Handler, h.User.Create, http.StatusCreated, &CreateUserRequest{}))
v1.GET("/users/:id", handler.Handle(h.User.Handler, h.User.GetByID, http.StatusOK, &GetUserRequest{}))
```

### 8. Write Tests

Use the test infrastructure:

```go
func TestCreateUser(t *testing.T) {
    _, srv, cleanup := testing.SetupTest(t)
    defer cleanup()
    // Test with real database...
}
```

---

## Tech Stack Summary

| Category            | Technology              | Purpose                             |
| ------------------- | ----------------------- | ----------------------------------- |
| **Language**        | Go 1.21+                | Backend application                 |
| **Web Framework**   | Echo v4                 | HTTP server, routing, middleware    |
| **Database**        | PostgreSQL 15+          | Primary data store                  |
| **DB Driver**       | pgx v5 / pgxpool        | Connection pooling, query execution |
| **Migrations**      | Tern v2                 | Embedded SQL migrations             |
| **Cache / Queue**   | Redis 7+                | Caching and job queue backend       |
| **Job Queue**       | Asynq                   | Background job processing           |
| **Auth**            | Clerk SDK v2            | Authentication & session management |
| **Email**           | Resend                  | Transactional email delivery        |
| **Logging**         | Zerolog                 | Structured logging                  |
| **Observability**   | New Relic               | APM, tracing, log forwarding        |
| **Validation**      | go-playground/validator | Request validation                  |
| **Config**          | Koanf v2                | Environment variable management     |
| **Testing**         | Testcontainers          | Isolated integration tests          |
| **Linting**         | golangci-lint           | Code quality                        |
| **Monorepo**        | Turborepo + Bun         | Multi-package management            |
| **TypeScript**      | Zod + ts-rest           | API contracts & OpenAPI generation  |
| **Email Templates** | React Email             | Email template design               |

---

## License

MIT © [Yaswanth Kumar](https://github.com/yaswanth-byk)
