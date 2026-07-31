# Data Model: RSS Feed Subscription Management MVP

**Date**: 2026-07-31 | **Branch**: 001-subscription-management | **Phase**: 1 (Design & Contracts)

This document defines the data structures, validation rules, relationships, and storage model for the RSS Feed Reader MVP.

---

## Core Entity: Subscription

The `Subscription` entity represents a single RSS/Atom feed subscription added by the user.

### Definition

```csharp
public class Subscription
{
    /// <summary>
    /// Unique identifier for the subscription within the current session.
    /// Generated as a GUID (globally unique identifier) when the subscription is created.
    /// Format: UUID v4 string (e.g., "550e8400-e29b-41d4-a716-446655440000")
    /// Required: Yes
    /// Mutable: No (immutable after creation)
    /// </summary>
    public string Id { get; set; }

    /// <summary>
    /// The feed URL as provided by the user.
    /// Stored as-is (no parsing, validation, or normalization in MVP).
    /// Examples: "https://devblogs.microsoft.com/dotnet/feed/", "https://example.com/feed.xml"
    /// Required: Yes
    /// Length: No arbitrary limit enforced (supports URLs up to 2000+ characters)
    /// Mutable: No (immutable after creation)
    /// Validation: Non-empty string required; whitespace trimmed before storage
    /// </summary>
    public string Url { get; set; }

    /// <summary>
    /// Timestamp indicating when the subscription was added (creation time).
    /// Format: ISO 8601 UTC (e.g., "2026-07-31T16:17:16Z")
    /// Precision: Milliseconds
    /// Timezone: Always UTC (no local time)
    /// Required: Yes
    /// Mutable: No (set to current time when created; never updated)
    /// </summary>
    public DateTime AddedAt { get; set; }
}
```

### Property Details

| Property | Type | Required | Mutable | Default | Constraints | Notes |
|----------|------|----------|---------|---------|-------------|-------|
| `Id` | string (UUID) | Yes | No | Guid.NewGuid().ToString() | Unique within session; non-empty | Generated at creation time |
| `Url` | string | Yes | No | N/A | Non-empty after trim; max 2048 chars | User-provided; stored as-is |
| `AddedAt` | DateTime | Yes | No | DateTime.UtcNow | UTC timezone only | Set at creation; never modified |

### Example Instance

```csharp
new Subscription
{
    Id = "a1b2c3d4-e5f6-47a8-b9c0-d1e2f3a4b5c6",
    Url = "https://devblogs.microsoft.com/dotnet/feed/",
    AddedAt = new DateTime(2026, 7, 31, 16, 17, 16, DateTimeKind.Utc)
}
```

---

## Validation Rules

### URL Validation (Input Level)

**Requirement**: Ensure URL is a valid, non-empty string before accepting from user input.

**Rule 1: Non-Empty**
- **Condition**: `url` parameter must not be null, empty string, or whitespace-only
- **Action**: Trim leading/trailing whitespace before validation
- **Error Response (if violated)**: HTTP 400 Bad Request
  ```json
  {
    "error": "Invalid request",
    "message": "URL is required and must not be empty"
  }
  ```
- **Test Case**: `AddSubscription("")` → should reject; `AddSubscription("  ")` → should reject

**Rule 2: Maximum Length (Future Consideration)**
- **Condition** (MVP): No explicit length limit enforced; system accepts URLs up to 2048 characters
- **Condition** (Extended-MVP): Consider adding max 2048 character limit to prevent buffer overflow attacks
- **Action**: Reject if URL exceeds limit
- **Error Response**: HTTP 400 Bad Request with message "URL exceeds maximum length"

**Rule 3: No Format Validation (MVP Scope)**
- **Condition**: MVP does not validate URL format (no URI parsing, no scheme validation, no domain validation)
- **Action**: Accept any non-empty string as a URL; assume user provides valid feed URL
- **Rationale**: MVP speed priority; format validation deferred to Extended-MVP
- **Example**: Both valid URL and malformed input accepted (e.g., "https://example.com/feed" and "not a url" both stored)

### Subscription Entity Validation (Storage Level)

**Rule 4: ID Uniqueness**
- **Condition**: Each subscription must have a unique `Id` within the current session
- **Implementation**: Automatically enforced by GUID generation (extremely low collision probability)
- **Verification**: Check `List<Subscription>` does not contain duplicate IDs (should never happen, but testable)

**Rule 5: Timestamp Format**
- **Condition**: `AddedAt` must be UTC DateTime (no local time)
- **Implementation**: Use `DateTime.UtcNow` at creation; serialize as ISO 8601 format for API responses
- **Example Response**: `"addedAt": "2026-07-31T16:17:16.123Z"` (Z indicates UTC)

