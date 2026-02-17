# Pi-hole Mobile App — Design Document

**Date:** 2026-02-17
**Status:** Approved
**Scope:** Prototype — single user, iOS + Android + Apple Watch

---

## 1. Overview

A mobile app for controlling a Pi-hole v6 DNS filter from iOS, Android, and Apple Watch. The primary use case is checking whether blocking is active and pausing it for 10 seconds with a single tap. A secondary use case — adding domains to the allowlist or blocklist — is available behind an Advanced disclosure on the phone.

The app targets Pi-hole v6 (Core v6.0.6 / FTL v6.1 / Web v6.1), which introduced a breaking change to authentication: the old static API token is gone, replaced by a session-based REST API.

---

## 2. Architecture

### 2.1 Technology Stack

| Layer | Technology |
|---|---|
| iOS & Android app | React Native CLI (bare workflow) |
| Apple Watch app | SwiftUI / WatchKit (native Xcode target) |
| Phone ↔ Watch bridge | `react-native-watch-connectivity` (WCSession) |
| Credential storage | `react-native-keychain` |
| Test runner (unit + integration) | Jest + React Native Testing Library |
| End-to-end tests | Detox (iOS Simulator + Android Emulator) |
| Integration test target | Docker Pi-hole (`pihole/pihole:2025.04.0`) |
| CI | GitHub Actions |

React Native CLI is chosen over Expo because the WatchKit extension requires direct ownership of the Xcode project. Expo's managed workflow abstracts this away and would require ejecting.

### 2.2 High-Level Diagram

```
┌─────────────────────────────────────────────────┐
│                 React Native App                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │  iOS Build  │        │   Android Build     │ │
│  │  (Xcode)    │        │   (Gradle)          │ │
│  └──────┬──────┘        └─────────────────────┘ │
│         │ react-native-watch-connectivity        │
│         │ (WCSession bridge)                     │
└─────────┼───────────────────────────────────────┘
          │
┌─────────▼──────────────┐
│  WatchKit Extension    │
│  (Swift / SwiftUI)     │
│  - Status display      │
│  - Disable 10s button  │
└────────────────────────┘

React Native App ──────► Pi-hole REST API (v6)
                          http://pi.hole/api
                          POST   /api/auth           login → SID
                          DELETE /api/auth           logout
                          GET    /api/info/host       server name
                          GET    /api/dns/blocking    status
                          POST   /api/dns/blocking    disable w/ timer
                          POST   /api/domains/allow   add to allowlist
                          POST   /api/domains/deny    add to blocklist
                          DELETE /api/domains/allow   remove from allowlist
                          DELETE /api/domains/deny    remove from blocklist
```

### 2.3 Key Architectural Decisions

**The React Native app is the sole API client.** The watch never calls Pi-hole directly. It sends commands to the phone via WCSession; the phone makes the API call and returns the result. This keeps credentials on the phone only and means the watch functions even when it is not independently connected to the home network.

**Sessions span the foreground lifetime of the app.** The app logs in on `AppState → active` and logs out on `AppState → inactive`. A session ID (SID) is held in memory only, never written to disk. React Native's `AppState` API handles both iOS and Android with a single cross-platform implementation. If the app crashes without logging out, Pi-hole's default 5-minute idle timeout will release the session.

**One app password, stored in the device keychain.** Pi-hole v6 has no per-user access control — any valid session has full admin rights. Multi-user role gating is an app-layer concern deferred to a future version. The server URL (defaulting to `http://pi.hole`) is stored in device preferences, not the keychain, since it is not a secret.

---

## 3. Screen & Component Design

### 3.1 Phone App

**First launch:** If the keychain contains no credentials, the app opens directly to Settings. After a successful save, it navigates to Home.

**Settings Screen**

```
┌─────────────────────────────┐
│   Settings                  │
│                             │
│  Server URL                 │
│  [pi.hole                 ] │  ← selectTextOnFocus
│                             │
│  App Password               │
│  [•••••••••••••••••••••   ] │  ← selectTextOnFocus
│                             │
│  [ Save & Connect ]         │
│                             │
│  ✓ Connected to "raspberrypi" │  ← appears after successful auth
└─────────────────────────────┘
```

Both fields use `selectTextOnFocus` so the existing value is selected on tap — a single keystroke replaces it. After tapping Save & Connect, the app calls `POST /api/auth` and then `GET /api/info/host`. On success it shows the server's hostname as confirmation. On failure it shows a specific error toast (see Section 5).

