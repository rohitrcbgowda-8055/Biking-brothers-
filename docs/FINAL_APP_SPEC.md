# Biking Brothers — Final Mobile App Specification

## Product goal
A group motorcycle touring app for iOS and Android. The existing single-file web app is a prototype; the production app must use native background location, push notifications, and audio so safety features continue when the phone is locked.

## Product decisions
- Platforms: iOS and Android.
- Product name: Biking Brothers.
- Tagline: Connected by Roads. United by Brotherhood.
- Live location: visible to members of the active ride, with clear join/leave and privacy controls.
- Road Captain: fully automatic, based on the rider furthest ahead by progress along the shared route. Ride creator remains Ride Organizer/admin; these are separate roles.
- Voice navigation: ON by default, English initially; add Kannada, Hindi, Tamil, Telugu and Malayalam.
- Automatic separated-rider alert threshold: 25 minutes.
- All ride alerts go to every active ride member, including Road Captain and Ride Organizer.
- SOS is highest priority and must never wait for the 25-minute timer.

## Core modules

### 1. Accounts and ride groups
- Sign up, sign in, password recovery, rider profile and motorcycle details.
- Create ride, join by invite/code, member roster, ride organizer controls.
- Active ride membership and location sharing consent.
- Reconnect and sync missed events after network loss.

### 2. Live group map and navigation
- Live rider markers, shared destination and route, route progress and group spread.
- Turn-by-turn voice prompts, distance-to-turn, ETA and rerouting.
- Bluetooth helmet/intercom audio through native audio session support.
- Screen-locked/background navigation and background GPS permissions.
- Route/voice language and volume settings; repeat instruction and mute controls.
- Offline map regions and cached route/instructions. Clearly indicate when map/routing data is stale.
- Routing provider must be selected and licensed for production; the current prototype's public OSRM/Nominatim endpoints are not a production SLA.

### 3. Automatic Road Captain
- Map-match fresh, accurate rider GPS fixes to the current shared route.
- Compare cumulative along-route progress, not raw latitude/longitude.
- Candidate must be at least 30 metres ahead for 10 continuous seconds before handover.
- Ignore stale fixes and poor accuracy; initial target: fix age <= 15 seconds and horizontal accuracy <= 25 metres.
- Add hysteresis and junction/off-route handling to prevent leadership flapping.
- If no eligible fix exists, retain the last valid captain and show GPS quality status.
- Server-authoritative, atomic captain election; broadcast the change to all riders.
- New captain can publish route changes. Organizer permissions remain separate.

### 4. Group-wide alert centre
Every alert is persisted and delivered to every active ride member. Show sender, alert type, timestamp, location, status and acknowledgements. Retry push delivery and sync missed alerts when a rider reconnects. In-app realtime alone is not sufficient for background delivery.

Alert types:
- SOS / emergency
- Accident
- Breakdown (puncture, engine, battery, other)
- Medical help
- Coffee break
- Fuel stop / low fuel
- Group stop / resume ride
- Rider left behind / separated rider
- Location signal lost
- Wrong route / off-route
- Road hazard / obstruction

Critical alerts (SOS, accident, medical, breakdown, separated rider) receive priority over routine alerts. Voice announcements are available when audio is enabled. Show which members acknowledged the alert and provide Call Rider and Navigate to Rider actions.

### 5. Automatic 25-minute separated-rider safety alert
- During an active ride, determine the moving group's rear reference rider (the second-last rider in route-progress order).
- Start a separation timer when a rider remains materially behind that rear reference rider, using route progress and a configurable minimum gap/distance; do not use straight-line distance alone.
- Default timer: 25 continuous minutes.
- Pause/avoid false positives when the whole group is stationary, GPS is stale/poor, the rider is temporarily off-route due to map-match uncertainty, or the group is stopped together.
- Reset when the rider rejoins the group threshold.
- At 25 minutes, automatically create a high-priority group alert: rider name, last known location, last update time, separation duration, route progress/gap, Navigate to Rider and Call Rider.
- If GPS updates stop, issue a separate Location Signal Lost warning; do not silently assume the rider is safe.
- SOS remains manually available at all times; never require waiting for this timer.
- The precise gap threshold and stationary detection must be tuned in real-world ride tests before release.

### 6. SOS and breakdown flows
- SOS press-and-hold with accidental-trigger guard; immediate alert to all active ride members.
- Include rider identity, live/last-known location, timestamp and call/navigate actions.
- Breakdown flow lets rider choose puncture, engine, battery, accident or other, then sends group alert.
- Alerts remain active until resolved/cancelled with audit history.
- Optional emergency contacts, configured by the rider; share location only with explicit consent.
- SMS fallback may be offered where supported and configured, but must not be represented as guaranteed.

### 7. Stops and group coordination
- Any rider can request Coffee Break, Fuel Stop or Group Stop.
- Everyone receives push + in-app alert; voice announcement where enabled.
- Show proposed stop location and acknowledgements.
- Road Captain/Organizer can mark break started, announce resume countdown and resume ride.
- Any rider can raise a safety request; no rider is forced to stop automatically.

### 8. Ride records and expenses
- Trip history, distance, route and state/UT progress.
- States Conquered India map based only on verified qualifying GPS ride history, with anti-spoofing review.
- Fuel, toll, food, accommodation and general expense entries.
- Per-rider/private expense options and group split summary.

## Production architecture
- Mobile client: React Native with Expo development builds / native modules where needed; evaluate background location and audio requirements on physical iOS and Android devices before choosing managed vs prebuild workflow.
- Backend: existing Supabase project for Auth, Postgres, Realtime and Edge Functions.
- Push delivery: APNs for iOS and FCM for Android, mediated by a trusted server/Edge Function. Never put service-role credentials in the app.
- Server-side scheduled/continuous safety evaluation for 25-minute separation; do not rely only on a foreground phone timer.
- Database migrations with Row Level Security for profiles, rides, ride_members, ride_locations, ride_events, alerts, acknowledgements, route state and captain state.
- Atomic server-side operations for captain election and alert creation to avoid client race conditions.
- Observability: delivery receipts, location freshness, alert retry logs, crash/error reporting and audit trail.
- Keep secrets in server-side environment variables. Existing publishable Supabase key may be embedded in the client only with strict RLS; service-role key must remain server-only.

## Minimum release acceptance tests
1. Two or more iPhones and Android devices can create/join the same ride and see live rider locations.
2. App backgrounded/phone locked: location updates continue within OS permission and battery constraints; UI clearly reports when the OS suspends location.
3. Voice navigation speaks turn instructions through phone and Bluetooth helmet audio with screen locked.
4. Leader election changes only after the configured progress lead and stability window; all devices agree on the same captain.
5. Route change from the current captain reaches every rider and follower route does not overwrite it.
6. SOS and breakdown alerts reach every active member, display acknowledgements, and recover after temporary network loss.
7. Coffee/fuel/stop/resume alerts reach everyone.
8. 25-minute separated-rider test uses simulated GPS tracks and verifies alert timing, no false alert while group is stationary, reset on rejoining, and separate GPS-loss warning.
9. RLS tests prove users cannot read or alter rides/locations/alerts for groups they do not belong to.
10. Test battery use, GPS drift, tunnel/network loss, parallel roads, roundabouts, route deviations and phone restart on physical devices.

## Release status
This document defines the agreed final product scope. It is not evidence that these production features have already been implemented or tested. The current web prototype remains separate until the native mobile app and backend migrations pass the acceptance tests above.
