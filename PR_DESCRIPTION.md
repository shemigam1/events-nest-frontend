# fix: check-in scanner — 4 correctness bugs + multi-day day picker

## Summary

The `/checkin` route and scanner UI existed but had four bugs that caused every scan to fail in practice. This PR fixes all of them and adds first-class multi-day event support with an automatic day picker.

---

## Bugs fixed

### 1. Wrong field name in scan API — blocked every single scan
`checkinApi.js` was sending `qrCode` in the request body. The backend's `CheckInRequest` field is `ticketCode`. Spring silently ignores unknown fields, so `ticketCode` was always `null`, triggering `@NotBlank` validation on every request.

```js
// before — 400 on every scan
body: { staffToken, qrCode, shortCode: qrCode }

// after
body: { staffToken, ticketCode, ...(eventDayId ? { eventDayId } : {}) }
```

### 2. No `eventDayId` ever sent — broke all multi-day events
Neither scanner page included `eventDayId` in the scan body. For multi-day events the backend throws `"eventDayId is required for multi-day events"`. Fixed by adding a day picker (see below).

### 3. Response field mismatches — blanked out success cards
Both scanner pages read `attendeeFirstName`, `attendeeLastName`, and `seatNumber` from the scan response. The backend `CheckInResponse` returns `holderName` (combined), `seatLabel`, and `tierName`. Every successful scan showed an empty admission card.

| Before (wrong) | After (correct) |
|----------------|----------------|
| `d.attendeeFirstName + ' ' + d.attendeeLastName` | `d.holderName` |
| `d.seatNumber` | `d.seatLabel` |

### 4. Import path case bug — crashed on Linux
`CheckInPage.jsx` imported from `'../checkInApi'` (capital `I`). The file is `checkinApi.js` (lowercase). Case-insensitive on Windows, fatal on any Linux deployment.

```js
// before
import { useScanTicketMutation } from '../checkInApi';

// after
import { useScanTicketMutation } from '../checkinApi';
```

---

## Multi-day day picker

`EventInfoScreen` now reads `event.eventDays` from the `EventResponse` (already returned by the existing `useGetEventByIdQuery` call — no new API needed).

**Single-day events** — no UI change. The day is resolved automatically by the backend.

**Multi-day events:**
- A pill-button row renders: `Day 1 · Day 2 · Day 3`
- Today's day is **auto-selected** by matching `dayDate` to today's ISO date
- The check-in window status (open / not yet open) respects the selected day's `checkInStartTime` override, falling back to the event-level time
- **"Start scanning tickets"** is disabled until a day is selected
- The selected `eventDayId` is passed forward to `ActiveSession` and included in every scan call
- A sticky `DayBanner` at the top of the active session lets staff switch days mid-session without ending the session

---

## Other

### `CheckInScannerPage.jsx` — routed and fixed
This page existed but had no registered route. Added `/checkin/:eventId`. Also applied the same field name fixes and day picker so both scanner entry points behave consistently.

```jsx
// routes/index.jsx — added
{ path: '/checkin/:eventId', element: <CheckInScannerPage /> },
```

### Short code support
The backend's `resolveTicket` already accepts all three input formats via a single `ticketCode` field:

| Input | Resolved by |
|-------|-------------|
| Signed payload (Ed25519 QR) | `QrPayloadCodec.decode` + signature verification |
| UUID QR code | `findByQrCode` |
| 8-char short code (e.g. `JAZZ-0001`) | `findByShortCode` |

Staff can type or scan any of the three — the backend figures out which format it is.

---

## Files changed

| File | Change |
|------|--------|
| `checkinApi.js` | Fixed field name `qrCode → ticketCode`, added `eventDayId` param |
| `CheckInPage.jsx` | Fixed import, response fields, added day picker to `EventInfoScreen` + `DayBanner` in `ActiveSession` |
| `CheckInScannerPage.jsx` | Fixed response fields, added day selector, added event days query |
| `routes/index.jsx` | Registered `/checkin/:eventId` route for `CheckInScannerPage` |
