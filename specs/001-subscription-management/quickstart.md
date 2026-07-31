# Quickstart & Validation Guide

**Date**: 2026-07-31 | **Branch**: 001-subscription-management | **Phase**: 1 (Design & Contracts)

This guide provides step-by-step instructions to set up, run, and validate the RSS Feed Subscription Management MVP.

---

## Overview

This quickstart demonstrates the complete MVP workflow:
1. Start backend API (ASP.NET Core, port 5151)
2. Start frontend UI (Blazor WebAssembly, port 5213)
3. Add subscriptions via the UI
4. Verify subscriptions appear in the list
5. Clean up and verify

**Time Required**: ~30 minutes (including setup and validation)

**Expected Outcome**: Both backend and frontend running; user can add and view subscriptions without errors.

---

## Prerequisites

### System Requirements

- **Operating System**: Windows, macOS, or Linux
- **RAM**: 2 GB minimum (8 GB recommended)
- **Disk Space**: 1 GB for .NET SDK and project

### Software Requirements

- **.NET 8 SDK or later** (download from https://dotnet.microsoft.com/download)
- **Git** (for cloning the repository)
- **Modern web browser** (Chrome, Firefox, Safari, or Edge with WebAssembly support)
- **Text editor or IDE** (Visual Studio 2022 Community Edition recommended, or Visual Studio Code)

### Port Availability

Verify that ports 5151 and 5213 are available:

**Windows**:
```powershell
# Check if ports are in use
Get-NetTCPConnection -LocalPort 5151 -ErrorAction SilentlyContinue
Get-NetTCPConnection -LocalPort 5213 -ErrorAction SilentlyContinue
```

**macOS/Linux**:
```bash
# Check if ports are in use
lsof -i :5151
lsof -i :5213
```

If either port is in use, stop the occupying service or choose alternative ports (update launchSettings.json in both projects).

### Verification

**Check .NET installation**:
```bash
dotnet --version
# Should output: 8.0.x or higher
```

---

## Setup Steps

### Step 1: Clone the Repository

```bash
git clone https://github.com/albert-abasov/speckitlearning.git
cd speckitlearning
git checkout 001-subscription-management
```

### Step 2: Verify Project Structure

```bash
# Expected directory structure:
# ./RssReader.Backend/
#   ├── RssReader.Api/
#   │   ├── Program.cs
#   │   ├── Controllers/
#   │   ├── Models/
#   │   ├── Services/
#   │   └── Dtos/
#   ├── RssReader.Api.Tests/
#   └── RssReader.sln
#
# ./RssReader.Frontend/
#   ├── RssReader.Frontend/
#   │   ├── Program.cs
#   │   ├── Pages/
#   │   ├── Services/
#   │   └── wwwroot/
#   └── RssReader.Frontend.sln
```

Verify both solution files exist:
```bash
ls RssReader.Backend/*.sln
ls RssReader.Frontend/*.sln
```

### Step 3: Restore Dependencies

**Backend**:
```bash
cd RssReader.Backend
dotnet restore
cd ..
```

**Frontend**:
```bash
cd RssReader.Frontend
dotnet restore
cd ..
```

---

## Running the Application

### Option A: Visual Studio 2022 (Recommended)

**Backend**:
1. Open `RssReader.Backend/RssReader.sln` in Visual Studio
2. Set `RssReader.Api` as the startup project
3. Press F5 or click "Start Debugging"
4. Wait for browser to open at `http://localhost:5151`
5. A blank page or swagger documentation should appear

**Frontend**:
1. Open `RssReader.Frontend/RssReader.Frontend.sln` in another Visual Studio instance
2. Set `RssReader.Frontend` as the startup project
3. Press F5 or click "Start Debugging"
4. Wait for browser to open at `http://localhost:5213`
5. The application UI should load

### Option B: Visual Studio Code + .NET CLI

**Backend** (Terminal 1):
```bash
cd RssReader.Backend/RssReader.Api
dotnet run
# Output: info: Microsoft.Hosting.Lifetime[14]
#         Now listening on: http://localhost:5151
```

**Frontend** (Terminal 2):
```bash
cd RssReader.Frontend/RssReader.Frontend
dotnet run
# Output: info: Microsoft.Hosting.Lifetime[14]
#         Now listening on: http://localhost:5213
```

### Option C: .NET CLI with Multiple Launch Profiles

**In Visual Studio Code** or using `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "compounds": [
    {
      "name": "Backend + Frontend",
      "configurations": ["Backend", "Frontend"]
    }
  ],
  "configurations": [
    {
      "name": "Backend",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build-backend",
      "program": "${workspaceFolder}/RssReader.Backend/RssReader.Api/bin/Debug/net8.0/RssReader.Api.dll",
      "args": [],
      "cwd": "${workspaceFolder}/RssReader.Backend/RssReader.Api",
      "stopAtEntry": false
    },
    {
      "name": "Frontend",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build-frontend",
      "program": "${workspaceFolder}/RssReader.Frontend/RssReader.Frontend/bin/Debug/net8.0/RssReader.Frontend.dll",
      "args": [],
      "cwd": "${workspaceFolder}/RssReader.Frontend/RssReader.Frontend",
      "stopAtEntry": false
    }
  ]
}
```

---

## Validation Scenarios

### Scenario 1: Empty List Display

**Objective**: Verify the application loads with an empty subscription list.

**Steps**:
1. Open browser to `http://localhost:5213`
2. Wait for Blazor WebAssembly to initialize (should see loading message briefly)
3. Verify the page displays a message: "No subscriptions yet. Add one to get started."

**Expected Result**: ✅ Empty list message appears

**Failure Diagnosis**:
- If blank page appears: Check browser console for errors (F12 → Console tab)
- If "Loading..." persists: Backend may not be running; verify `http://localhost:5151` is accessible
- If 404 or connection refused: Check CORS configuration in backend Program.cs

---

### Scenario 2: Add Single Subscription

**Objective**: Verify user can add one subscription via UI; verify it appears in the list.

**Steps**:
1. In the browser at `http://localhost:5213`, locate the input field with placeholder "Paste feed URL here..."
2. Enter a test URL: `https://devblogs.microsoft.com/dotnet/feed/`
3. Click the "Add Subscription" button
4. Verify the URL appears in the list below the input field
5. Verify the input field is now empty

**Expected Result**: ✅ Subscription appears in list; input field cleared

**Failure Diagnosis**:
- If input field remains filled: Component may not be clearing state; check Subscriptions.razor component code
- If subscription doesn't appear: Backend API error; check browser DevTools Network tab (F12 → Network)
  - Open POST request to `/api/v1/subscriptions`
  - Verify response is 200 OK with subscription object
  - Check Response body for `id`, `url`, `addedAt` fields
- If error message displays: Read message carefully; common issues:
  - "URL is required" → URL was empty/whitespace
  - "Connection refused" → Backend not running
  - "CORS error" → Backend CORS policy doesn't include frontend origin

---

### Scenario 3: Add Multiple Subscriptions (FIFO Order)

**Objective**: Verify user can add multiple subscriptions; verify they appear in FIFO order (oldest first).

**Steps**:
1. Clear the list (restart application, or use delete functionality if implemented)
2. Add first subscription: `https://example.com/feed1`
   - Verify it appears in list (position 1)
3. Add second subscription: `https://example.com/feed2`
   - Verify both subscriptions appear (position 1: feed1, position 2: feed2)
4. Add third subscription: `https://example.com/feed3`
   - Verify all three appear (position 1: feed1, position 2: feed2, position 3: feed3)

**Expected Result**: ✅ All subscriptions appear in FIFO order (by addition time)

**Verification Checklist**:
- [ ] First subscription appears at top of list
- [ ] Second subscription appears below first (in addition order)
- [ ] Third subscription appears below second (in addition order)
- [ ] Order matches insertion sequence (not alphabetical or random)

**Failure Diagnosis**:
- If subscriptions appear in wrong order: Check backend service (InMemorySubscriptionService); verify List<Subscription> is not being sorted
- If duplicate URLs don't both appear: Backend may be preventing duplicates (MVP allows duplicates; check service code)

---

### Scenario 4: Backend API Validation (via REST Client or cURL)

**Objective**: Verify backend API accepts valid requests and rejects invalid ones.

**Equipment**: REST client tool (Postman, Insomnia, VS Code REST Client) or cURL command-line tool.

#### Test 4a: Add subscription with valid URL (HTTP 200)

**Request**:
```http
POST http://localhost:5151/api/v1/subscriptions HTTP/1.1
Content-Type: application/json

{
  "url": "https://tech.example.com/feed.xml"
}
```

**Expected Response**:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "url": "https://tech.example.com/feed.xml",
  "addedAt": "2026-07-31T16:17:16.123Z"
}
```

**Verification**:
- [ ] HTTP status is 200
- [ ] Response contains `id` (non-empty GUID string)
- [ ] Response contains `url` (matches input)
- [ ] Response contains `addedAt` (ISO 8601 UTC datetime)

**cURL Command**:
```bash
curl -X POST http://localhost:5151/api/v1/subscriptions \
  -H "Content-Type: application/json" \
  -d '{"url":"https://tech.example.com/feed.xml"}'