**Rule 6: Immutability**
- **Condition**: Once created, `Id`, `Url`, and `AddedAt` must not be modified
- **Implementation**: No public setters; properties are read-only after construction
- **Rationale**: Prevents data corruption; audit trail remains accurate (creation timestamp stays fixed)

### Data Integrity Constraints

| Constraint | Scope | Enforcement Level | Description |
|-----------|-------|------------------|-------------|
| Unique ID | Session-wide | Automatic (GUID) | No duplicate subscription IDs |
| Non-empty URL | Per subscription | Input validation | Reject empty/whitespace-only URLs |
| UTC timestamp | Per subscription | Automatic (UtcNow) | Always UTC; never local time |
| Immutability | Per subscription | Code design | No setter access after creation |
| Order preservation | Session-wide | List ordering | Subscriptions returned in FIFO order (add order) |

---

## Storage Model: In-Memory (MVP)

### Implementation: Session-Scoped List

**Storage Structure**:
```csharp
public class InMemorySubscriptionService : ISubscriptionService
{
    // Session-scoped storage: List<Subscription>
    private readonly List<Subscription> _subscriptions = new();

    public async Task<Subscription> AddSubscriptionAsync(string url)
    {
        // 1. Validate URL (trim, check non-empty)
        if (string.IsNullOrWhiteSpace(url))
            throw new InvalidOperationException("URL is required");

        url = url.Trim();

        // 2. Create subscription with GUID, current UTC time
        var subscription = new Subscription
        {
            Id = Guid.NewGuid().ToString(),
            Url = url,
            AddedAt = DateTime.UtcNow
        };

        // 3. Add to in-memory list
        _subscriptions.Add(subscription);

        return subscription;
    }

    public async Task<List<Subscription>> GetSubscriptionsAsync()
    {
        // Return shallow copy to prevent external modification
        return new List<Subscription>(_subscriptions);
    }
}
```

### Characteristics

| Aspect | Behavior | Rationale |
|--------|----------|-----------|
| **Lifetime** | Application instance lifetime | Single-user, local development; data lost on restart |
| **Scope** | Application-wide (singleton) | All users/sessions share single list (MVP single-user assumption) |
| **Performance** | O(1) add, O(n) list retrieval | No database overhead; suitable for <1000 subscriptions |
| **Memory footprint** | ~500 bytes per subscription (metadata) + URL string length | Example: 1000 subscriptions × 500 bytes + avg 50-byte URL = ~55 KB |
| **Concurrency** | Not thread-safe (MVP) | Single-user development environment; no multi-threaded access expected |
| **Data loss** | Complete data loss on app restart | Expected behavior for MVP; users accept session-scoped storage |

### Future Evolution: Database Backend

**Design for Extended-MVP**: Service layer abstraction enables swapping storage implementation.

```csharp
// ISubscriptionService interface (used by controllers)
public interface ISubscriptionService
{
    Task<Subscription> AddSubscriptionAsync(string url);
    Task<List<Subscription>> GetSubscriptionsAsync();
}

// MVP implementation: InMemorySubscriptionService (uses List<Subscription>)
// Extended-MVP implementation: EfCoreSubscriptionService (uses EF Core + SQLite)

// Controllers depend on interface, not implementation
public class SubscriptionsController
{
    private readonly ISubscriptionService _service;

    public SubscriptionsController(ISubscriptionService service)
    {
        _service = service;
    }

    [HttpPost]
    public async Task<ActionResult<SubscriptionResponse>> AddSubscription(
        CreateSubscriptionRequest request)
    {
        var subscription = await _service.AddSubscriptionAsync(request.Url);
        return Ok(subscription);
    }
}
```

No API changes or UI changes required when switching to EF Core implementation.

---

## Relationships

### MVP Relationships

**Current (MVP)**: No relationships. Subscription is a standalone entity.

### Future Relationships (Extended-MVP and beyond)

**Placeholder** for future design:
- `Subscription ↔ FeedItem`: Each subscription can have multiple feed items (1:N)
- `Subscription ↔ User`: Each subscription belongs to one user (when multi-user support added, N:1)
- `Subscription ↔ RefreshLog`: Each subscription has optional refresh history (1:N)

Example future schema (Post-MVP):
```csharp
// Extended-MVP: Add user association
public class Subscription
{
    public string Id { get; set; }
    public string Url { get; set; }
    public DateTime AddedAt { get; set; }
    public string UserId { get; set; }  // Foreign key to User

    public User User { get; set; }  // Navigation property
    public List<FeedItem> Items { get; set; }  // Navigation property
}

public class FeedItem
{
    public string Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public DateTime PublishedAt { get; set; }
    public string SubscriptionId { get; set; }  // Foreign key

    public Subscription Subscription { get; set; }  // Navigation property
}
```

**Design Note**: Current MVP structure allows future extension without breaking existing code.

---

## State Transitions