**Home Screen — collapsed**

```
┌─────────────────────────────┐
│  raspberrypi          ⚙️    │  ← gear navigates to Settings
│                             │
│  ● BLOCKING ACTIVE          │  ← green; or red BLOCKING DISABLED
│                             │
│  [ Pause 10s ]              │  ← disabled + spinner while in-flight
│                             │
│  ▶ Advanced…                │  ← collapsed by default
└─────────────────────────────┘
```

**Home Screen — Advanced expanded**

```
┌─────────────────────────────┐
│  raspberrypi          ⚙️    │
│                             │
│  ● BLOCKING ACTIVE          │
│                             │
│  [ Pause 10s ]              │
│                             │
│  ▼ Advanced…                │
│  ───────────────────────    │
│  [example.com             ] │  ← selectTextOnFocus
│  [ ✓ Allow ]  [ ✗ Block ]   │
└─────────────────────────────┘
```

The Advanced section is the natural seam for future role-based access control: a "child" user profile simply never renders it.

### 3.2 Apple Watch App

Single screen. The watch never shows the Advanced section — it is inherently a restricted surface.

```
┌─────────────────┐
│  Pi-hole        │
│                 │
│  ● ACTIVE       │
│                 │
│  [Pause 10s]    │
└─────────────────┘
```

Status is pushed from the phone to the watch via WCSession `applicationContext` every 10 seconds while the phone app is active. If the phone is unreachable, the watch shows a grey status and a disabled button rather than stale data.

---

## 4. Data Flow & API Layer

### 4.1 Module Structure

```
src/
├── api/
│   ├── piholeClient.ts      # All HTTP calls, session management
│   └── types.ts             # TypeScript interfaces for API responses
├── config.ts                # Shared constants (e.g. BLOCKING_PAUSE_SECONDS)
├── store/
│   └── AppStateContext.tsx  # React context: SID, status, server name
├── hooks/
│   ├── usePiholeStatus.ts   # Polling hook (5s interval)
│   └── useAppLifecycle.ts   # AppState login/logout orchestration
├── keychain/
│   └── credentials.ts       # react-native-keychain read/write helpers
└── watch/
    └── WatchBridge.ts       # react-native-watch-connectivity wrapper
```

File naming follows TypeScript/React conventions: PascalCase for classes and components, camelCase for hooks (the `use` prefix is a React rule the filename mirrors), lowercase for plain type/utility files.

### 4.2 Configuration

```typescript
// src/config.ts
export const BLOCKING_PAUSE_SECONDS = 10;
export const STATUS_POLL_INTERVAL_MS = 5000;
export const WATCH_PUSH_INTERVAL_MS = 10000;
export const DEFAULT_SERVER_URL = 'http://pi.hole';
```

No hardcoded durations anywhere else in the codebase. Tests import these constants rather than duplicating magic numbers.

### 4.3 PiholeClient API Surface

```typescript
class PiholeClient {
  // Auth
  login(baseUrl: string, password: string): Promise<{ sid: string }>
  logout(baseUrl: string, sid: string): Promise<void>

  // Info
  getHostInfo(baseUrl: string, sid: string): Promise<{ hostname: string }>

  // Blocking
  getBlockingStatus(baseUrl: string, sid: string): Promise<{ blocking: boolean }>
  disableBlocking(baseUrl: string, sid: string, seconds: number): Promise<void>

  // Domain management
  addDomain(baseUrl: string, sid: string, domain: string, list: 'allow' | 'deny'): Promise<void>
  deleteDomain(baseUrl: string, sid: string, domain: string, list: 'allow' | 'deny'): Promise<void>
}
```

All methods throw a typed `PiholeApiError` (with `.statusCode` and `.message`). The UI handles each error class distinctly (see Section 5). `deleteDomain` is included as a first-class method — it is needed for domain management and is also used as the idempotent teardown mechanism in integration tests.

### 4.4 Session Lifecycle

```
App becomes active
      │
      ▼
credentials.load()  ── not found ──► navigate to Settings
      │
    found
      │
      ▼
piholeClient.login()
  POST /api/auth { password }
      │
    SID ──────────────────────────────────────────┐
      │                                           │
      ▼                                    held in AppStateContext
usePiholeStatus begins polling                  (memory only)
  GET /api/dns/blocking every 5s
  → update status in context
      │
      │   App goes inactive / background
      │
      ▼
piholeClient.logout()
  DELETE /api/auth
  clear SID from context
  stop polling
```

