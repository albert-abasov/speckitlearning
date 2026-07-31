# Research: RSS Feed Subscription Management MVP

**Date**: 2026-07-31 | **Branch**: 001-subscription-management

This document resolves all NEEDS CLARIFICATION items from the Technical Context by researching best practices, architecture patterns, and technology choices for the RSS Feed Reader MVP.

---

## Research Topic 1: ASP.NET Core Web API Best Practices for Stateless Subscription Management

### Decision

Use ASP.NET Core 8 with a clean, stateless REST API architecture:
- Controllers for HTTP request handling (SubscriptionsController)
- Service layer abstraction (ISubscriptionService) for business logic
- In-memory List<Subscription> implementation for MVP storage
- Dependency injection via AddScoped/AddSingleton
- Built-in System.Text.Json for serialization (no external JSON libraries needed)

### Rationale

ASP.NET Core 8 is the latest LTS release with native support for:
1. **Async/await patterns**: Natural fit for HTTP I/O and future feed fetching
2. **Dependency injection**: Built-in, enables testable service abstraction
3. **Minimal setup**: Program.cs with no Startup class clutter
4. **Cross-platform**: Windows, macOS, Linux via dotnet CLI
5. **Performance**: High throughput for subscription add/list operations (<100ms target)
6. **Incremental growth**: Service layer pattern supports swapping in-memory store for EF Core + SQLite

Stateless architecture (no session state on server) enables:
- Easy horizontal scaling (future phases)
- Predictable memory footprint
- No session affinity requirements
- Clear contract between frontend and API

### Alternatives Considered

1. **Express.js (Node.js)**: Cross-platform, but adds npm dependency overhead; .NET is more strongly-typed for data contracts
2. **FastAPI (Python)**: Good for rapid prototyping, but .NET is specified in constitution; Python lacks strong typing for API contracts
3. **Stateful server with Session**: Would couple frontend to server state; violates contract clarity principle
4. **GraphQL instead of REST**: Over-engineered for MVP; REST is simpler and meets spec requirements

### Evidence

