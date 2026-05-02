---
name: go-config
description: |
  Write godotenv-based configuration for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying config files in internal/config/.
  Covers Config struct with nested sections, godotenv loading, helper functions (getEnv, getEnvInt, getEnvBool, getEnvDuration, getEnvSlice), database URL building, and environment helpers.
  Trigger when: writing config struct, setting up environment variables, creating config loader, adding godotenv integration, defining database configuration.
---

# Go Config Skill

## Principles

- **godotenv only**: No Viper, no envconfig. Just `godotenv.Load()` + `os.Getenv()`.
- **Nested structs**: Group related config into sub-structs (App, Database, Security, Logger, RabbitMQ).
- **Helper functions**: `getEnv`, `getEnvInt`, `getEnvBool`, `getEnvDuration`, `getEnvSlice` for clean defaults.
- **Computed fields**: Build database URLs from parts in `Load()`.
- **Environment helpers**: `IsDevelopment()`, `IsProduction()` for conditional logic.

## File Structure

```
internal/config/
└── config.go    ← Config struct + Load() + helpers
```

## Config Struct

```go
package config

import (
    "fmt"
    "os"
    "strconv"
    "time"

    "github.com/joho/godotenv"
)

type Config struct {
    App      AppConfig
    Database DatabaseConfig
    Security SecurityConfig
    Logger   LoggerConfig
    RabbitMQ RabbitMQConfig
}

type AppConfig struct {
    Name        string
    Version     string
    Environment string
    Port        string
    Timezone    string
}

type DatabaseConfig struct {
    Host            string
    Port            string
    User            string
    Password        string
    DBName          string
    SSLMode         string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration
    AutoMigrate     bool
    LogLevel        string
    URL             string // Computed DSN
}

type SecurityConfig struct {
    JWTSecret       string
    S2SAPIKey       string
    EnableCORS      bool
    AllowedOrigins  []string
    EnableRateLimit bool
    RateLimitRPS    int
}

type LoggerConfig struct {
    Level      string
    Format     string
    Output     string
    FilePath   string
    MaxSize    int
    MaxBackups int
    MaxAge     int
    Compress   bool
}

type RabbitMQConfig struct {
    URL       string
    QueueName string
}
```

## Load Function

```go
func Load() (*Config, error) {
    // Load .env file (optional — no error if missing)
    _ = godotenv.Load()

    cfg := &Config{
        App: AppConfig{
            Name:        getEnv("APP_NAME", "jrku-wp-notification"),
            Version:     getEnv("APP_VERSION", "1.0.0"),
            Environment: getEnv("APP_ENV", "development"),
            Port:        getEnv("APP_PORT", "8080"),
            Timezone:    getEnv("APP_TIMEZONE", "UTC"),
        },
        Database: DatabaseConfig{
            Host:            getEnv("DB_HOST", "localhost"),
            Port:            getEnv("DB_PORT", "5432"),
            User:            getEnv("DB_USER", "postgres"),
            Password:        getEnv("DB_PASSWORD", "postgres"),
            DBName:          getEnv("DB_NAME", "jrku_notification"),
            SSLMode:         getEnv("DB_SSL_MODE", "disable"),
            MaxOpenConns:    getEnvInt("DB_MAX_OPEN_CONNS", 25),
            MaxIdleConns:    getEnvInt("DB_MAX_IDLE_CONNS", 5),
            ConnMaxLifetime: getEnvDuration("DB_CONN_MAX_LIFETIME", "1h"),
            ConnMaxIdleTime: getEnvDuration("DB_CONN_MAX_IDLE_TIME", "5m"),
            AutoMigrate:     getEnvBool("DB_AUTO_MIGRATE", true),
            LogLevel:        getEnv("DB_LOG_LEVEL", "info"),
        },
        Security: SecurityConfig{
            JWTSecret:       getEnv("JWT_SECRET", "your-secret-key"),
            S2SAPIKey:       getEnv("S2S_API_KEY", "default-api-key"),
            EnableCORS:      getEnvBool("SECURITY_ENABLE_CORS", true),
            AllowedOrigins:  getEnvSlice("SECURITY_ALLOWED_ORIGINS", []string{"*"}),
            EnableRateLimit: getEnvBool("SECURITY_ENABLE_RATE_LIMIT", false),
            RateLimitRPS:    getEnvInt("SECURITY_RATE_LIMIT_RPS", 100),
        },
        Logger: LoggerConfig{
            Level:      getEnv("LOG_LEVEL", "info"),
            Format:     getEnv("LOG_FORMAT", "json"),
            Output:     getEnv("LOG_OUTPUT", "stdout"),
            FilePath:   getEnv("LOG_FILE_PATH", "./logs/app.log"),
            MaxSize:    getEnvInt("LOG_MAX_SIZE", 100),
            MaxBackups: getEnvInt("LOG_MAX_BACKUPS", 3),
            MaxAge:     getEnvInt("LOG_MAX_AGE", 28),
            Compress:   getEnvBool("LOG_COMPRESS", true),
        },
        RabbitMQ: RabbitMQConfig{
            URL:       getEnv("RABBITMQ_URL", "amqp://guest:guest@localhost:5672/"),
            QueueName: getEnv("RABBITMQ_QUEUE", "wp_service_notification"),
        },
    }

    // Build database URL
    cfg.Database.URL = buildDatabaseURL(&cfg.Database)

    return cfg, nil
}
```

