# Patterns for `csharp-rules`

Section numbers match the rules in `SKILL.md`.

## Project settings (rules 1, 2)

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
  <WarningsAsErrors>nullable;CS8509;CS8524</WarningsAsErrors>
</PropertyGroup>
```

## 1. Result without a library

The private constructor closes the hierarchy, so `default` or a `null` error cannot produce a fake success.

```csharp
public sealed record Error(string Code, string Message);

public abstract record Result<T>
{
    private Result() { }

    public sealed record Ok(T Value) : Result<T>;
    public sealed record Err(Error Error) : Result<T>;

    public static implicit operator Result<T>(T value) => new Ok(value);
    public static implicit operator Result<T>(Error error) => new Err(error);
}
```

Producer:

```csharp
public async Task<Result<User>> GetUserAsync(UserId id, CancellationToken ct)
{
    if (id.Value <= 0)
        return new Error("user.invalid_id", $"Invalid id {id.Value}");

    User? user = await _db.Users.AsNoTracking()
        .FirstOrDefaultAsync(u => u.Id == id, ct);

    return user is null
        ? new Error("user.not_found", $"User {id.Value} not found")
        : user;
}
```

Consumer handling both cases:

```csharp
string message = await GetUserAsync(id, ct) switch
{
    Result<User>.Ok(var user) => $"Hello {user.Name}",
    Result<User>.Err(var error) => $"Error {error.Code}: {error.Message}",
    _ => throw new UnreachableException()
};
```

Propagation (early return plays the role of Rust's `?`):

```csharp
public async Task<Result<Order>> PlaceOrderAsync(UserId id, Cart cart, CancellationToken ct)
{
    Result<User> userResult = await GetUserAsync(id, ct);
    if (userResult is not Result<User>.Ok(var user))
        return ((Result<User>.Err)userResult).Error;

    return Order.Create(user, cart);
}
```

When a project already uses a library, follow its idiom instead (`Match`, `Bind`, `ErrorOr.Then`, `OneOf.Switch`).

## 2. Closed domain unions

```csharp
public abstract record PaymentResult
{
    private PaymentResult() { }

    public sealed record Succeeded(TransactionId Id, Money Amount) : PaymentResult;
    public sealed record Failed(ErrorCode Code, string Reason) : PaymentResult;
    public sealed record Pending(PaymentId Id) : PaymentResult;
}

string Describe(PaymentResult result) => result switch
{
    PaymentResult.Succeeded s => $"Paid {s.Amount}",
    PaymentResult.Failed f => $"Failed: {f.Reason}",
    PaymentResult.Pending p => $"Pending {p.Id}",
    _ => throw new UnreachableException()
};
```

## 3. Immutability

```csharp
// Prefer
public sealed record Order(OrderId Id, ImmutableArray<OrderLine> Lines, Money Total);

// Over
public class Order
{
    public List<OrderLine> Lines { get; set; } = new();
}
```

Lookup tables built once at startup:

```csharp
private static readonly FrozenDictionary<string, Currency> Currencies =
    LoadCurrencies().ToFrozenDictionary(c => c.Code, StringComparer.OrdinalIgnoreCase);
```

## 4. Ownership of disposables

```csharp
public sealed class Connection : IAsyncDisposable
{
    public static async Task<Connection> OpenAsync(Uri endpoint, CancellationToken ct) { /* ... */ }
    public ValueTask DisposeAsync() { /* ... */ }
}

// The caller owns the connection and disposes it at end of scope.
await using Connection conn = await Connection.OpenAsync(endpoint, ct);
```

Ownership transfer is explicit:

```csharp
/// <summary>Takes ownership of <paramref name="stream"/> and disposes it.</summary>
public static PacketReader FromOwnedStream(Stream stream) => new(stream, leaveOpen: false);
```

## 6. Concurrency

Bounded channel with a completed writer:

```csharp
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

Task producer = Task.Run(async () =>
{
    try
    {
        await foreach (WorkItem item in source.ReadAllAsync(ct))
            await channel.Writer.WriteAsync(item, ct);
    }
    finally
    {
        channel.Writer.Complete();
    }
}, ct);

Task consumer = Task.Run(async () =>
{
    await foreach (WorkItem item in channel.Reader.ReadAllAsync(ct))
        await ProcessAsync(item, ct);
}, ct);

await Task.WhenAll(producer, consumer);
```

Lock owned by the type that owns the data (`System.Threading.Lock` is .NET 9+; use `private readonly object` before that):

```csharp
public sealed class Counter
{
    private readonly Lock _gate = new();
    private int _value;

    public int Increment()
    {
        lock (_gate) return ++_value;
    }
}
```

## 7. Async

Timeout without losing the caller's cancellation:

```csharp
using var timeout = CancellationTokenSource.CreateLinkedTokenSource(ct);
timeout.CancelAfter(TimeSpan.FromSeconds(5));
Response response = await _client.SendAsync(request, timeout.Token);
```

`ValueTask` on a cache-backed hot path:

```csharp
public ValueTask<int> GetCountAsync(CancellationToken ct) =>
    _cache.TryGetValue(out int count)
        ? ValueTask.FromResult(count)
        : new ValueTask<int>(LoadCountAsync(ct));
```

Avoid:

```csharp
User user = GetUserAsync(id, ct).Result;          // deadlock / thread-pool starvation
async void Save() { await _repo.SaveAsync(); }    // exceptions crash the process
await Task.Run(() => _client.GetAsync(url));      // pointless thread hop for I/O
```

## 8. Flat, allocation-conscious data (Goose-inspired)

Index handles stay valid when the backing storage grows, and the data stays contiguous.

```csharp
// Pointer-heavy: one allocation per node, scattered in memory
public sealed class Node
{
    public Node? Parent;
    public List<Node> Children = new();
    public string Name = "";
}

// Flat: parallel arrays addressed by handle
public readonly record struct NodeId(int Index);

public sealed class Tree
{
    private readonly List<NodeId> _parents = [];
    private readonly List<string> _names = [];

    public NodeId Add(string name, NodeId parent)
    {
        _parents.Add(parent);
        _names.Add(name);
        return new NodeId(_names.Count - 1);
    }

    public string NameOf(NodeId id) => _names[id.Index];
    public NodeId ParentOf(NodeId id) => _parents[id.Index];
}
```

Pooled and stack buffers:

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try
{
    int read = await stream.ReadAsync(buffer.AsMemory(0, 4096), ct);
    Parse(buffer.AsSpan(0, read));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}

Span<byte> scratch = length <= 256 ? stackalloc byte[length] : new byte[length];
```

Fixed-size inline data (C# 12):

```csharp
[InlineArray(8)]
public struct EightFloats
{
    private float _element0;
}
```

## 9. Simplicity checks

Before adding an abstraction, ask:
- Can this be a plain function or record?
- Does the interface have a second implementation or a test that needs it?
- Will a reader follow the control flow in one pass?