### 4.5 Watch Data Flow

```
Watch: user taps "Pause 10s"
      │
      ▼
WCSession message → { action: 'disableBlocking', seconds: BLOCKING_PAUSE_SECONDS }
      │
      ▼
WatchBridge.ts receives message on phone
      │
      ▼
piholeClient.disableBlocking(...)
      │
      ▼
WCSession reply → { success: true, blocking: false }
      │
      ▼
Watch UI updates status label
```

Phone pushes status to watch via `applicationContext` every `WATCH_PUSH_INTERVAL_MS` while the app is active.

---

## 5. Error Handling

All error states are testID-annotated so they can be asserted in tests without screen captures (see Section 6). Every interactive element that needs to be testable carries a `testID` in `kebab-case-description` format.

| Condition | User-facing behavior |
|---|---|
| Wrong password | Toast: "Authentication failed — check your app password" |
| Wrong URL / no network | Toast: "Cannot reach pi.hole — are you on your home network?" |
| Session expired mid-use | Silently re-authenticate once, retry the action; if re-auth fails, navigate to Settings with the authentication failed toast |
| Pi-hole API error (5xx) | Toast with the raw `message` field from the API response body |
| Watch: phone unreachable | Grey status indicator, Pause button disabled |
| Domain field empty on submit | Inline validation error below the field; no network call made |
| Domain field contains spaces | Inline validation error below the field; no network call made |

---

## 6. Testing Plan

### 6.1 Philosophy

TDD with three layers: unit (pure logic, mocked network), integration (real Docker Pi-hole), and end-to-end (full app on simulator/emulator via Detox). The integration layer targets the real API so we catch actual API surprises — no mocking at that layer.

### 6.2 Layer 1 — Unit Tests (Jest + React Native Testing Library)

**Auth & session**
- `login()` with valid credentials returns a SID string
- `login()` with wrong password throws `PiholeApiError` with `statusCode: 401`
- `logout()` calls `DELETE /api/auth` with the correct SID header
- `logout()` failure is silently swallowed (session cleanup must not crash the app)
- `AppState → active` triggers `login()` then starts polling
- `AppState → inactive` triggers `logout()` and stops polling
- If `login()` fails on becoming active, polling does not start
- First launch with empty keychain routes to Settings screen

**Blocking**
- `getBlockingStatus()` correctly parses `{ blocking: true }`
- `getBlockingStatus()` correctly parses `{ blocking: false }`
- `disableBlocking(sid, BLOCKING_PAUSE_SECONDS)` sends `{ blocking: false, timer: BLOCKING_PAUSE_SECONDS }` in the request body
- Pause button is disabled and shows a spinner while `disableBlocking` is in-flight
- Pause button re-enables after the call resolves (success or failure)

**Domain management**
- `addDomain(domain, 'allow')` calls `POST /api/domains/allow` with the correct body
- `addDomain(domain, 'deny')` calls `POST /api/domains/deny` with the correct body
- `deleteDomain(domain, 'allow')` calls `DELETE /api/domains/allow` with the correct body
- Empty string domain is rejected before the network call (client-side validation)
- Domain containing spaces is rejected before the network call (client-side validation)

**Error handling**
- 401 mid-session triggers one silent re-auth attempt and retries the original action
- 401 on re-auth navigates to Settings with the authentication failed toast
- Network timeout shows the network error toast
- 5xx response shows a toast with the raw `message` field from the API body

**Credentials**
- `credentials.save()` writes to keychain; `credentials.load()` reads it back
- `credentials.load()` returns `null` (not an exception) when the keychain is empty

### 6.3 Layer 2 — Integration Tests (Jest against Docker Pi-hole)

**Setup**

```bash
docker run -d --name pihole-test \
  -e FTLCONF_webserver_port=8080 \
  -e FTLCONF_webserver_api_password=testpassword \
  -p 8080:8080 \
  pihole/pihole:2025.04.0
```

The test suite starts the container in `globalSetup`, runs all integration tests against `http://localhost:8080`, and tears the container down in `globalTeardown`.

**Auth**
- Full login → SID → logout cycle completes without error
- 16 successive login/logout pairs all succeed without error, confirming that clean session management never exhausts the session pool regardless of server-side limits