```

---

#### Test 4b: Add subscription with empty URL (HTTP 400)

**Request**:
```http
POST http://localhost:5151/api/v1/subscriptions HTTP/1.1
Content-Type: application/json

{
  "url": ""
}
```

**Expected Response**:
```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "Invalid request",
  "message": "URL is required and must not be empty"
}
```

**Verification**:
- [ ] HTTP status is 400
- [ ] Response contains `error` field
- [ ] Response contains `message` field with helpful text

**cURL Command**:
```bash
curl -X POST http://localhost:5151/api/v1/subscriptions \
  -H "Content-Type: application/json" \
  -d '{"url":""}'
```

---

#### Test 4c: Get all subscriptions (HTTP 200 empty)

**Request**:
```http
GET http://localhost:5151/api/v1/subscriptions HTTP/1.1
Accept: application/json
```

**Expected Response** (if list is empty):
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "subscriptions": []
}
```

**Verification**:
- [ ] HTTP status is 200
- [ ] Response contains `subscriptions` array
- [ ] Array is empty (length 0)

**cURL Command**:
```bash
curl -X GET http://localhost:5151/api/v1/subscriptions \
  -H "Accept: application/json"
```

---

#### Test 4d: Get all subscriptions (HTTP 200 with data)

**Request** (after adding subscriptions via Test 4a):
```http
GET http://localhost:5151/api/v1/subscriptions HTTP/1.1
Accept: application/json
```

