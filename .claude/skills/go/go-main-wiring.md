---
name: go-main-wiring
description: |
  Write main.go entry point and dependency wiring for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying main.go or wiring code.
  Covers numbered initialization steps, dependency injection chain, graceful shutdown, HTTP server setup, and RabbitMQ RPC server startup.
  Trigger when: writing main.go, setting up dependency injection, wiring services/handlers/repositories, creating server entry point, implementing graceful shutdown.
---

# Go Main Wiring Skill

## Principles

- **Numbered steps**: Each initialization step has a numbered comment. Easy to trace in logs.
- **Dependency chain**: Config → Logger → DB → Repos → Services → Handlers → Routes → Server.
- **No AutoMigrate in main.go**: Database migrations run via SQL files, not GORM AutoMigrate at startup.
- **Graceful shutdown**: Listen for SIGINT/SIGTERM, shutdown HTTP and RPC servers, close DB.
- **Two servers**: HTTP (Echo) and RabbitMQ RPC. Both start in goroutines.
- **Fail-fast**: `os.Exit(1)` if critical dependencies fail.

## File Structure

```
├── main.go                       ← Entry point + wiring + server start
├── internal/database/database.go ← GORM init (separate, called from main)
└── internal/rpc/server.go        ← RPC server (separate, called from main)
```

## main.go

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/labstack/echo/v4"
    "gorm.io/gorm"

    "myapp/internal/config"
    "myapp/internal/database"
    "myapp/internal/handler"
    "myapp/internal/repository"
    "myapp/internal/routes"
    rpcServer "myapp/internal/rpc"
    "myapp/internal/service"
    appLogger "myapp/pkg/logger"
)

func main() {
    // ── 1. Load configuration ───────────────────────────────────
    cfg, err := config.Load()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Failed to load configuration: %v\n", err)
        os.Exit(1)
    }

    // ── 2. Initialize logger ────────────────────────────────────
    if err := appLogger.Init(appLogger.Config{
        Level:      cfg.Logger.Level,
        Format:     cfg.Logger.Format,
        Output:     cfg.Logger.Output,
        FilePath:   cfg.Logger.FilePath,
        MaxSize:    cfg.Logger.MaxSize,
        MaxBackups: cfg.Logger.MaxBackups,
        MaxAge:     cfg.Logger.MaxAge,
        Compress:   cfg.Logger.Compress,
    }); err != nil {
        fmt.Fprintf(os.Stderr, "Failed to initialize logger: %v\n", err)
        os.Exit(1)
    }

    logger := appLogger.GetLogger()
    logger.Infof("Starting %s v%s", cfg.App.Name, cfg.App.Version)
    logger.Infof("Environment: %s", cfg.App.Environment)

    // ── 3. Initialize database ──────────────────────────────────
    db, err := database.InitAndMigrate(cfg)
    if err != nil {
        logger.Errorf("Failed to initialize database: %v", err)
        os.Exit(1)
    }

    // ── 4. Initialize repositories ──────────────────────────────
    notifRepo := repository.NewNotificationRepository(db, cfg.App.AppOrigin)
    tokenRepo := repository.NewDeviceTokenRepository(db, cfg.App.AppOrigin)
    prefRepo  := repository.NewPreferenceRepository(db, cfg.App.AppOrigin)
    healthRepo := repository.NewHealthRepository(db, cfg.App.Version)

    // ── 5. Initialize services ──────────────────────────────────
    prefService  := service.NewPreferenceService(prefRepo, logger)
    tokenService := service.NewDeviceTokenService(tokenRepo, logger)
    notifService := service.NewNotificationService(notifRepo, prefService, tokenService, logger)
    healthService := service.NewHealthService(healthRepo, logger)

    // ── 6. Initialize handlers ──────────────────────────────────
    notifHandler  := handler.NewNotificationHandler(notifService)
    prefHandler   := handler.NewPreferenceHandler(prefService)
    healthHandler := handler.NewHealthHandler(healthService)

    // ── 7. Initialize RPC server ─────────────────────────────────
    rpc, err := rpcServer.NewRPCServer(
        cfg.RabbitMQ.URL,
        cfg.RabbitMQ.QueueName,
        notifService,
        prefService,
        tokenService,
        logger,
    )
    if err != nil {
        logger.Warnf("Failed to initialize RPC server (continuing without RPC): %v", err)
        // RPC is optional — service can run with HTTP only
    }

    // ── 8. Setup HTTP server and routes ─────────────────────────
    e := echo.New()
    e.HideBanner = true
    e.HidePort = true

    routes.SetupRoutes(e, cfg, healthHandler, notifHandler, prefHandler)

    // ── 9. Start servers with graceful shutdown ─────────────────
    startServers(e, rpc, cfg, logger)
}

