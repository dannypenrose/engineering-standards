# Real-Time Communication Standards

> Authoritative standards for WebSocket, SSE, and real-time patterns across all applications.

## Purpose

Define when and how to implement real-time communication, covering protocol selection, connection management, scaling, and reliability patterns for production systems.

## Core Principles

1. **Choose the simplest protocol** — SSE for one-way, WebSocket for bidirectional, HTTP polling as fallback
2. **Expect disconnection** — Every client must handle reconnection gracefully
3. **Authenticate at connection time** — Verify identity before upgrading to WebSocket
4. **Backpressure over flooding** — Never send faster than the client can consume
5. **Stateless servers** — Use external pub/sub for multi-instance scaling

## Protocol Selection

| Protocol | Direction | Use When | Not For |
| -------- | --------- | -------- | ------- |
| **Server-Sent Events (SSE)** | Server → Client | Notifications, live feeds, progress updates | Chat, gaming, collaborative editing |
| **WebSocket** | Bidirectional | Chat, collaborative editing, real-time dashboards | Simple notifications, file uploads |
| **Long Polling** | Simulated push | Fallback when SSE/WS not available | New applications (use SSE/WS instead) |
| **WebTransport** | Bidirectional (UDP) | Low-latency gaming, video streaming | General applications (limited support) |

### Decision Tree

```
Does the client need to send frequent messages to the server?
├── No → Use SSE (simpler, auto-reconnect, HTTP/2 multiplexing)
└── Yes
    ├── Is low latency critical (< 50ms)? → WebSocket
    ├── Is ordering guaranteed important? → WebSocket
    └── Can you tolerate some latency? → Consider SSE + REST POST hybrid
```

## Server-Sent Events (SSE)

SSE is the preferred choice for server-to-client streaming. It is simpler than WebSocket, works over standard HTTP, has built-in reconnection, and multiplexes over HTTP/2.

### Node.js / NestJS

```typescript
// NestJS SSE endpoint
@Controller('events')
export class EventsController {
  constructor(private events: EventEmitter2) {}

  @Sse('stream')
  @UseGuards(JwtAuthGuard)
  stream(@Req() req: Request): Observable<MessageEvent> {
    const userId = req.user.sub;

    return new Observable(subscriber => {
      const handler = (data: any) => {
        subscriber.next({ data: JSON.stringify(data) } as MessageEvent);
      };

      this.events.on(`user.${userId}.notification`, handler);

      // Heartbeat every 30s to keep connection alive
      const heartbeat = setInterval(() => {
        subscriber.next({ data: '', type: 'heartbeat' } as MessageEvent);
      }, 30000);

      // Cleanup on disconnect
      req.on('close', () => {
        this.events.off(`user.${userId}.notification`, handler);
        clearInterval(heartbeat);
        subscriber.complete();
      });
    });
  }
}
```

### ASP.NET Core

```csharp
[ApiController]
[Route("api/events")]
public class EventsController : ControllerBase
{
    [HttpGet("stream")]
    [Authorize]
    public async Task Stream(CancellationToken cancellationToken)
    {
        Response.Headers.Append("Content-Type", "text/event-stream");
        Response.Headers.Append("Cache-Control", "no-cache");
        Response.Headers.Append("Connection", "keep-alive");

        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        while (!cancellationToken.IsCancellationRequested)
        {
            var notification = await _notificationService
                .WaitForNotificationAsync(userId, cancellationToken);

            if (notification is not null)
            {
                var json = JsonSerializer.Serialize(notification);
                await Response.WriteAsync($"data: {json}\n\n", cancellationToken);
                await Response.Body.FlushAsync(cancellationToken);
            }
        }
    }
}
```

### Client-Side SSE

```typescript
function createEventSource(url: string, token: string) {
  const eventSource = new EventSource(url, {
    headers: { Authorization: `Bearer ${token}` },
  });

  eventSource.onmessage = (event) => {
    if (event.data === '') return; // Heartbeat
    const data = JSON.parse(event.data);
    handleNotification(data);
  };

  eventSource.onerror = () => {
    // Browser auto-reconnects with Last-Event-ID header
    console.warn('SSE connection lost, reconnecting...');
  };

  return eventSource;
}
```