**Expected Response**:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "subscriptions": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "url": "https://tech.example.com/feed.xml",
      "addedAt": "2026-07-31T16:17:16.123Z"
    }
  ]
}
```

**Verification**:
- [ ] HTTP status is 200
- [ ] Response contains `subscriptions` array
- [ ] Array length matches number of added subscriptions
- [ ] Each subscription contains `id`, `url`, `addedAt`

---

### Scenario 5: Data Persistence Within Session

**Objective**: Verify subscriptions persist during the session and are lost on app restart.

**Steps**:
1. Add 2 subscriptions via UI
2. Verify both appear in list
3. Refresh page (Ctrl+R or Cmd+R in browser)
4. Verify subscriptions still appear in list (request sent to backend, data still in memory)
5. Stop the backend application (Ctrl+C in terminal)
6. Refresh page in browser
7. Verify connection error appears (backend unreachable)
8. Restart backend application
9. Refresh page in browser
10. Verify subscription list is now empty (in-memory storage was cleared on restart)

**Expected Result**: ✅ Subscriptions persist during session; lost on restart

**Verification Checklist**:
- [ ] After refresh, subscriptions still appear
- [ ] After restart, backend is accessible again
- [ ] After restart, list is empty
- [ ] New subscriptions can be added after restart

---

## Cleanup & Teardown

### Stop Services

**Backend** (if running in terminal):
```bash
# Press Ctrl+C
^C
```

**Frontend** (if running in terminal):
```bash
# Press Ctrl+C
^C
```

**Visual Studio**:
- Stop Debugger (Shift+F5)
- Close solution windows

### Verify Cleanup

```bash
# Ensure ports are no longer in use
netstat -an | grep 5151  # Should show no connections
netstat -an | grep 5213  # Should show no connections
```

---

## Troubleshooting

### Issue: Backend fails to start (port 5151 in use)

**Diagnosis**:
```bash
# Find process using port 5151
lsof -i :5151  # macOS/Linux
Get-NetTCPConnection -LocalPort 5151  # Windows PowerShell
```

**Solution**:
- Kill the process: `kill -9 <PID>` (macOS/Linux) or `Stop-Process -Id <PID>` (Windows)
- Or change port in `launchSettings.json`: Update `applicationUrl` to `http://localhost:5152`

