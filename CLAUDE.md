# HolePunch

A mobile app for iOS, Android, and Apple Watch to control your Pi-hole DNS filter. Check blocking status, pause filtering with one tap, and manage your allowlist and blocklist — all from your wrist or pocket. Built with React Native and a native Watch extension.

## Project Overview

- **Phone app:** React Native CLI (bare workflow) — iOS & Android
- **Watch app:** SwiftUI / WatchKit native Xcode target
- **Phone ↔ Watch bridge:** `react-native-watch-connectivity` (WCSession)
- **Credential storage:** `react-native-keychain`
- **Pi-hole target:** v6 REST API (Core v6.0.6 / FTL v6.1 / Web v6.1)

## Design Documents

All design documents live in `docs/plans/`. Start here:

- [`docs/plans/2026-02-17-pihole-mobile-design.md`](docs/plans/2026-02-17-pihole-mobile-design.md) — full architecture, screen design, data flow, error handling, testing plan, and future directions

## Key Architectural Decisions

- The React Native app is the **sole Pi-hole API client** — the Watch never calls Pi-hole directly
- Sessions span the **app foreground lifetime**, managed via React Native's `AppState` API
  - `AppState → active`: login, begin polling
  - `AppState → inactive`: logout, stop polling
- The session ID (SID) lives **in memory only**, never written to disk
- The **app password** is stored in the device keychain via `react-native-keychain`
- The **server URL** (default: `http://pi.hole`) is stored in device preferences

## Shared Constants

All magic numbers live in `src/config.ts`. Never hardcode durations or defaults elsewhere:

```typescript
export const BLOCKING_PAUSE_SECONDS = 10;
export const STATUS_POLL_INTERVAL_MS = 5000;
export const WATCH_PUSH_INTERVAL_MS = 10000;
export const DEFAULT_SERVER_URL = 'http://pi.hole';
```

## Pi-hole API Endpoints Used

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/auth` | Login → SID |
| DELETE | `/api/auth` | Logout |
| GET | `/api/info/host` | Server hostname |
| GET | `/api/dns/blocking` | Blocking status |
| POST | `/api/dns/blocking` | Disable with timer |
| POST | `/api/domains/allow` | Add to allowlist |
| POST | `/api/domains/deny` | Add to blocklist |
| DELETE | `/api/domains/allow` | Remove from allowlist |
| DELETE | `/api/domains/deny` | Remove from blocklist |

## Testing

Three-layer TDD strategy:

- **Unit tests:** Jest + React Native Testing Library — pure logic, mocked network
- **Integration tests:** Jest against a real Docker Pi-hole (`pihole/pihole:2025.04.0`) on `localhost:8080`
- **E2E tests:** Detox on iOS Simulator and Android Emulator

### Running integration tests locally

```bash
docker run -d --name pihole-test \
  -e FTLCONF_webserver_port=8080 \
  -e FTLCONF_webserver_api_password=testpassword \
  -p 8080:8080 \
  pihole/pihole:2025.04.0

yarn test:integration

docker rm -f pihole-test
```

## File Naming Conventions

| Pattern | Used for |
|---|---|
| `PascalCase.ts` | Classes and React components |
| `camelCase.ts` | React hooks (mirrors the `use` prefix convention) |
| `lowercase.ts` | Plain types, interfaces, and utility files |

All testable UI elements carry a `testID` in `kebab-case-description` format.

## Future Directions

See the design document for full detail. Key deferred features:

- Multi-user with role-based access (app-layer only — Pi-hole has no native user management)
- Recently blocked domain suggestions in the allowlist input (via `GET /api/queries`)
- Server auto-discovery via mDNS/Bonjour
- Watch independence (direct API calls when on home Wi-Fi)
- Configurable pause duration with watch presets