## WebSocket

### Connection Lifecycle

```
Client                                 Server
  │                                      │
  │──── HTTP Upgrade Request ──────────▶│
  │     (with auth token)                │
  │                                      │── Validate token
  │◀─── 101 Switching Protocols ────────│
  │                                      │
  │◀───────── Heartbeat (ping) ─────────│  every 30s
  │──────────  Pong  ──────────────────▶│
  │                                      │
  │◀──────── Message ───────────────────│
  │──────── Message ───────────────────▶│
  │                                      │
  │──── Close Frame ───────────────────▶│
  │◀─── Close Frame ───────────────────│
```

### Node.js (Socket.IO)

```typescript
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

// Server setup with Redis adapter for scaling
const io = new Server(httpServer, {
  cors: { origin: process.env.FRONTEND_URL },
  pingInterval: 25000,
  pingTimeout: 20000,
  maxHttpBufferSize: 1e6, // 1 MB max message size
});

// Redis adapter for multi-instance scaling
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

// Authentication middleware
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const user = await verifyJwt(token);
    socket.data.userId = user.sub;
    next();
  } catch {
    next(new Error('Authentication failed'));
  }
});

// Connection handling
io.on('connection', (socket) => {
  const userId = socket.data.userId;

  // Join user-specific room
  socket.join(`user:${userId}`);

  // Handle messages
  socket.on('chat:message', async (data, callback) => {
    try {
      const message = await chatService.createMessage(userId, data);
      io.to(`room:${data.roomId}`).emit('chat:message', message);
      callback({ status: 'ok' });
    } catch (error) {
      callback({ status: 'error', message: error.message });
    }
  });

  // Room management
  socket.on('room:join', (roomId) => {
    socket.join(`room:${roomId}`);
  });

  socket.on('disconnect', (reason) => {
    logger.info('Client disconnected', { userId, reason });
  });
});
```

### ASP.NET Core (SignalR)

```csharp
// Hub
[Authorize]
public class ChatHub : Hub
{
    public async Task SendMessage(string roomId, string content)
    {
        var userId = Context.User!.FindFirst(ClaimTypes.NameIdentifier)!.Value;
        var message = await _chatService.CreateMessage(userId, roomId, content);
        await Clients.Group(roomId).SendAsync("ReceiveMessage", message);
    }

    public async Task JoinRoom(string roomId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomId);
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        await _presenceService.SetOffline(Context.User!.Identity!.Name!);
        await base.OnDisconnectedAsync(exception);
    }
}

// Program.cs
builder.Services.AddSignalR()
    .AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis")!);
```

### Client-Side WebSocket

```typescript
import { io, Socket } from 'socket.io-client';

class RealtimeClient {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;

  connect(token: string) {
    this.socket = io(process.env.NEXT_PUBLIC_WS_URL!, {
      auth: { token },
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 30000,
      reconnectionAttempts: this.maxReconnectAttempts,
      timeout: 20000,
    });

    this.socket.on('connect', () => {
      this.reconnectAttempts = 0;
      console.log('Connected to real-time server');
    });

    this.socket.on('disconnect', (reason) => {
      if (reason === 'io server disconnect') {
        // Server disconnected — may need to re-authenticate
        this.socket?.connect();
      }
    });

    this.socket.on('connect_error', (error) => {
      this.reconnectAttempts++;
      if (error.message === 'Authentication failed') {
        this.socket?.disconnect();
        // Trigger token refresh flow
      }
    });
  }

  disconnect() {
    this.socket?.disconnect();
    this.socket = null;
  }
}
```

## Message Format

### Standard Message Envelope

```typescript
interface RealtimeMessage {
  type: string;           // Event type (e.g., "chat.message", "user.typing")
  data: unknown;          // Event payload
  id?: string;            // Unique message ID (for deduplication)
  timestamp: string;      // ISO 8601 timestamp
  correlationId?: string; // For request-response patterns
}
```