## Helper Functions

```go
func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.Atoi(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}

func getEnvBool(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.ParseBool(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}

func getEnvDuration(key string, defaultValue string) time.Duration {
    if value := os.Getenv(key); value != "" {
        if parsed, err := time.ParseDuration(value); err == nil {
            return parsed
        }
    }
    if parsed, err := time.ParseDuration(defaultValue); err == nil {
        return parsed
    }
    return 0
}

func getEnvSlice(key string, defaultValue []string) []string {
    if value := os.Getenv(key); value != "" {
        return []string{value}
    }
    return defaultValue
}
```

## Database URL Builder

```go
func buildDatabaseURL(dbCfg *DatabaseConfig) string {
    return fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=%s sslmode=%s",
        dbCfg.Host, dbCfg.Port, dbCfg.User, dbCfg.Password, dbCfg.DBName, dbCfg.SSLMode,
    )
}
```

## Environment Helpers

```go
func (c *Config) IsDevelopment() bool {
    return c.App.Environment == "development"
}

func (c *Config) IsProduction() bool {
    return c.App.Environment == "production"
}
```

## .env.example

```
# Application
APP_NAME=jrku-wp-notification
APP_VERSION=1.0.0
APP_ENV=development
APP_PORT=8080
APP_TIMEZONE=UTC

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=jrku_notification
DB_SSL_MODE=disable
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=1h
DB_CONN_MAX_IDLE_TIME=5m
DB_AUTO_MIGRATE=true
DB_LOG_LEVEL=info

# Security
JWT_SECRET=your-secret-key
S2S_API_KEY=your-api-key
SECURITY_ENABLE_CORS=true
SECURITY_ALLOWED_ORIGINS=*
SECURITY_ENABLE_RATE_LIMIT=false
SECURITY_RATE_LIMIT_RPS=100

# Logger
LOG_LEVEL=info
LOG_FORMAT=json
LOG_OUTPUT=stdout
LOG_FILE_PATH=./logs/app.log
LOG_MAX_SIZE=100
LOG_MAX_BACKUPS=3
LOG_MAX_AGE=28
LOG_COMPRESS=true

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
RABBITMQ_QUEUE=wp_service_notification
```

## Checklist

- [ ] Config struct with nested sub-structs (App, Database, Security, Logger, RabbitMQ)
- [ ] `godotenv.Load()` with silent failure (env file is optional)
- [ ] Helper functions for string, int, bool, duration, slice
- [ ] Database URL built from parts in `Load()`
- [ ] `IsDevelopment()` / `IsProduction()` helpers
- [ ] All env vars have sensible defaults
- [ ] `.env.example` file matches config struct
- [ ] S2S API key config for internal endpoints
- [ ] RabbitMQ URL and queue name config