# API Contract: Get All Subscriptions

**Endpoint**: `GET /api/v1/subscriptions`

**Version**: v1 (stable for MVP)

**Date**: 2026-07-31

**Status**: Ready for implementation

---

## Summary

Retrieves all RSS/Atom feed subscriptions in the user's collection. The server returns a list of all stored subscriptions in the order they were added (FIFO).

---

## Request

### HTTP Method & Path

```
GET /api/v1/subscriptions
```

### Headers

| Header | Value | Required | Notes |
|--------|-------|----------|-------|
| `Accept` | `application/json` | Yes | Indicates JSON response expected |

### Query Parameters

**None for MVP**

No query parameters are defined in the MVP contract. Future versions may add:
- `?skip=10&take=20` for pagination
- `?search=keyword` for filtering
- `?sort=addedAt` for sorting

All subscriptions are returned in a single response for MVP.

### Request Examples

**Basic request** (no query parameters):
```bash
curl -X GET http://localhost:5151/api/v1/subscriptions \
  -H "Accept: application/json"
```

**Using httpie**:
```bash
http GET localhost:5151/api/v1/subscriptions
```

---

## Success Response

### HTTP Status Code

```
200 OK
```

### Response Headers

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |

### Response Body (JSON)

**Schema**:
```json
{
  "subscriptions": [
    {
      "id": "string (UUID)",
      "url": "string",
      "addedAt": "string (ISO 8601 UTC datetime)"
    }
  ]
}
```

### Response Examples

**Example 1: With Subscriptions**

**Response**:
```json
{
  "subscriptions": [
    {
      "id": "a1b2c3d4-e5f6-47a8-b9c0-d1e2f3a4b5c6",
      "url": "https://devblogs.microsoft.com/dotnet/feed/",
      "addedAt": "2026-07-31T16:17:16.123Z"
    },
    {
      "id": "b2c3d4e5-f6a7-48b9-c0d1-e2f3a4b5c6d7",
      "url": "https://example.com/feed",
      "addedAt": "2026-07-31T16:18:00.456Z"
    },
    {
      "id": "c3d4e5f6-a7b8-49ca-d1e2-f3a4b5c6d7e8",
      "url": "https://tech.news/rss",
      "addedAt": "2026-07-31T16:19:30.789Z"
    }
  ]
}
```

**Example 2: Empty Subscriptions List**

**Response**:
```json
{
  "subscriptions": []
}
```

**Field Details**:

| Field | Type | Description |
|-------|------|-------------|
| `subscriptions` | array | List of subscription objects; may be empty |
| `subscriptions[].id` | string (UUID) | Unique identifier for subscription |
| `subscriptions[].url` | string | Feed URL as stored |
| `subscriptions[].addedAt` | string (ISO 8601 UTC) | Timestamp when subscription was created |

### Order Guarantee

**FIFO (First-In-First-Out)**: Subscriptions are returned in the order they were added.

**Example Order**:
1. User adds "https://feed1.com" at 16:00:00 → appears first in list
2. User adds "https://feed2.com" at 16:05:00 → appears second
3. User adds "https://feed3.com" at 16:10:00 → appears third

GET response returns subscriptions in this order (oldest first).

---

## Error Responses

### Error 500: Internal Server Error

**Scenario**: Unexpected server error while retrieving subscriptions (e.g., out of memory, unhandled exception).

**HTTP Status Code**:
```
500 Internal Server Error
```

**Response Headers**:
```
Content-Type: application/json
```

**Response Body (JSON)**:
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred while retrieving subscriptions"
}
```

**Example Trigger**:
```bash
curl -X GET http://localhost:5151/api/v1/subscriptions
```

**Response**:
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred while retrieving subscriptions"
}
```

**Client Action**: Log error; inform user; allow retry.

**Development Note**: Server MUST log full exception details to error log for debugging; client receives sanitized message only.

---

## State Semantics

### What GET Returns

- **Snapshot**: GET returns a snapshot of all subscriptions at the time the request is processed
- **Consistency**: Responses are consistent within a single HTTP request-response cycle
- **Order**: Guaranteed FIFO order (insertion order preserved)
- **Completeness**: All subscriptions added via POST are included (no filtering, deduplication, or archiving)

### What GET Does NOT Return

- **Deleted subscriptions**: MVP has no delete endpoint; all added subscriptions are returned
- **Feed content**: No feed items, parsed content, or refresh status returned
- **Metadata**: No refresh history, last updated time, or error logs (future Extended-MVP features)

---

## Caching

**Caching**: Not applicable for MVP