### Message Size Limits

| Context | Max Size | Enforcement |
| ------- | -------- | ----------- |
| WebSocket message | 1 MB | Server config (`maxHttpBufferSize`) |
| SSE event | 64 KB | Application-level check |
| Binary data | Not over WebSocket | Use presigned URL + REST upload |

## Scaling

### Multi-Instance Architecture

```
                    Load Balancer (sticky sessions OR Redis adapter)
                    ┌───────────────────────────────────────────────┐
                    │                                               │
              ┌─────▼─────┐   ┌─────────────┐   ┌─────────────┐
              │  Server 1  │   │  Server 2    │   │  Server 3   │
              │  (WS/SSE)  │   │  (WS/SSE)   │   │  (WS/SSE)  │
              └─────┬──────┘   └──────┬───────┘   └──────┬──────┘
                    │                 │                    │
                    └────────┬────────┘                    │
                             │                             │
                    ┌────────▼─────────────────────────────▼──┐
                    │              Redis Pub/Sub                │
                    └──────────────────────────────────────────┘
```

**Requirements for multi-instance:**
- **Redis adapter** (Socket.IO) or **Redis backplane** (SignalR) for cross-server messaging
- **Sticky sessions** at the load balancer if Redis adapter is not used
- Connection state stored in Redis, not in-memory

## Rate Limiting

```typescript
// Per-connection rate limiting
const RATE_LIMITS = {
  'chat:message': { maxPerMinute: 60, maxPerSecond: 5 },
  'chat:typing':  { maxPerMinute: 30, maxPerSecond: 2 },
  'room:join':    { maxPerMinute: 10, maxPerSecond: 1 },
};

function rateLimitMiddleware(socket: Socket, event: string): boolean {
  const limit = RATE_LIMITS[event];
  if (!limit) return true;

  const key = `ratelimit:${socket.data.userId}:${event}`;
  const count = incrementCounter(key, 60);

  if (count > limit.maxPerMinute) {
    socket.emit('error', { code: 'RATE_LIMITED', message: 'Too many requests' });
    return false;
  }
  return true;
}
```

## Error Handling

### Graceful Degradation

```typescript
// React hook with SSE fallback
function useRealtimeNotifications(userId: string) {
  const [notifications, setNotifications] = useState<Notification[]>([]);

  useEffect(() => {
    // Try WebSocket first
    const socket = io(WS_URL, { auth: { token } });

    socket.on('connect_error', () => {
      // Fall back to SSE
      const eventSource = new EventSource(`${API_URL}/events/stream`);
      eventSource.onmessage = (e) => {
        setNotifications(prev => [JSON.parse(e.data), ...prev]);
      };

      return () => eventSource.close();
    });

    socket.on('notification', (data) => {
      setNotifications(prev => [data, ...prev]);
    });

    return () => socket.disconnect();
  }, [userId]);

  return notifications;
}
```

## Checklist

### Implementation

- [ ] Protocol selection justified (SSE vs WebSocket vs polling)
- [ ] Authentication verified before connection upgrade
- [ ] Heartbeat/ping-pong configured (25-30s interval)
- [ ] Reconnection with exponential backoff implemented
- [ ] Message size limits enforced
- [ ] Rate limiting per connection configured

### Scaling

- [ ] Redis pub/sub adapter configured for multi-instance
- [ ] Connection state externalised (not in-memory)
- [ ] Load balancer configured (sticky sessions or Redis adapter)
- [ ] Graceful shutdown drains existing connections

### Reliability

- [ ] Client handles disconnection and reconnection
- [ ] Fallback to SSE or polling if WebSocket unavailable
- [ ] Message deduplication (idempotent handlers)
- [ ] Monitoring: connection count, message throughput, error rate

## References

- [MDN — Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [MDN — WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
- [Socket.IO Documentation](https://socket.io/docs/v4/)
- [ASP.NET Core SignalR](https://learn.microsoft.com/en-us/aspnet/core/signalr/)
- [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
