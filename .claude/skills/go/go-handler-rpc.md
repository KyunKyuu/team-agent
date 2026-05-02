---
name: go-handler-rpc
description: |
  Write RabbitMQ RPC handler implementations for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying RPC files in internal/rpc/.
  Covers RPC server setup with amqp091-go, message routing by routing key, JSON unmarshal/marshal, error replies, and graceful shutdown.
  Trigger when: writing RPC handler, creating RabbitMQ consumer, implementing inter-service communication, setting up RPC server, handling RPC routing keys.
---

# Go RPC Handler Skill

## Principles

- **Same service layer**: RPC handlers call the SAME service layer as HTTP handlers. No business logic duplication.
- **Routing key dispatch**: Match `routing_key` to handler method via switch/map.
- **JSON protocol**: Request and response bodies are JSON. Unmarshal → call service → marshal reply.
- **Correlation ID**: Use `amqp.Delivery.CorrelationId` to match request/response.
- **Error replies**: Return `{"error": "message"}` in JSON on failure.
- **Graceful shutdown**: Close channel and connection on `Close()`.

## File Structure

```
internal/rpc/
├── server.go    ← RPCServer struct + constructor + Start + handleMessage + Close
└── handlers.go  ← Individual RPC handler methods (split if >300 lines)
```

## RPC Server Setup

```go
// internal/rpc/server.go
package rpc

import (
    "context"
    "encoding/json"
    "fmt"
    "sync"

    "github.com/sirupsen/logrus"
    amqp "github.com/rabbitmq/amqp091-go"

    "myapp/internal/domain/service"
)

type RPCServer struct {
    conn         *amqp.Connection
    channel      *amqp.Channel
    queueName    string
    notifService service.NotificationService
    prefService  service.PreferenceService
    tokenService service.DeviceTokenService
    logger       *logrus.Logger
    wg           sync.WaitGroup
}

func NewRPCServer(
    amqpURL string,
    queueName string,
    notifService service.NotificationService,
    prefService service.PreferenceService,
    tokenService service.DeviceTokenService,
    logger *logrus.Logger,
) (*RPCServer, error) {
    conn, err := amqp.Dial(amqpURL)
    if err != nil {
        return nil, fmt.Errorf("rpc: failed to connect to RabbitMQ: %w", err)
    }

    ch, err := conn.Channel()
    if err != nil {
        conn.Close()
        return nil, fmt.Errorf("rpc: failed to open channel: %w", err)
    }

    // Declare queue
    _, err = ch.QueueDeclare(
        queueName, // name
        true,      // durable
        false,      // autoDelete
        false,      // exclusive
        false,      // noWait
        nil,        // args
    )
    if err != nil {
        ch.Close()
        conn.Close()
        return nil, fmt.Errorf("rpc: failed to declare queue: %w", err)
    }

    // Set QoS
    if err := ch.Qos(10, 0, false); err != nil {
        ch.Close()
        conn.Close()
        return nil, fmt.Errorf("rpc: failed to set QoS: %w", err)
    }

    return &RPCServer{
        conn:         conn,
        channel:      ch,
        queueName:    queueName,
        notifService: notifService,
        prefService:  prefService,
        tokenService: tokenService,
        logger:       logger,
    }, nil
}
```

## Start and Message Dispatch

```go
func (s *RPCServer) Start(ctx context.Context) error {
    msgs, err := s.channel.Consume(s.queueName, "", false, false, false, false, nil)
    if err != nil {
        return fmt.Errorf("rpc: failed to register consumer: %w", err)
    }

    s.wg.Add(1)
    go func() {
        defer s.wg.Done()
        for {
            select {
            case <-ctx.Done():
                s.logger.Info("RPC server shutting down")
                return
            case d, ok := <-msgs:
                if !ok {
                    return
                }
                s.handleMessage(ctx, d)
            }
        }
    }()

    s.logger.Infof("RPC Server started — listening on %s", s.queueName)
    return nil
}

func (s *RPCServer) handleMessage(ctx context.Context, d amqp.Delivery) {
    var response interface{}
    var err error

    switch d.RoutingKey {
    case "wp_notif.sendNotif":
        response, err = s.handleSendNotif(ctx, d.Body)
    case "wp_notif.sendNotifBulk":
        response, err = s.handleSendNotifBulk(ctx, d.Body)
    case "wp_notif.saveToken":
        response, err = s.handleSaveToken(ctx, d.Body)
    case "wp_notif.deleteToken":
        response, err = s.handleDeleteToken(ctx, d.Body)
    case "wp_notif.getNotif":
        response, err = s.handleGetNotif(ctx, d.Body)
    case "wp_notif.markRead":
        response, err = s.handleMarkRead(ctx, d.Body)
    case "wp_notif.countUnread":
        response, err = s.handleCountUnread(ctx, d.Body)
    case "wp_notif.getPreferences":
        response, err = s.handleGetPreferences(ctx, d.Body)
    case "wp_notif.updatePreferences":
        response, err = s.handleUpdatePreferences(ctx, d.Body)
    default:
        s.logger.Warnf("Unknown routing key: %s", d.RoutingKey)
        response = rpcError(fmt.Sprintf("Unknown routing key: %s", d.RoutingKey))
    }

    if err != nil {
        s.logger.Errorf("RPC handler error for %s: %v", d.RoutingKey, err)
        response = rpcError(err.Error())
    }

    s.reply(ctx, d, response)
    d.Ack(false)
}
```