---

### Issue: Frontend shows "Loading..." indefinitely

**Diagnosis**:
1. Open browser DevTools (F12)
2. Check Console tab for errors
3. Check Network tab for failed requests to `/api/v1/subscriptions`

**Common Causes**:
- Backend not running → Start backend on port 5151
- CORS not configured → Check backend Program.cs for `UseCors()` call
- Wrong API URL in frontend → Check `wwwroot/appsettings.json` for correct `Api.BaseUrl`

**Solution**:
1. Verify backend is running: `curl http://localhost:5151/api/v1/subscriptions`
2. Verify CORS is enabled: Response should include `Access-Control-Allow-Origin` header
3. Check frontend logs in DevTools Console tab for detailed error messages

---

### Issue: 400 Bad Request when adding subscription

**Diagnosis**:
1. Check Network tab in DevTools (F12)
2. Click POST request to `/api/v1/subscriptions`
3. Check Request body and Response body

**Common Causes**:
- Empty URL submitted → Check that input field is not empty before submit
- Wrong JSON format → Verify request is `{"url":"..."}` (lowercase "url" key)
- Whitespace not trimmed → Backend should trim; if error persists, check controller validation

**Solution**:
1. Verify input field has non-empty text
2. Check DevTools Network tab for exact request/response bodies
3. Add console logging to frontend component to debug state

---

### Issue: Subscriptions don't appear after adding

**Diagnosis**:
1. Check HTTP status code (F12 Network tab)
2. Check response body contains valid subscription object
3. Check browser console for errors in component code

**Common Causes**:
- Response wasn't processed correctly → Missing `await` on async call
- Component state not updated → Subscription added to service but UI not refreshed
- Event handler not bound to button → Button click doesn't trigger add function

**Solution**:
1. Add console.log() statements in Blazor component to debug
2. Verify await is used on async calls
3. Check that button has `@onclick="AddSubscription"` or similar binding

---

### Issue: CORS error (browser console shows "No 'Access-Control-Allow-Origin' header")

**Diagnosis**:
```
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource...
Reason: CORS header 'Access-Control-Allow-Origin' missing
```

**Solution**:
1. Verify backend Program.cs has CORS configuration:
   ```csharp
   builder.Services.AddCors(options =>
   {
       options.AddPolicy("AllowFrontend", policy =>
           policy.WithOrigins("http://localhost:5213")
               .AllowAnyMethod()
               .AllowAnyHeader());
   });
   
   app.UseCors("AllowFrontend");
   ```