func startServers(e *echo.Echo, rpc *rpcServer.RPCServer, cfg *config.Config, logger *logrus.Logger) {
    // Start RPC server in background
    if rpc != nil {
        ctx, cancelRpc := context.WithCancel(context.Background())
        defer cancelRpc()

        go func() {
            if err := rpc.Start(ctx); err != nil {
                logger.Errorf("RPC server error: %v", err)
            }
        }()
    }

    // Start HTTP server in background
    addr := fmt.Sprintf(":%s", cfg.App.Port)
    srv := &http.Server{
        Addr:         addr,
        ReadTimeout:  30 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    go func() {
        appLogger.Infof("Server starting on %s", addr)
        if err := e.StartServer(srv); err != nil && err != http.ErrServerClosed {
            appLogger.Fatalf("Failed to start server: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    appLogger.Info("Shutting down servers...")

    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // Shutdown HTTP server
    if err := e.Shutdown(ctx); err != nil {
        appLogger.Errorf("HTTP server forced to shutdown: %v", err)
    }

    // Shutdown RPC server
    if rpc != nil {
        if err := rpc.Close(); err != nil {
            appLogger.Errorf("RPC server shutdown error: %v", err)
        }
    }

    // Close database connection
    if err := database.Close(); err != nil {
        appLogger.Errorf("Error closing database: %v", err)
    }

    appLogger.Info("Server exited")
}
```

## Dependency Injection Chain

```
Config
  │
  ├──→ Logger
  │
  ├──→ Database (GORM)
  │      │
  │      ├──→ Repositories (notifRepo, tokenRepo, prefRepo, healthRepo)
  │      │        │
  │      │        └──→ Services (notifService, tokenService, prefService, healthService)
  │      │                  │
  │      │                  └──→ Handlers (notifHandler, prefHandler, healthHandler)
  │      │
  │      └──→ Routes (SetupRoutes)
  │
  └──→ RPC Server (notifService, prefService, tokenService)
```

## Numbered Steps Convention

Each step in `main()` has a numbered comment:

```
1. Load configuration
2. Initialize logger
3. Initialize database
4. Initialize repositories
5. Initialize services
6. Initialize handlers
7. Initialize RPC server
8. Setup HTTP server and routes
9. Start servers with graceful shutdown
```

This convention makes it easy to:
- Trace startup sequence in logs
- Identify which step failed
- Add new steps (e.g., "4b. Initialize external clients")

## GORM Database Init (separate file)

```go
// internal/database/database.go
package database

import (
    "fmt"
    "sync"
    "time"

    "myapp/internal/config"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

var (
    db   *gorm.DB
    once sync.Once
)

func InitAndMigrate(cfg *config.Config) (*gorm.DB, error) {
    var initErr error
    once.Do(func() {
        var gormLogLevel logger.LogLevel
        switch cfg.Database.LogLevel {
        case "silent":
            gormLogLevel = logger.Silent
        case "error":
            gormLogLevel = logger.Error
        case "warn":
            gormLogLevel = logger.Warn
        default:
            gormLogLevel = logger.Info
        }

        db, initErr = gorm.Open(postgres.Open(cfg.Database.URL), &gorm.Config{
            Logger: logger.Default.LogMode(gormLogLevel),
            NowFunc: func() time.Time { return time.Now().UTC() },
        })
        if initErr != nil {
            initErr = fmt.Errorf("failed to connect to database: %w", initErr)
            return
        }

        sqlDB, err := db.DB()
        if err != nil {
            initErr = fmt.Errorf("failed to get database instance: %w", err)
            return
        }

        sqlDB.SetMaxOpenConns(cfg.Database.MaxOpenConns)
        sqlDB.SetMaxIdleConns(cfg.Database.MaxIdleConns)
        sqlDB.SetConnMaxLifetime(cfg.Database.ConnMaxLifetime)
        sqlDB.SetConnMaxIdleTime(cfg.Database.ConnMaxIdleTime)

        if err := sqlDB.Ping(); err != nil {
            initErr = fmt.Errorf("failed to ping database: %w", err)
            return
        }
    })

    return db, initErr
}

func Close() error {
    if db != nil {
        sqlDB, err := db.DB()
        if err != nil {
            return err
        }
        return sqlDB.Close()
    }
    return nil
}
```

## Checklist

- [ ] Numbered initialization steps (1-9)
- [ ] Config loaded first, logger second, database third
- [ ] Repositories initialized with `db` and `appOrigin`
- [ ] Services initialized with repo interfaces + logger
- [ ] Handlers initialized with service interfaces
- [ ] RPC server optional (service runs without it)
- [ ] No GORM AutoMigrate in main.go
- [ ] Graceful shutdown: SIGINT/SIGTERM → HTTP shutdown → RPC close → DB close
- [ ] HTTP server with timeouts (Read, Write, Idle)
- [ ] Database connection pool configured
- [ ] Database singleton via `sync.Once`