## Reply and Error Helpers

```go
func (s *RPCServer) reply(ctx context.Context, d amqp.Delivery, body interface{}) {
    data, err := json.Marshal(body)
    if err != nil {
        s.logger.Errorf("RPC: failed to marshal response: %v", err)
        return
    }

    err = s.channel.PublishWithContext(
        ctx,
        "",        // exchange
        d.ReplyTo, // routing key
        false,     // mandatory
        false,     // immediate
        amqp.Publishing{
            CorrelationId: d.CorrelationId,
            ContentType:   "application/json",
            Body:          data,
        },
    )
    if err != nil {
        s.logger.Errorf("RPC: failed to publish reply: %v", err)
    }
}

func rpcError(msg string) map[string]interface{} {
    return map[string]interface{}{
        "data":  nil,
        "error": msg,
    }
}

func rpcSuccess(data interface{}) map[string]interface{} {
    return map[string]interface{}{
        "data":  data,
        "error": "",
    }
}
```

## Individual RPC Handlers

```go
// internal/rpc/handlers.go
package rpc

import (
    "context"
    "encoding/json"

    "myapp/internal/dto"
)

func (s *RPCServer) handleSendNotif(ctx context.Context, body []byte) (interface{}, error) {
    var req dto.SendNotificationManifestRequest
    if err := json.Unmarshal(body, &req); err != nil {
        return nil, fmt.Errorf("invalid request body: %w", err)
    }
    if err := s.notifService.SendNotification(ctx, &req); err != nil {
        return nil, err
    }
    return rpcSuccess(nil), nil
}

func (s *RPCServer) handleGetNotif(ctx context.Context, body []byte) (interface{}, error) {
    var req dto.GetNotificationsRequest
    if err := json.Unmarshal(body, &req); err != nil {
        return nil, fmt.Errorf("invalid request body: %w", err)
    }
    // UserID comes from request body (RPC has no JWT context)
    return s.notifService.GetNotifications(ctx, req.UserID, &req)
}

func (s *RPCServer) handleMarkRead(ctx context.Context, body []byte) (interface{}, error) {
    var req struct {
        NotificationID string `json:"notification_id"`
    }
    if err := json.Unmarshal(body, &req); err != nil {
        return nil, fmt.Errorf("invalid request body: %w", err)
    }
    if err := s.notifService.MarkRead(ctx, req.NotificationID); err != nil {
        return nil, err
    }
    return rpcSuccess(nil), nil
}

func (s *RPCServer) handleCountUnread(ctx context.Context, body []byte) (interface{}, error) {
    var req struct {
        UserID string `json:"user_id"`
    }
    if err := json.Unmarshal(body, &req); err != nil {
        return nil, fmt.Errorf("invalid request body: %w", err)
    }
    count, err := s.notifService.CountUnread(ctx, req.UserID)
    if err != nil {
        return nil, err
    }
    return rpcSuccess(map[string]int64{"unread_count": count}), nil
}

func (s *RPCServer) handleSaveToken(ctx context.Context, body []byte) (interface{}, error) {
    var req struct {
        UserID      string `json:"userId"`
        DeviceToken string `json:"deviceToken"`
        Platform    string `json:"platform"`
    }
    if err := json.Unmarshal(body, &req); err != nil {
        return nil, fmt.Errorf("invalid request body: %w", err)
    }
    if err := s.tokenService.SaveToken(ctx, req.UserID, req.DeviceToken); err != nil {
        return nil, err
    }
    return rpcSuccess(nil), nil
}

func (s *RPCServer) handleGetPreferences(ctx context.Context, body []byte) (interface{}, error) {
    var req struct {
        UserID string `json:"user_id"`
    }
    if err := json.Unmarshal(body, &req); err != nil {
        return nil, fmt.Errorf("invalid request body: %w", err)
    }
    return s.prefService.GetPreferences(ctx, req.UserID)
}
```

## Graceful Shutdown

```go
func (s *RPCServer) Close() error {
    s.wg.Wait()
    if s.channel != nil {
        s.channel.Close()
    }
    if s.conn != nil {
        return s.conn.Close()
    }
    return nil
}
```

## Routing Key Convention

```
{scope_short}_{entity}.{method}

Examples (jr-web-partner):
  wp_notif.sendNotif
  wp_notif.sendNotifBulk
  wp_notif.saveToken
  wp_notif.deleteToken
  wp_notif.getNotif
  wp_notif.markRead
  wp_notif.countUnread
  wp_notif.getPreferences
  wp_notif.updatePreferences

Examples (jr-external):
  ext_notif.sendNotif
  ext_notif.sendNotifBulk
  ...
```

## Checklist

- [ ] RPCServer struct holds service interfaces (not concrete types)
- [ ] Queue declared as durable
- [ ] QoS set (prefetch count)
- [ ] Routing key dispatch via switch
- [ ] JSON unmarshal for request, marshal for reply
- [ ] CorrelationId preserved in reply
- [ ] Error replies use `rpcError()` format
- [ ] Same service layer called as HTTP handlers
- [ ] `d.Ack(false)` after processing
- [ ] Graceful shutdown with `sync.WaitGroup`
- [ ] Context propagation to all service calls
- [ ] Files under 300-500 lines each