2. Verify `UseCors()` is called before `MapControllers()`

3. Restart backend after changes to Program.cs

---

## Performance Validation

### API Response Time

**Objective**: Verify backend responds to requests in <100ms.

**Test Tool**: curl with timing

```bash
curl -w "\nTime: %{time_total}s\n" \
  http://localhost:5151/api/v1/subscriptions
```

**Expected Output**:
```
{"subscriptions":[...]}
Time: 0.050s
```

**Pass Criteria**: Time < 0.1s (100ms)

### UI Responsiveness

**Objective**: Verify subscription appears in list within 500ms of clicking "Add".

**Manual Test**:
1. Open DevTools Performance tab (F12 → Performance)
2. Click "Record"
3. Click "Add Subscription" button
4. Click "Stop" recording
5. Review timeline; look for:
   - HTTP request to `/api/v1/subscriptions` (POST)
   - JSON parsing (deserialization)
   - DOM update (list item added)
6. Total time should be <500ms

**Pass Criteria**: Full workflow completes in <500ms

---

## Verification Checklist

Complete this checklist to verify MVP is working correctly:

### Setup ✓
- [ ] .NET 8 SDK installed
- [ ] Ports 5151 and 5213 available
- [ ] Repository cloned and checked out to `001-subscription-management` branch
- [ ] Dependencies restored (`dotnet restore`)

### Backend ✓
- [ ] Backend starts without errors on port 5151
- [ ] Backend accepts HTTP requests
- [ ] CORS policy configured for frontend origin

### Frontend ✓
- [ ] Frontend starts without errors on port 5213
- [ ] Page loads (no blank screen or errors)
- [ ] Empty list message displays initially

### API Contract ✓
- [ ] POST /api/v1/subscriptions accepts valid URL (HTTP 200)
- [ ] POST /api/v1/subscriptions rejects empty URL (HTTP 400)
- [ ] GET /api/v1/subscriptions returns empty list initially (HTTP 200)
- [ ] GET /api/v1/subscriptions returns added subscriptions (HTTP 200)

### User Workflow ✓
- [ ] User can type URL in input field
- [ ] User can click "Add Subscription" button
- [ ] Subscription appears in list immediately
- [ ] Input field clears after adding
- [ ] Multiple subscriptions appear in FIFO order
- [ ] Page refresh preserves subscriptions (session-scoped)
- [ ] App restart clears subscriptions (in-memory lost)

### Data & Contracts ✓
- [ ] Subscription has `id` (GUID)
- [ ] Subscription has `url` (user input, trimmed)
- [ ] Subscription has `addedAt` (ISO 8601 UTC timestamp)
- [ ] Error responses include `error` and `message` fields

### Performance ✓
- [ ] API responds in <100ms
- [ ] UI updates within 500ms of user action
- [ ] No performance degradation with multiple subscriptions

---

## Related Documentation

- [Feature Specification](spec.md) - Full feature requirements
- [Data Model](data-model.md) - Subscription entity, validation rules, storage
- [API Contracts](contracts) - Detailed endpoint contracts and DTOs
- [Implementation Plan](plan.md) - Design architecture and project structure

---

## Next Steps

After successfully running and validating the MVP:

1. **Review Code**: Walk through backend and frontend implementation
2. **Read Tests**: Examine unit and integration test cases
3. **Extended-MVP Planning**: Consider adding feed fetching, persistence, advanced features
4. **Deployment**: Package and deploy to staging/production environment (future phases)

---

## Support & Questions

**For issues or questions**:
1. Check [Troubleshooting](#troubleshooting) section
2. Review browser DevTools Console and Network tabs
3. Check backend logs in terminal/IDE output
4. Consult [API Contracts](contracts) for endpoint details
5. Review [Data Model](data-model.md) for schema information

**For bug reports**:
- File issue on GitHub with: Error message, steps to reproduce, browser/OS info, attached DevTools screenshots