**Blocking**
- `getBlockingStatus()` returns `{ blocking: true }` on a freshly started container
- `disableBlocking(BLOCKING_PAUSE_SECONDS)` succeeds and status becomes `false`
- Polling with `pollUntil()` (1s interval, timeout = `BLOCKING_PAUSE_SECONDS * 1000 + 5000` ms) confirms blocking resumes automatically without relying on fixed wall-clock sleeps

**Domain management**

A shared `TEST_DOMAIN = 'integration-test-fixture.example.com'` constant is used for all domain tests. `beforeEach` and `afterEach` hooks call `deleteDomain` on both lists (ignoring 404), ensuring each test starts and ends with the domain absent — regardless of test order or prior failure.

- `addDomain(TEST_DOMAIN, 'allow')` succeeds; verified via `GET /api/domains/allow`
- `addDomain(TEST_DOMAIN, 'deny')` succeeds; verified via `GET /api/domains/deny`
- Adding a duplicate domain returns the appropriate API error without throwing an uncaught exception

**Host info**
- `getHostInfo()` returns a non-empty `hostname` string

**Settings screen hostname display**
- After Save & Connect, the element with `testID="connection-status-label"` renders the hostname returned by `getHostInfo()` — asserted with `screen.findByTestId()` (no screenshot required)

### 6.4 Layer 3 — End-to-End Tests (Detox)

Detox runs the same test suite on both iOS Simulator and Android Emulator. Docker Pi-hole runs on the host machine; the app is pointed at `http://localhost:8080`.

**First launch**
- App opens to Settings when keychain is empty
- Entering a bad URL and tapping Save & Connect shows the network error toast (`testID="error-toast"`)
- Entering correct credentials shows the connected hostname label and enables navigation to Home

**Home screen**
- Status indicator shows BLOCKING ACTIVE on load
- Tapping Pause 10s disables the button, shows a spinner, then shows BLOCKING DISABLED
- Polling eventually returns status to BLOCKING ACTIVE automatically

**Advanced section**
- Advanced disclosure is collapsed by default
- Tapping it expands the domain input and Allow / Block buttons
- Entering a valid domain and tapping Allow shows a success toast
- Entering a valid domain and tapping Block shows a success toast
- Tapping Allow or Block with an empty field shows the inline validation error; no network call is made

**Settings navigation**
- Tapping the gear icon from Home navigates to Settings with fields pre-populated
- Changing the app password and saving re-authenticates immediately

### 6.5 Test Infrastructure

| Tool | Purpose |
|---|---|
| Jest | Unit + integration test runner |
| React Native Testing Library | Component rendering and interaction assertions |
| Detox | E2E on iOS Simulator and Android Emulator |
| Docker Pi-hole `2025.04.0` | Real integration target |
| GitHub Actions | Unit + integration on every PR; E2E nightly |

---

## 7. Future Directions

These features are explicitly out of scope for the prototype but are noted here so the architecture does not foreclose them.

**Multi-user with role-based access.** Pi-hole has no native user management — any authenticated session has full admin rights. A future version could store a local user profile table in the app. Each profile determines which UI surfaces render (e.g. a "child" profile suppresses the Advanced section). All API calls would continue to use the single admin app password stored in the keychain; Pi-hole itself would never be aware of multiple users.

**Recently blocked domain suggestions in the allowlist input.** Pi-hole's `GET /api/queries` endpoint returns the full query log. Each record includes the client IP and hostname (if Pi-hole has resolved it via reverse DNS). The app could detect the device's current local IP via `react-native-network-info` and query `GET /api/queries?client=<device-ip>&blocked=true&from=<10-minutes-ago>` to populate a suggestion list below the domain input. A toggle would let the user see recently blocked domains across all devices by omitting the client filter.

**Server auto-discovery.** mDNS/Bonjour browsing via `react-native-zeroconf` to detect Pi-hole instances on the local network, rather than requiring manual URL entry.

**Watch independence.** Currently the watch proxies all API calls through the phone. A future version could have the watch make API calls directly when connected to the home Wi-Fi, falling back to the phone proxy otherwise.

**Configurable pause duration.** The current design hardcodes `BLOCKING_PAUSE_SECONDS = 10` as a shared constant. A future version could expose this as a user-configurable setting, and optionally offer multiple duration presets on the watch (10s, 30s, 5m).