- [ASP.NET Core docs: Dependency Injection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
- [ASP.NET Core performance benchmarks](https://www.techempower.com/benchmarks/)
- Constitution Principle V: Incremental Architecture requirement for swappable storage layer

---

## Research Topic 2: Blazor WebAssembly HTTP Client Patterns for Frontend-Backend Communication

### Decision

Use Blazor WebAssembly with:
- Built-in `HttpClient` injected via dependency injection
- `HttpClient.GetFromJsonAsync<T>()` and `HttpClient.PostAsJsonAsync()` for typed communication
- Service layer (SubscriptionApiClient) to encapsulate API communication
- CORS configuration on backend to allow localhost:5213 → localhost:5151 requests
- appsettings.json in wwwroot/ for environment-specific API endpoint configuration

### Rationale

Blazor WebAssembly advantages for MVP:
1. **Native .NET code in browser**: Same language (C#) on frontend and backend; no JavaScript transpilation
2. **Strong typing**: DTO classes ensure compile-time checking of API contracts
3. **Built-in HttpClient**: No third-party HTTP library needed (unlike axios or fetch)
4. **Dependency injection**: Consistent with backend patterns
5. **Component state management**: Simple local state for subscription list
6. **WASM interop**: Foundation for future JavaScript integration (e.g., IndexedDB for offline support)

Service layer pattern (SubscriptionApiClient) enables:
- Centralized API endpoint management
- Easy mocking for component testing
- Separation of UI logic from HTTP communication
- Clear error handling and retry patterns (future phases)

CORS configuration approach:
```csharp
// In Program.cs (backend)
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("http://localhost:5213")
            .AllowAnyMethod()
            .AllowAnyHeader());
});
app.UseCors("AllowFrontend");
```

### Alternatives Considered

1. **React + TypeScript**: Popular, but adds Node.js build complexity; Blazor is integrated with .NET toolchain
2. **Vue.js**: Lighter weight, but less strongly-typed; C# type safety reduces runtime errors
3. **Plain JavaScript/Fetch API**: No framework overhead, but loses IDE support and type checking
4. **SignalR (real-time WebSockets)**: Over-engineered for MVP; HTTP polling sufficient; SignalR can be added in Extended-MVP for push notifications

### Evidence

- [Blazor official documentation](https://learn.microsoft.com/en-us/aspnet/core/blazor/)
- [HttpClient for Blazor](https://learn.microsoft.com/en-us/aspnet/core/blazor/http-request-examples)
- [CORS in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/cors)
- Constitution Principle II: API-WebAssembly Contract Clarity (explicit, versioned endpoints; typed DTOs)

---

## Research Topic 3: xUnit Testing Patterns for In-Memory Service Testing

### Decision

Use xUnit with the following patterns:
- **Unit tests**: InMemorySubscriptionServiceTests class testing service logic in isolation
- **Test methods**: One assertion per test (Arrange-Act-Assert pattern)
- **Mocking**: xUnit.Abstractions for output capture; Moq library for dependencies (if needed later)
- **Fixtures**: xUnit IAsyncLifetime for setup/teardown (future phases with EF Core)
- **Theory tests**: `[Theory]` + `[InlineData]` for parameterized scenarios (e.g., multiple subscriptions)

Example test structure:
```csharp
public class InMemorySubscriptionServiceTests
{
    [Fact]
    public async Task AddSubscription_WithValidUrl_ReturnsSubscriptionWithId()
    {
        // Arrange
        var service = new InMemorySubscriptionService();
        var url = "https://example.com/feed";

        // Act
        var result = await service.AddSubscriptionAsync(url);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(url, result.Url);
        Assert.NotEmpty(result.Id);
    }

    [Theory]
    [InlineData("https://feed1.com")]
    [InlineData("https://feed2.com")]
    [InlineData("https://feed3.com")]
    public async Task AddMultipleSubscriptions_AllStored(string url)
    {
        // Test implementation
    }
}
```

### Rationale

xUnit chosen (per constitution) because:
1. **ASP.NET Core native**: First-class integration with .NET 8
2. **Simple syntax**: No complex setup; `[Fact]` and `[Theory]` are intuitive
3. **Async-first**: Native Task-based testing (no complex async wrappers)
4. **Extensible**: IAsyncLifetime for complex fixtures; ITestOutputHelper for logging
5. **Performance**: Fast test execution; suitable for CI/CD pipelines

Testing patterns for MVP:
- **Happy path tests**: Valid URL → subscription created with unique ID
- **Error cases**: Empty URL → 400 Bad Request response
- **State verification**: Multiple subscriptions → list contains all in order
- **Isolation**: Each test uses fresh service instance (no state leakage)

### Alternatives Considered

1. **NUnit**: Similar to xUnit, but slightly more verbose; xUnit is modern .NET standard
2. **MSTest**: Microsoft-owned, but less popular in open-source .NET ecosystem
3. **SpecFlow (BDD)**: Over-engineered for MVP; plain xUnit sufficient for simple scenarios
4. **Hand-written mock objects**: Possible for MVP, but Moq is lightweight and standard

### Evidence

- [xUnit.net documentation](https://xunit.net/)
- [xUnit best practices](https://xunit.net/docs/getting-started/netcore)
- [ASP.NET Core testing patterns](https://docs.microsoft.com/en-us/aspnet/core/test/unit-tests-with-nunit)
- Constitution Principle IV: Test-Driven Quality (xUnit is designated framework)

---

## Research Topic 4: CORS Configuration for Local Development (Ports 5151, 5213)

### Decision

Configure CORS on ASP.NET Core backend (port 5151) to allow Blazor frontend (port 5213):
1. Add CORS policy in Program.cs specifying allowed origin: "http://localhost:5213"
2. Allow all HTTP methods (GET, POST, OPTIONS, etc.)
3. Allow all headers (Content-Type, Authorization, etc.)
4. Enable credentials (future phases for authentication)
5. Document policy and rationale in configuration file

Configuration code:
```csharp
// In Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("http://localhost:5213")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials());
});

app.UseCors("AllowFrontend");
```

### Rationale

CORS (Cross-Origin Resource Sharing) is required because:
1. **Different origins**: Frontend (localhost:5213) and backend (localhost:5151) have different ports → same-origin policy blocks requests
2. **Preflight requests**: Browser sends OPTIONS request before POST; backend must respond with CORS headers
3. **Security principle**: CORS explicitly defines allowed origins; prevents arbitrary cross-origin requests
4. **Standard practice**: CORS is industry standard for web APIs; improves security posture

MVP CORS policy is permissive (AllowAnyMethod, AllowAnyHeader) because:
- No authentication/authorization in MVP
- Local development environment only
- Simple MVP doesn't require fine-grained access control
- Future phases can restrict to specific headers/methods as needed

Security note:
- MVP assumes no malicious frontend (single-user, local development)
- Production deployment will use restrictive CORS policy with specific origins, methods, and headers
- Constitution Principle III (Security by Design) establishes this foundation for Extended-MVP

### Alternatives Considered

1. **No CORS restriction**: Security risk; explicit policy is better practice
2. **AllowAnyOrigin**: Works for development, but disables security validation; explicit origin is better
3. **Proxy pattern**: Frontend proxies requests through same-origin backend (add complexity; CORS is simpler)
4. **WebSockets (SignalR)**: Different protocol, eliminates CORS; over-engineered for MVP

### Evidence

- [CORS in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/cors)
- [MDN CORS documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- Constitution Principle III: Security by Design (explicit CORS policy)
- [Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)

---

## Research Topic 5: Port Configuration and Launch Settings for Dual-Project Startup

### Decision

Use Visual Studio / Visual Studio Code launch configurations:
1. **Backend (RssReader.Api)**: Port 5151 via launchSettings.json
2. **Frontend (RssReader.Frontend)**: Port 5213 via launchSettings.json
3. **Startup approach**: 
   - Use Visual Studio "Launch Multiple Projects" or
   - Use dotnet CLI: two terminals, `dotnet run` in each project directory
4. **Environment-specific config**: appsettings.json in frontend wwwroot/ specifies backend API URL
5. **Debugging**: Visual Studio debugger supports breakpoints in both backend and frontend (WASM debugging requires browser dev tools)

Backend launchSettings.json:
```json
{
  "profiles": {
    "Api": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "api/v1/subscriptions",
      "applicationUrl": "http://localhost:5151",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

Frontend launchSettings.json:
```json
{
  "profiles": {
    "Frontend": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "http://localhost:5213",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

Frontend appsettings.json (development):
```json
{
  "Api": {
    "BaseUrl": "http://localhost:5151"
  }
}
```

### Rationale

Separate port configuration:
1. **No port conflicts**: 5151 and 5213 are standard .NET Blazor ports; unlikely to be in use
2. **Isolation**: Each project independently startable; easier troubleshooting
3. **Scalability**: Facilitates deploying to separate servers/containers (future phases)
4. **Development experience**: Developers can stop/restart one project without affecting the other
5. **CI/CD**: Separate build/test/deploy steps per project

Environment-specific configuration:
- **Development**: localhost:5151 and localhost:5213
- **Staging/Production**: Will use DNS names / load balancer URLs (future phases)
- Frontend reads API URL from appsettings.json at runtime (no hardcoding)

Debugging approach:
- Backend: Set breakpoints in SubscriptionsController.cs, attach Visual Studio debugger
- Frontend: WASM debugging is limited; use browser console, HttpClient logging, or Blazor REPL

### Alternatives Considered

1. **Single port with reverse proxy**: Adds complexity (nginx/IIS setup); separate ports are simpler for MVP
2. **Embedded frontend**: Serve Blazor from backend; loses flexibility and couples frontend/backend deployments
3. **Docker Compose for local dev**: Adds Docker dependency; dotnet run is simpler and faster for MVP
4. **Hard-coded API URL**: Works but makes switching environments difficult; configuration is better practice

### Evidence

- [ASP.NET Core launchSettings.json documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/)
- [Blazor WebAssembly hosting](https://learn.microsoft.com/en-us/aspnet/core/blazor/host-and-deploy/webassembly)
- [Visual Studio debugging](https://docs.microsoft.com/en-us/visualstudio/debugger/getting-started-with-the-debugger)
- Constitution Principle V: Incremental Architecture (configuration enables future Docker/K8s deployment)

---

## Summary of Research Findings

All NEEDS CLARIFICATION items from Technical Context have been resolved:

| Item | Decision | Confidence |
|------|----------|-----------|
| Language/Version | C# 12 (.NET 8 LTS / ASP.NET Core 8) | ✅ High |
| Primary Dependencies | ASP.NET Core, Blazor WebAssembly, xUnit | ✅ High |
| Storage | In-memory List<Subscription> (C#) | ✅ High |
| Testing | xUnit with unit/integration patterns | ✅ High |
| Target Platform | Cross-platform (Windows, macOS, Linux) | ✅ High |
| Project Type | Web service hybrid (backend + frontend) | ✅ High |
| Performance Goals | <100ms API response, 1000 subscriptions max | ✅ High |
| Constraints | Single-user, in-memory, no persistence | ✅ High |
| Scale | MVP for single session | ✅ High |

### Constitutional Alignment

All research decisions align with the ratified constitution:
- ✅ **MVP-First Delivery**: Focused scope (add + list), in-memory storage, no feed operations
- ✅ **API-WebAssembly Contract Clarity**: Versioned REST endpoints, typed DTOs, explicit CORS
- ✅ **Security by Design**: URL input validation, no hardcoded secrets, CORS policy, no shell commands
- ✅ **Test-Driven Quality**: xUnit patterns, service abstraction, testable architecture
- ✅ **Incremental Architecture**: Service layer abstraction, configuration-driven setup, extensible design

**Ready for Phase 1: Design & Contracts**