### MVP State Model

**Current (MVP)**: Subscription has no state machine; it exists in a single state once created.

```
[Creation] --[AddSubscription]--> [Stored] --[AppRestart]--> [Deleted]
```

### Future State Transitions (Extended-MVP and beyond)

**Placeholder** for future design when subscription lifecycle becomes more complex:

```
[New] --[Add]--> [Active] --[Refresh]--> [Refreshed]
                    |
                    +--[Delete]--> [Deleted]
                    |
                    +--[Error Fetching]--> [ErrorState]
```

Example future properties to support state transitions:
```csharp
public enum SubscriptionStatus
{
    Active,
    Paused,
    ErrorFetching,
    Deleted
}

public class Subscription
{
    // ... existing properties ...
    public SubscriptionStatus Status { get; set; } = SubscriptionStatus.Active;
    public DateTime? LastRefreshedAt { get; set; }
    public string? LastErrorMessage { get; set; }
}
```

**No state transitions in MVP**: All subscriptions are immediately "Active" upon creation; no state changes occur during session.

---

## API Data Transfer Objects (DTOs)

### Request DTO: CreateSubscriptionRequest

```csharp
public class CreateSubscriptionRequest
{
    /// <summary>
    /// The feed URL provided by the user.
    /// </summary>
    [JsonPropertyName("url")]
    public string Url { get; set; }
}
```

**JSON Example**:
```json
{
  "url": "https://devblogs.microsoft.com/dotnet/feed/"
}
```

### Response DTO: SubscriptionResponse

```csharp
public class SubscriptionResponse
{
    [JsonPropertyName("id")]
    public string Id { get; set; }

    [JsonPropertyName("url")]
    public string Url { get; set; }

    [JsonPropertyName("addedAt")]
    public DateTime AddedAt { get; set; }
}
```

**JSON Example**:
```json
{
  "id": "a1b2c3d4-e5f6-47a8-b9c0-d1e2f3a4b5c6",
  "url": "https://devblogs.microsoft.com/dotnet/feed/",
  "addedAt": "2026-07-31T16:17:16Z"
}
```

### Response DTO: GetSubscriptionsResponse

```csharp
public class GetSubscriptionsResponse
{
    [JsonPropertyName("subscriptions")]
    public List<SubscriptionResponse> Subscriptions { get; set; }
}
```

**JSON Example** (with subscriptions):
```json
{
  "subscriptions": [
    {
      "id": "a1b2c3d4-e5f6-47a8-b9c0-d1e2f3a4b5c6",
      "url": "https://devblogs.microsoft.com/dotnet/feed/",
      "addedAt": "2026-07-31T16:17:16Z"
    },
    {
      "id": "b2c3d4e5-f6a7-48b9-c0d1-e2f3a4b5c6d7",
      "url": "https://example.com/rss",
      "addedAt": "2026-07-31T16:18:00Z"
    }
  ]
}
```

**JSON Example** (empty list):
```json
{
  "subscriptions": []
}
```

---

## Design Decisions & Rationale

### Why GUID for ID?

- **Uniqueness**: GUID provides global uniqueness without coordination (no central ID server)
- **Simplicity**: Built-in to .NET; no additional library needed
- **String representation**: Can be stored in JSON directly; no numeric ID type confusion
- **Future-proof**: Works with databases (SQLite, PostgreSQL) in Extended-MVP

### Why UTC DateTime?

- **Consistency**: Eliminates timezone ambiguity; all times comparable regardless of server location
- **Serialization**: ISO 8601 format is language-agnostic and widely understood
- **Database compatibility**: UTC is standard for web applications; no conversion issues

### Why No URL Validation?

- **MVP speed**: Validation is deferred to Extended-MVP
- **User responsibility**: Users are trusted to provide valid feed URLs
- **Simplicity**: Avoids regex complexity or external URL parsing library
- **Future flexibility**: Can add validation without changing API contract (returns 400 if URL invalid)

### Why Immutability?

- **Data integrity**: Prevents accidental/malicious modification of subscription metadata
- **Audit trail**: `AddedAt` timestamp remains accurate (not updated by mistake)
- **Thread-safety**: Read-only data is inherently thread-safe (future phases)
- **API contract**: Clients know data won't change unexpectedly

---

## Summary

The MVP Subscription entity is minimal but complete:
- ✅ Three properties: `Id` (GUID), `Url` (string), `AddedAt` (DateTime UTC)
- ✅ Simple validation: Non-empty URL; trimmed
- ✅ In-memory storage: Session-scoped List<Subscription>
- ✅ No relationships or state transitions
- ✅ Extensible design: Service layer enables future database implementation without API changes
- ✅ Strong typing: DTOs ensure correct serialization and type safety

**Ready for Phase 1: API Contracts** (see contracts/ directory)
