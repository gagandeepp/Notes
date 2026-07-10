# SignalR — Complete Guide (Basic → Expert)
### Server: ASP.NET Core (C#) · Client: React (JavaScript/TypeScript)

---

## Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Project Setup](#2-project-setup)
3. [Hubs 101](#3-hubs-101)
4. [Connection Lifecycle](#4-connection-lifecycle)
5. [Groups, Users & Targeted Messaging](#5-groups-users--targeted-messaging)
6. [React Client Patterns](#6-react-client-patterns)
7. [Streaming](#7-streaming)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Scaling Out](#9-scaling-out)
10. [Performance Tuning](#10-performance-tuning)
11. [Security Hardening](#11-security-hardening)
12. [Testing Hubs](#12-testing-hubs)
13. [Advanced Patterns](#13-advanced-patterns)
14. [Error Handling & Resilience](#14-error-handling--resilience)
15. [Tricky Interview Questions & Answers](#15-tricky-interview-questions--answers)

---

## 1. Fundamentals

### What is SignalR?

ASP.NET Core SignalR is a library that abstracts real-time, bidirectional communication between server and client over a persistent connection. Instead of the client polling the server, the server can push data to clients the instant something happens (chat messages, live dashboards, notifications, collaborative editing cursors, etc.).

### Transport negotiation

SignalR doesn't force a single transport — it negotiates the best one available and falls back gracefully:

| Transport | How it works | When used |
|---|---|---|
| WebSockets | Full-duplex, single TCP connection | Default/preferred, if supported by client+server+proxies |
| Server-Sent Events (SSE) | Server→client stream over HTTP; client→server uses separate HTTP POST | Fallback when WebSockets unavailable (browser support), one-directional stream |
| Long Polling | Client repeatedly issues HTTP requests, server holds them open until data or timeout | Last resort fallback, highest latency/overhead |

The handshake happens during the `/negotiate` HTTP request, where the server tells the client which transport(s) it can use.

### The wire protocol

On top of the transport sits a **protocol** that defines how messages are framed and serialized:
- **JSON** (default, human-readable, `Microsoft.AspNetCore.SignalR` built-in)
- **MessagePack** (binary, smaller payloads, faster — requires `Microsoft.AspNetCore.SignalR.Protocols.MessagePack`)

### SignalR vs alternatives

| | SignalR | Raw WebSockets | gRPC streaming | Socket.IO |
|---|---|---|---|---|
| Fallback transports | Yes (auto) | No (you build it) | No | Yes |
| RPC-style method calls | Yes (`Clients.Caller.SendAsync(...)`) | No (manual framing) | Yes (protobuf-defined) | Yes |
| Groups/rooms | Built-in | Manual | Manual | Built-in |
| Browser support | Excellent (JS client) | Native | Needs grpc-web proxy | Excellent |
| Best for | .NET ecosystems needing RPC-style push | Full control, custom protocols | Cross-language typed streaming | Node ecosystems |

---

## 2. Project Setup

### Server (ASP.NET Core)

```bash
dotnet new webapi -n SignalRDemo.Api
cd SignalRDemo.Api
```

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSignalR(); // registers hub infrastructure

builder.Services.AddCors(options =>
{
    options.AddPolicy("ReactClient", policy =>
    {
        policy.WithOrigins("http://localhost:5173") // Vite dev server
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials(); // required for SignalR (cookies/auth)
    });
});

var app = builder.Build();

app.UseCors("ReactClient");
app.UseAuthentication();
app.UseAuthorization();

app.MapHub<ChatHub>("/hubs/chat");

app.Run();
```

> **Gotcha:** `AllowAnyOrigin()` + `AllowCredentials()` together throw a runtime exception. You must whitelist explicit origins when credentials are involved.

### Client (React)

```bash
npm install @microsoft/signalr
```

---

## 3. Hubs 101

A **Hub** is the central abstraction — a class the server and client use to call methods on each other.

```csharp
public class ChatHub : Hub
{
    // Called BY the client, runs ON the server
    public async Task SendMessage(string user, string message)
    {
        // Push to ALL connected clients (including sender)
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
}
```

### The `Clients` property — who receives the push

```csharp
Clients.All.SendAsync(...);              // everyone connected
Clients.Caller.SendAsync(...);           // only the invoker
Clients.Others.SendAsync(...);           // everyone except the invoker
Clients.Client(connectionId).SendAsync(...); // one specific connection
Clients.Clients(connectionIdList).SendAsync(...); // specific list
Clients.Group("room1").SendAsync(...);   // a named group
Clients.GroupExcept("room1", excludedIds).SendAsync(...);
Clients.User(userId).SendAsync(...);     // all connections for a given user
```

### Strongly-typed hubs (recommended for production)

Untyped hubs use magic strings (`"ReceiveMessage"`) which are error-prone. `Hub<T>` gives you a compiler-checked interface:

```csharp
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserTyping(string user);
}

public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.ReceiveMessage(user, message); // no string, no typo risk
    }
}
```

---

## 4. Connection Lifecycle

```csharp
public class ChatHub : Hub<IChatClient>
{
    private readonly IConnectionMapping _connections;

    public ChatHub(IConnectionMapping connections) => _connections = connections;

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;
        _connections.Add(userId, Context.ConnectionId);
        await Clients.Others.ReceiveMessage("System", $"{userId} joined");
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;
        _connections.Remove(userId, Context.ConnectionId);
        await Clients.Others.ReceiveMessage("System", $"{userId} left");
        await base.OnDisconnectedAsync(exception);
    }
}
```

Key `Context` properties:

| Property | Purpose |
|---|---|
| `Context.ConnectionId` | Unique per physical connection (changes on reconnect!) |
| `Context.UserIdentifier` | Stable identity across reconnects (from claims, via `IUserIdProvider`) |
| `Context.User` | `ClaimsPrincipal` if authenticated |
| `Context.Items` | Per-connection dictionary for storing custom state |
| `Context.ConnectionAborted` | `CancellationToken` fired on disconnect — vital for streaming |

> **Gotcha:** One logical *user* can have multiple `ConnectionId`s (multiple browser tabs, multiple devices). Never key business state off `ConnectionId` alone if you care about the user across tabs — use `UserIdentifier`.

---

## 5. Groups, Users & Targeted Messaging

```csharp
public class ChatHub : Hub<IChatClient>
{
    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).ReceiveMessage("System", $"joined {roomName}");
    }

    public async Task LeaveRoom(string roomName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomName);
    }
}
```

> **Gotcha:** Group membership is **not persisted**. It lives in the backplane/in-memory store tied to the connection. If a client reconnects (even automatically), it gets a *new* `ConnectionId` and must rejoin all groups — this is a very common production bug. Always rejoin groups in your client's `onreconnected` handler.

### Per-user messaging with `IUserIdProvider`

By default `UserIdentifier` maps to the `ClaimTypes.NameIdentifier` claim. You can customize it:

```csharp
public class CustomUserIdProvider : IUserIdProvider
{
    public string? GetUserId(HubConnectionContext connection)
        => connection.User?.FindFirst("sub")?.Value;
}

// Program.cs
builder.Services.AddSingleton<IUserIdProvider, CustomUserIdProvider>();
```

---

## 6. React Client Patterns

### Basic connection

```jsx
import { useEffect, useRef, useState } from "react";
import * as signalR from "@microsoft/signalr";

function ChatWindow({ authToken }) {
  const connectionRef = useRef(null);
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    const connection = new signalR.HubConnectionBuilder()
      .withUrl("https://localhost:7000/hubs/chat", {
        accessTokenFactory: () => authToken, // JWT for auth
      })
      .withAutomaticReconnect([0, 2000, 5000, 10000, null]) // retry intervals, null = stop
      .configureLogging(signalR.LogLevel.Information)
      .build();

    connection.on("ReceiveMessage", (user, message) => {
      setMessages((prev) => [...prev, { user, message }]);
    });

    connection.onreconnecting((error) => {
      console.warn("Reconnecting...", error);
    });

    connection.onreconnected(async () => {
      // MUST rejoin groups after reconnect — new ConnectionId!
      await connection.invoke("JoinRoom", "general");
    });

    connection.onclose((error) => {
      console.error("Connection closed permanently", error);
    });

    connection
      .start()
      .then(() => connection.invoke("JoinRoom", "general"))
      .catch((err) => console.error("Connection failed:", err));

    connectionRef.current = connection;

    return () => {
      connection.stop(); // cleanup on unmount — avoids leaked connections
    };
  }, [authToken]);

  const sendMessage = async (user, text) => {
    try {
      await connectionRef.current.invoke("SendMessage", user, text);
    } catch (err) {
      console.error(err);
    }
  };

  return (
    <div>
      {messages.map((m, i) => (
        <div key={i}>
          <b>{m.user}:</b> {m.message}
        </div>
      ))}
    </div>
  );
}
```

### Custom hook (reusable across components)

```jsx
import { useEffect, useState, useCallback, useRef } from "react";
import * as signalR from "@microsoft/signalr";

function useSignalR(hubUrl, accessTokenFactory) {
  const [connectionState, setConnectionState] = useState("Disconnected");
  const connectionRef = useRef(null);

  useEffect(() => {
    const connection = new signalR.HubConnectionBuilder()
      .withUrl(hubUrl, { accessTokenFactory })
      .withAutomaticReconnect()
      .build();

    connection.onreconnecting(() => setConnectionState("Reconnecting"));
    connection.onreconnected(() => setConnectionState("Connected"));
    connection.onclose(() => setConnectionState("Disconnected"));

    connection
      .start()
      .then(() => setConnectionState("Connected"))
      .catch(() => setConnectionState("Failed"));

    connectionRef.current = connection;
    return () => connection.stop();
  }, [hubUrl]);

  const invoke = useCallback((method, ...args) => {
    if (connectionRef.current?.state === signalR.HubConnectionState.Connected) {
      return connectionRef.current.invoke(method, ...args);
    }
    return Promise.reject(new Error("Not connected"));
  }, []);

  const on = useCallback((event, handler) => {
    connectionRef.current?.on(event, handler);
    return () => connectionRef.current?.off(event, handler);
  }, []);

  return { connectionState, invoke, on };
}
```

> **Gotcha:** In React 18 Strict Mode, effects run twice in development. This can start two connections and then immediately tear one down — usually harmless because of the cleanup function, but it's a common source of "why is my connection restarting?" confusion during dev. Production builds don't double-invoke.

---

## 7. Streaming

### Server → Client streaming (`IAsyncEnumerable`)

```csharp
public class DataHub : Hub
{
    public async IAsyncEnumerable<int> CounterStream(
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        for (int i = 0; i < 100; i++)
        {
            // Honor cancellation — fires if client calls stream.dispose()
            // or disconnects
            cancellationToken.ThrowIfCancellationRequested();
            await Task.Delay(500, cancellationToken);
            yield return i;
        }
    }
}
```

React client consuming the stream:

```jsx
const stream = connection.stream("CounterStream");

const subscription = stream.subscribe({
  next: (value) => console.log("Received:", value),
  complete: () => console.log("Stream complete"),
  error: (err) => console.error("Stream error:", err),
});

// To cancel early:
// subscription.dispose();
```

### Client → Server streaming (`ChannelReader`)

```csharp
public class UploadHub : Hub
{
    public async Task UploadStream(ChannelReader<string> stream)
    {
        await foreach (var chunk in stream.ReadAllAsync())
        {
            // process each chunk as it arrives
            Console.WriteLine($"Chunk: {chunk}");
        }
    }
}
```

```jsx
const subject = new signalR.Subject();
connection.send("UploadStream", subject);

subject.next("chunk1");
subject.next("chunk2");
subject.complete();
```

---

## 8. Authentication & Authorization

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
        };

        // CRITICAL for SignalR: browsers cannot set Authorization headers
        // on WebSocket handshakes, so the token must come via query string
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;

                if (!string.IsNullOrEmpty(accessToken) &&
                    path.StartsWithSegments("/hubs"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });
```

```csharp
[Authorize]
public class ChatHub : Hub<IChatClient>
{
    [Authorize(Roles = "Admin")]
    public async Task DeleteMessage(string messageId) { /* ... */ }
}
```

> **Gotcha:** This query-string token pattern means access tokens end up in server logs and browser history by default unless you're careful — treat these as short-lived tokens, and consider log scrubbing.

---

## 9. Scaling Out

### The problem

A single SignalR server keeps all connections + group memberships in memory. The moment you run **2+ server instances behind a load balancer**, `Clients.All.SendAsync(...)` on server A will *not* reach clients connected to server B.

### Solution 1: Redis backplane

```bash
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis("localhost:6379", options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("MyApp");
    });
```

Every server publishes outgoing messages to Redis pub/sub; every server subscribes and forwards to its own locally-connected clients. This solves fan-out but:

- Redis pub/sub adds latency (extra network hop)
- If Redis goes down, cross-server messaging breaks (though same-server messaging still works)
- Group/user membership state is still tracked per-server, but broadcast is coordinated via Redis

### Solution 2: Azure SignalR Service (fully managed)

```bash
dotnet add package Microsoft.Azure.SignalR
```

```csharp
builder.Services.AddSignalR()
    .AddAzureSignalR(connectionString);
```

This completely changes the architecture: clients connect to the **Azure SignalR Service**, not directly to your app servers. Your servers become simple message producers behind the scenes. This eliminates the need for sticky sessions and scales connection count independently of app server count (important because a single ASP.NET Core server has a practical WebSocket connection ceiling).

### Sticky sessions (if NOT using a backplane)

If you scale without a backplane, you **must** configure your load balancer for sticky sessions (session affinity) — otherwise a client's long-polling/reconnect requests may hit a different server than the one holding its connection state, breaking everything. This is a band-aid, not a real solution — always prefer a backplane or Azure SignalR Service for real horizontal scaling.

| Approach | Sticky sessions needed? | Extra latency | Ops complexity |
|---|---|---|---|
| No backplane, single server | N/A | None | Lowest, but no scale |
| No backplane, multi-server | Required | None | Fragile |
| Redis backplane | Not required | Redis round-trip | Medium (manage Redis) |
| Azure SignalR Service | Not required | Managed by Azure | Lowest for scaling |

---

## 10. Performance Tuning

### Use MessagePack instead of JSON

```bash
dotnet add package Microsoft.AspNetCore.SignalR.Protocols.MessagePack
```

```csharp
builder.Services.AddSignalR()
    .AddMessagePackProtocol();
```

```jsx
import * as signalR from "@microsoft/signalr";
import * as signalRMsgPack from "@microsoft/signalr-protocol-msgpack";

const connection = new signalR.HubConnectionBuilder()
  .withUrl("/hubs/chat")
  .withHubProtocol(new signalRMsgPack.MessagePackHubProtocol())
  .build();
```

MessagePack is a binary format — noticeably smaller payloads and faster de/serialization than JSON, especially valuable for high-frequency small messages (e.g., game state, cursor positions).

### Tune keep-alive & timeout intervals

```csharp
builder.Services.AddSignalR(options =>
{
    options.KeepAliveInterval = TimeSpan.FromSeconds(15); // server ping interval
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30); // must be >= 2x KeepAlive
    options.HandshakeTimeout = TimeSpan.FromSeconds(15);
    options.MaximumReceiveMessageSize = 32 * 1024; // 32 KB, guard against huge payloads
    options.StreamBufferCapacity = 10; // backpressure control for streaming
});
```

> **Gotcha:** `ClientTimeoutInterval` must be at least double `KeepAliveInterval`, or the client will falsely detect a dead connection and disconnect. This is called out in Microsoft's docs but frequently misconfigured.

### Backpressure in streaming

`StreamBufferCapacity` limits how many un-consumed items the server buffers when the client is a slow consumer — without this, a fast producer + slow consumer combination can balloon server memory.

### Batching

For very high-frequency updates (e.g., 60 fps cursor positions), batch multiple updates into a single `SendAsync` call on an interval (e.g., every 50ms) rather than sending one message per event — this drastically cuts protocol/transport overhead.

### Avoid large hub method payloads

Large arguments get fully buffered before deserialization; prefer streaming (`IAsyncEnumerable`) for large or unbounded datasets instead of one giant method call.

---

## 11. Security Hardening

### Lock down CORS explicitly

Never use `AllowAnyOrigin()` for anything beyond local prototyping. Combine with `AllowCredentials()` only for named origins (see Section 2).

### Validate everything server-side

Hub methods are effectively public RPC endpoints reachable by any authenticated (or unauthenticated, if you forgot `[Authorize]`) client. Never trust client-supplied group names, user IDs, or content without validation:

```csharp
public async Task JoinRoom(string roomName)
{
    if (!await _roomService.UserCanJoinAsync(Context.User, roomName))
    {
        throw new HubException("Not authorized to join this room.");
    }
    await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
}
```

### Rate limiting

SignalR hub invocations bypass typical ASP.NET Core middleware rate limiting unless you apply it explicitly. Use a custom `IHubFilter` (see Section 13) to throttle invocations per connection:

```csharp
public class RateLimitFilter : IHubFilter
{
    private static readonly ConcurrentDictionary<string, (int Count, DateTime WindowStart)> _hits = new();

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var key = invocationContext.Context.ConnectionId;
        var now = DateTime.UtcNow;

        var entry = _hits.AddOrUpdate(key,
            (1, now),
            (_, existing) => now - existing.WindowStart > TimeSpan.FromSeconds(1)
                ? (1, now)
                : (existing.Count + 1, existing.WindowStart));

        if (entry.Count > 20) // max 20 calls/sec per connection
        {
            throw new HubException("Rate limit exceeded.");
        }

        return await next(invocationContext);
    }
}

// Program.cs
builder.Services.AddSignalR(options =>
{
    options.AddFilter<RateLimitFilter>();
});
```

### Enforce `wss://` (TLS) in production

Never allow plain `ws://` outside local development — the handshake token and all message payloads are otherwise plaintext on the wire.

### Guard connection count / DoS surface

Each open connection consumes server memory and (without a backplane/Azure SignalR) counts against your practical connection ceiling. Consider connection limits per user/IP and aggressive timeouts for idle/unauthenticated connections.

### Don't leak internal exceptions to clients

By default, unhandled exceptions in hub methods return a generic error to the client — but if `EnableDetailedErrors` is turned on (useful for dev), stack traces leak to clients. Never enable it in production:

```csharp
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = false; // default in production; be explicit
});
```

---

## 12. Testing Hubs

Hub methods are just C# methods — but they depend on `Hub` base-class members (`Clients`, `Context`, `Groups`) that need mocking.

### Unit testing a hub method

```csharp
public class ChatHubTests
{
    [Fact]
    public async Task SendMessage_BroadcastsToAllClients()
    {
        // Arrange
        var hub = new ChatHub();

        var mockClients = new Mock<IHubCallerClients<IChatClient>>();
        var mockClientProxy = new Mock<IChatClient>();
        mockClients.Setup(c => c.All).Returns(mockClientProxy.Object);
        hub.Clients = mockClients.Object;

        var mockContext = new Mock<HubCallerContext>();
        mockContext.Setup(c => c.ConnectionId).Returns("conn-123");
        hub.Context = mockContext.Object;

        // Act
        await hub.SendMessage("alice", "hello");

        // Assert
        mockClientProxy.Verify(
            c => c.ReceiveMessage("alice", "hello"),
            Times.Once);
    }
}
```

### Integration testing with `TestServer` + a real client

```csharp
public class ChatHubIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ChatHubIntegrationTests(WebApplicationFactory<Program> factory)
        => _factory = factory;

    [Fact]
    public async Task Client_Receives_BroadcastMessage()
    {
        var client = _factory.CreateClient();

        var connection = new HubConnectionBuilder()
            .WithUrl("http://localhost/hubs/chat", options =>
            {
                options.HttpMessageHandlerFactory = _ => _factory.Server.CreateHandler();
            })
            .Build();

        string? receivedUser = null;
        string? receivedMessage = null;
        var tcs = new TaskCompletionSource();

        connection.On<string, string>("ReceiveMessage", (user, msg) =>
        {
            receivedUser = user;
            receivedMessage = msg;
            tcs.SetResult();
        });

        await connection.StartAsync();
        await connection.InvokeAsync("SendMessage", "bob", "hi there");
        await tcs.Task.WaitAsync(TimeSpan.FromSeconds(5));

        Assert.Equal("bob", receivedUser);
        Assert.Equal("hi there", receivedMessage);
    }
}
```

> **Gotcha:** Testing streaming and groups is harder because `Groups.AddToGroupAsync` in the real implementation talks to the connection-tracking layer. For pure unit tests, mock `IGroupManager` too; for real group-broadcast behavior, prefer the `TestServer` integration approach.

---

## 13. Advanced Patterns

### `IHubFilter` — cross-cutting concerns (logging, validation, rate limiting)

```csharp
public class LoggingFilter : IHubFilter
{
    private readonly ILogger<LoggingFilter> _logger;
    public LoggingFilter(ILogger<LoggingFilter> logger) => _logger = logger;

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        _logger.LogInformation("Invoking {Method}", invocationContext.HubMethodName);
        try
        {
            return await next(invocationContext);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error in {Method}", invocationContext.HubMethodName);
            throw;
        }
    }

    public Task OnConnectedAsync(HubLifetimeContext context, Func<HubLifetimeContext, Task> next)
        => next(context);

    public Task OnDisconnectedAsync(
        HubLifetimeContext context, Exception? exception, Func<HubLifetimeContext, Exception?, Task> next)
        => next(context, exception);
}
```

### Calling hub clients from OUTSIDE a hub (`IHubContext<T>`)

Very common need: pushing a notification from a background service, a controller, or a message-queue consumer — none of which are inside the hub itself.

```csharp
public class OrderController : ControllerBase
{
    private readonly IHubContext<ChatHub, IChatClient> _hubContext;

    public OrderController(IHubContext<ChatHub, IChatClient> hubContext)
        => _hubContext = hubContext;

    [HttpPost("orders")]
    public async Task<IActionResult> CreateOrder(OrderDto order)
    {
        // ... save order ...
        await _hubContext.Clients.User(order.UserId).ReceiveMessage(
            "System", $"Order {order.Id} confirmed!");
        return Ok();
    }
}
```

> **Gotcha:** `IHubContext<T>` gives you `Clients` but **not** `Context` or `Groups` in the same way — you can't call `Groups.AddToGroupAsync` on an arbitrary connection ID from outside a hub method directly via `IHubContext` in older versions; you send to `Clients.Group(...)` (which works) but managing group membership from outside typically still requires the hub itself to have called `AddToGroupAsync` at some point via a live connection.

### Custom hub protocol

For niche binary formats (e.g., Protobuf) beyond JSON/MessagePack, you implement `IHubProtocol` and register it — rare, but comes up in interviews about SignalR's extensibility model.

### Dependency injection scope inside hubs

Hubs are **transient** by default — a new instance is created per method invocation (not per connection!). This means:
- Don't store per-connection state as instance fields on the hub — it won't survive between calls.
- Use `Context.Items` (a per-connection dictionary) or an external singleton service (e.g., an in-memory `ConcurrentDictionary` keyed by `ConnectionId`) for state that must persist across calls.

```csharp
public class ChatHub : Hub
{
    // WRONG — this resets on every single method call, it does NOT persist per connection
    private int _messageCount = 0;

    public override Task OnConnectedAsync()
    {
        // RIGHT — persists for the life of the connection
        Context.Items["messageCount"] = 0;
        return base.OnConnectedAsync();
    }
}
```

---

## 14. Error Handling & Resilience

### Client-side automatic reconnect

```jsx
const connection = new signalR.HubConnectionBuilder()
  .withUrl("/hubs/chat")
  .withAutomaticReconnect({
    nextRetryDelayInMilliseconds: (retryContext) => {
      if (retryContext.previousRetryCount === 0) return 0;
      if (retryContext.elapsedMilliseconds < 60000) {
        return Math.random() * 10000; // jittered backoff
      }
      return null; // stop retrying after 60s
    },
  })
  .build();
```

### `HubException` — the only exception whose message reaches the client

```csharp
public async Task SendMessage(string user, string message)
{
    if (string.IsNullOrWhiteSpace(message))
    {
        throw new HubException("Message cannot be empty."); // message text IS sent to client
    }
    // any OTHER exception type: client just gets a generic
    // "An unexpected error occurred invoking ... on the server."
    // unless EnableDetailedErrors is true (dev only!)
}
```

```jsx
try {
  await connection.invoke("SendMessage", "alice", "");
} catch (err) {
  console.error(err.message); // "Message cannot be empty."
}
```

### Global reconnection state UI pattern

```jsx
connection.onreconnecting(() => setStatus("reconnecting"));
connection.onreconnected(() => setStatus("connected"));
connection.onclose(() => setStatus("disconnected")); // terminal — user must refresh / manually restart
```

---

## 15. Tricky Interview Questions & Answers

**Q1: Does SignalR guarantee message delivery or ordering?**
A: No. SignalR is fire-and-forget by default — there's no built-in acknowledgment, retry, or persistence for missed messages. If the connection drops between `SendAsync` and the client receiving it, that message is lost forever unless you build your own ack/replay layer (e.g., message IDs + a "catch-up" query on reconnect). Ordering across *multiple* connections/groups is also not guaranteed when a backplane is involved, since Redis pub/sub doesn't guarantee cross-channel ordering.

**Q2: Why does `Clients.All.SendAsync(...)` sometimes not reach all clients after scaling to multiple servers?**
A: Because each server only knows about connections made directly to it. Without a backplane (Redis) or Azure SignalR Service, "All" only means "all clients connected to *this* instance." This is one of the most common production surprises when an app is scaled out without adjusting SignalR's architecture.

**Q3: If a client has automatic reconnect enabled and reconnects, does it automatically rejoin the groups it was in?**
A: No. Reconnecting establishes a brand-new `ConnectionId`, and group membership was tied to the old connection ID. You must explicitly rejoin groups in the `onreconnected` callback — this is one of the most-cited "gotchas" in real SignalR codebases.

**Q4: Are Hub instances singletons?**
A: No — a new Hub instance is created per method invocation via DI (`transient` lifetime), not per connection and not shared across calls. Never store per-connection mutable state as instance fields; use `Context.Items` or an external store keyed by `ConnectionId`/`UserIdentifier`.

**Q5: What's the difference between `Context.ConnectionId` and `Context.UserIdentifier`?**
A: `ConnectionId` is unique per physical connection and changes every time the client reconnects (even automatically). `UserIdentifier` is derived from claims (via `IUserIdProvider`) and stays stable across reconnects and even across multiple simultaneous connections (e.g., two open tabs) for the same logical user.

**Q6: How would you push a message to a specific user from a plain MVC controller that has no access to the Hub instance?**
A: Inject `IHubContext<THub, TClient>` via DI (it's registered automatically by `AddSignalR()`), then call `hubContext.Clients.User(userId).SomeMethod(...)`. This lets any part of the app (controllers, background services, queue consumers) push notifications without needing an active Hub method invocation.

**Q7: Why must `ClientTimeoutInterval` be at least double `KeepAliveInterval`?**
A: `KeepAliveInterval` controls how often the server sends a ping to prove the connection is alive. `ClientTimeoutInterval` is how long the *client* waits without hearing from the server before assuming the connection died. If the timeout is shorter than (or too close to) 2x the keep-alive period, normal network jitter causes clients to falsely declare the connection dead and disconnect, even though the server is fine.

**Q8: Can you use `[Authorize]` at both the Hub class level and individual method level? What happens?**
A: Yes — `[Authorize]` on the class applies to all methods; you can further restrict individual methods with a more specific policy/role (e.g., `[Authorize(Roles = "Admin")]` on one method inside a hub whose class-level attribute just requires any authenticated user). The method-level attribute is additive/more restrictive, not a replacement.

**Q9: Why can't you just put the JWT in an `Authorization: Bearer` header for SignalR like a normal API call?**
A: Because browsers don't allow custom headers to be set on the WebSocket handshake request. SignalR's JS client works around this by sending the token as an `access_token` query string parameter for WebSocket/SSE connections, which the server-side JWT middleware must be explicitly configured to read (via `OnMessageReceived`).

**Q10: What happens if two servers behind a load balancer both have SignalR but no backplane, and no sticky sessions?**
A: Chaos. A client's initial negotiate request might hit server A, get a connection ID, then a subsequent long-polling or reconnect request gets routed by the load balancer to server B — which has no idea about that connection. This manifests as random disconnects, missed messages, or handshake failures. This is why sticky sessions are mandatory *unless* you have a backplane or use Azure SignalR Service (which removes the problem architecturally).

**Q11: What's the practical difference between using a Redis backplane vs. Azure SignalR Service for scaling?**
A: A Redis backplane keeps clients connected directly to your app servers; Redis just relays messages between server instances so broadcasts reach everyone, but you still need sticky sessions removed via the backplane's connection tracking and you still bear the WebSocket connection load on your own servers. Azure SignalR Service is a fully separate managed service that clients connect to directly — your app servers become thin message producers, so your own compute doesn't bear the connection-count ceiling, and you don't need a backplane or sticky sessions at all.

**Q12: If a Hub method throws a generic `Exception` (not `HubException`), what does the client see?**
A: A sanitized, generic error message (something like "An unexpected error occurred invoking 'MethodName' on the server."), NOT the real exception message or stack trace — unless `EnableDetailedErrors = true`, which should never be enabled in production since it leaks internal details to any connected client. Only `HubException` messages are passed through to the client verbatim by design.

**Q13: How do you cancel a long-running server-to-client stream when the client stops listening?**
A: Accept a `[EnumeratorCancellation] CancellationToken` parameter in the `IAsyncEnumerable` method signature. SignalR automatically triggers cancellation on that token when the client calls `.dispose()` on its stream subscription or disconnects — you must check/honor the token (e.g., `cancellationToken.ThrowIfCancellationRequested()` in the loop) or the server keeps producing values into the void.

**Q14: What is `StreamBufferCapacity` and why does it matter?**
A: It limits how many stream items the server will buffer server-side if the client is consuming slower than the server is producing. Without a cap, a fast producer + slow consumer scenario can cause unbounded memory growth on the server. It's SignalR's built-in backpressure mechanism for streaming.

**Q15: Why would you choose MessagePack over JSON, and what's the tradeoff?**
A: MessagePack is a binary serialization format — smaller payload size and faster serialization/deserialization than JSON, which matters at high message frequency or with many concurrent connections. The tradeoff is debuggability: you can't just read raw MessagePack bytes in browser dev tools like you can with JSON, making manual debugging/tracing harder.

**Q16: Can a single logical user have multiple SignalR connections simultaneously, and how do you send to "that user" everywhere at once?**
A: Yes — e.g., the same user with two browser tabs, or a phone + laptop both open. `Clients.User(userId).SendAsync(...)` sends to *all* connections currently mapped to that `UserIdentifier`, not just one — this relies on the `IUserIdProvider` correctly and consistently resolving the same user ID across all their connections.

**Q17: Why is `AllowAnyOrigin()` combined with `AllowCredentials()` a runtime error rather than just insecure?**
A: Per the CORS spec, wildcard origins cannot be paired with credentialed requests (cookies, auth headers) because that combination would allow any site on the internet to make authenticated requests on behalf of a logged-in user. ASP.NET Core enforces this at runtime by throwing rather than silently allowing a security hole — you must explicitly list allowed origins when using `AllowCredentials()`.

**Q18: What's a common cause of "it works locally but breaks behind our corporate proxy/load balancer" for SignalR?**
A: Many corporate/enterprise proxies don't support WebSocket upgrade requests properly, or strip the required headers — causing SignalR to fall back to Server-Sent Events or Long Polling. If the app was only ever tested with WebSockets locally, missing sticky-session configuration (needed for the fallback transports' repeated HTTP requests) or proxy buffering settings can cause connection failures or extreme latency in production that never appeared in dev.

**Q19: Why might you implement an `IHubFilter` instead of putting logging/validation code directly into every Hub method?**
A: `IHubFilter` acts like middleware specifically for Hub method invocations and lifecycle events (connect/disconnect) — it centralizes cross-cutting concerns (logging, rate limiting, validation, metrics) in one place instead of duplicating boilerplate in every single Hub method, and keeps hub methods focused purely on business logic.

**Q20: True or false: calling `Groups.AddToGroupAsync` persists across server restarts.**
A: False. Group membership (like all connection state) is entirely in-memory (or backplane-coordinated in-memory across servers) — it is not durable. A server restart, or the underlying connection dropping, wipes it. If you need "durable room membership" (e.g., for a chat app where users should auto-rejoin rooms after a server deploy), you must persist the *logical* membership yourself (e.g., in a database) and re-add the connection to groups in `OnConnectedAsync` by looking up that persisted state.

**Q21: Why does `Clients.Group("room1").SendAsync(...)` skip a client that appears to be in the room in your UI?**
A: Almost always because that client reconnected (new `ConnectionId`) after a network blip and never re-invoked `JoinRoom` — so server-side group membership was silently dropped for that new connection while the UI still shows the user as "in" the room from stale client-side state.

**Q22: What happens to in-flight Hub method calls if the connection drops mid-call?**
A: The server-side method continues executing to completion (or until it hits a cancellation token check, if the method is written to observe `Context.ConnectionAborted`) — but any attempt to send a response back to that now-dead connection is simply dropped. Long-running hub methods that don't check `Context.ConnectionAborted` can keep doing unnecessary work after the client is long gone; well-written streaming/long methods should observe it.

**Q23: How would you unit test a Hub method that calls `Clients.Group(...)`?**
A: Mock `IHubCallerClients<T>` so that `.Group("roomName")` returns a mocked `IChatClient`/`IClientProxy`, then verify the expected method was called with expected arguments via the mocking framework (e.g., Moq's `.Verify(...)`) — you don't need a real SignalR pipeline or transport for a pure unit test; save the real pipeline for integration tests via `WebApplicationFactory` + `HubConnectionBuilder`.

**Q24: Why is rate limiting on Hub methods not automatically covered by ASP.NET Core's built-in rate-limiting middleware?**
A: ASP.NET Core's rate-limiting middleware operates on the HTTP request pipeline. Once a SignalR connection is upgraded (e.g., to WebSockets), subsequent Hub method invocations travel over that persistent connection as SignalR protocol messages, not as discrete new HTTP requests — so standard request-based middleware never sees them. You need a SignalR-specific mechanism, like a custom `IHubFilter`, to rate-limit invocations per connection.

**Q25: What's the risk of leaving `EnableDetailedErrors = true` in production?**
A: It causes the server to send the full exception message and (depending on version/config) stack trace details back to the invoking client for any unhandled exception in a Hub method — potentially leaking internal implementation details, database error text, file paths, or other information useful to an attacker. It should only ever be enabled temporarily in local development.

---

## Quick Reference Cheat Sheet

```csharp
// Server-side essentials
Clients.All / Caller / Others / Client(id) / Clients(ids) / Group(name) / User(userId)
Groups.AddToGroupAsync(connId, group) / RemoveFromGroupAsync(connId, group)
Context.ConnectionId / Context.UserIdentifier / Context.User / Context.Items
IHubContext<THub, TClient>  // for pushing from outside a hub
HubException                 // only exception type whose message reaches the client
IAsyncEnumerable<T>           // server -> client streaming
ChannelReader<T>              // client -> server streaming
IHubFilter                    // middleware-like cross-cutting concerns
```

```jsx
// Client-side essentials
new signalR.HubConnectionBuilder().withUrl(...).withAutomaticReconnect().build()
connection.on(event, handler) / connection.off(event, handler)
connection.invoke(method, ...args)   // request-response
connection.send(method, ...args)     // fire-and-forget
connection.stream(method, ...args).subscribe({...})
connection.onreconnecting / onreconnected / onclose
```