- Response should include `Cache-Control: no-cache` header (don't cache at browser/proxy)
- Each GET request fetches current in-memory state
- Fresh data guaranteed on every request

Future consideration for Extended-MVP:
- ETags for conditional GET (return 304 Not Modified if unchanged)
- Cache expiration: e.g., `Cache-Control: max-age=60` for 60-second cache

---

## Performance & Scalability

### MVP Performance

**Response Time Goal**: <100ms for all subscriptions (at <1000 subscriptions)

**Memory**: ~55 KB for 1000 subscriptions (estimate)

**Bandwidth**: Minimal (single JSON array, no streaming)

### Future Scalability (Extended-MVP and beyond)

**Current limitation**: Returns ALL subscriptions in single response.

**Future improvements**:
- **Pagination**: Add `?skip=0&take=20` query parameters
- **Filtering**: Add `?search=keyword` to filter subscriptions
- **Sorting**: Add `?sort=addedAt&order=asc` options
- **Streaming**: Large result sets returned as JSON streaming (for extreme scale)

These changes can be made without breaking existing MVP clients (new query parameters are optional).

---

## Concurrency & Thread Safety

**Thread Safety (MVP)**: Not required

- MVP is single-user, local development environment
- No multi-threaded access expected
- In-memory List<Subscription> is not thread-safe

**Future consideration (Extended-MVP)**:
- Add lock/concurrent collection if multi-threaded scenarios emerge
- Document thread-safety guarantees in Extended-MVP contract update

---

## Security Considerations

### Data Exposure (MVP)

- **No authentication**: MVP assumes single user; no secrets exposed via API
- **Local access only**: Localhost-only deployment; no internet exposure
- **No sensitive data**: Subscription URLs are user-provided; no credentials, tokens, or secrets in responses

### Future Security (Extended-MVP)

- **Authentication**: Add JWT or API key if multi-user support added
- **Authorization**: Ensure users only see their own subscriptions
- **HTTPS**: Enforce HTTPS in production deployment
- **Rate limiting**: Limit requests to prevent DoS attacks

---

## Versioning

**API Version**: v1 (stable)

**Version Strategy**: Same as POST endpoint (see `/contracts/add-subscription.md`)

**Backwards Compatibility**:
- This contract is stable and will not change within v1
- Clients can safely depend on this contract
- Changes that add new fields to `SubscriptionResponse` are non-breaking (client can ignore unknown fields)

---

## Implementation Notes

### ASP.NET Core Controller Implementation

```csharp
[ApiController]
[Route("api/v1/subscriptions")]
public class SubscriptionsController : ControllerBase
{
    private readonly ISubscriptionService _service;

    public SubscriptionsController(ISubscriptionService service)
    {
        _service = service;
    }

    [HttpGet]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status500InternalServerError)]
    public async Task<ActionResult<GetSubscriptionsResponse>> GetSubscriptions()
    {
        try
        {
            var subscriptions = await _service.GetSubscriptionsAsync();
            var response = new GetSubscriptionsResponse
            {
                Subscriptions = subscriptions
                    .Select(s => new SubscriptionResponse
                    {
                        Id = s.Id,
                        Url = s.Url,
                        AddedAt = s.AddedAt
                    })
                    .ToList()
            };
            return Ok(response);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving subscriptions");
            return StatusCode(
                StatusCodes.Status500InternalServerError,
                new ErrorResponse
                {
                    Error = "Internal server error",
                    Message = "An unexpected error occurred while retrieving subscriptions"
                });
        }
    }
}
```

### Blazor Frontend Implementation

```csharp
@inject SubscriptionApiClient ApiClient
@implements IAsyncDisposable

<div class="subscription-list">
    @if (subscriptions == null)
    {
        <p>Loading...</p>
    }
    else if (subscriptions.Count == 0)
    {
        <p>No subscriptions yet. Add one to get started.</p>
    }
    else
    {
        <ul>
            @foreach (var sub in subscriptions)
            {
                <li>@sub.Url</li>
            }
        </ul>
    }
</div>

@code {
    private List<SubscriptionResponse> subscriptions = null;

    protected override async Task OnInitializedAsync()
    {
        await RefreshSubscriptions();
    }

    private async Task RefreshSubscriptions()
    {
        try
        {
            var response = await ApiClient.GetSubscriptionsAsync();
            subscriptions = response.Subscriptions;
        }
        catch (Exception ex)
        {
            // Log and display error
        }
    }

    async ValueTask IAsyncDisposable.DisposeAsync()
    {
        // Cleanup if needed
    }
}
```

### API Client Service Implementation

```csharp
public class SubscriptionApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<SubscriptionApiClient> _logger;

    public SubscriptionApiClient(HttpClient httpClient, ILogger<SubscriptionApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<GetSubscriptionsResponse> GetSubscriptionsAsync()
    {
        try
        {
            var response = await _httpClient.GetAsync("api/v1/subscriptions");
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadAsAsync<GetSubscriptionsResponse>();
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to retrieve subscriptions");
            throw;
        }
    }
}
```

---

## Test Cases

See `/specs/001-subscription-management/tests/` for comprehensive unit and integration test cases.

**Quick Reference**:
- ✅ Get empty list → 200 OK with empty `subscriptions: []`
- ✅ Get after adding 1 subscription → 200 OK with 1 item
- ✅ Get after adding 3 subscriptions → 200 OK with all 3 in FIFO order
- ✅ Get preserves subscription data → IDs, URLs, and timestamps are accurate
- ✅ Repeated GET calls are idempotent → same data returned (unless subscriptions added between calls)
- ✅ No query parameters in MVP → advanced filtering deferred to Extended-MVP

---

## Related Endpoints

- **POST /api/v1/subscriptions**: Add a new subscription (see `/contracts/add-subscription.md`)
- **Future DELETE /api/v1/subscriptions/{id}**: Remove a subscription (Extended-MVP)
- **Future PUT /api/v1/subscriptions/{id}**: Update a subscription (Extended-MVP)
- **Future GET /api/v1/subscriptions/{id}/items**: Fetch feed items for a subscription (Extended-MVP)
