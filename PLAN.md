# My Abucaynon App: Build Checklist

Tracks progress on [docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md), which has the details. The scope is defined in [docs/SPEC.md](docs/SPEC.md).

How to use: when an item is done (merged, CI green, QA-checked), change `- [ ]` to `- [x]`. Add a date or PR number at the end if useful, e.g. `- [x] Item (#12, 2026-11-03)`.

---

## Prerequisites (outside the code)

- [ ] Confirm the defense date and sprint length (assumed: ~6 months, 2-week sprints)
- [ ] Confirm the monthly budget (assumed: $10–25/month + $99/yr Apple)
- [x] Confirm team members and roles (assumed: 3 people, Larkken on QA)
- [x] Barangay: real list of puroks -> (Purok 1 - 9)
- [x] Barangay: hall office hours -> Monday to Friday (8-12Noon and 1-5pm)
- [ ] Barangay: Kagawad / committee list
- [ ] Barangay: required fields for Clearance, Residency, Indigency
- [ ] Barangay: equipment inventory counts (chairs, tables, tents)
- [ ] Barangay + adviser: privacy policy text, Data Protection Officer name, retention approval
- [ ] Apple Developer account ($99/yr) and Expo account
- [ ] Service accounts: Clerk (dev + prod), Neon, Cloudflare R2, Upstash, Sentry, Railway/Render

---

## Sprint 0: Foundation

### Monorepo

- [ ] Commit the template cleanup currently in the working tree
- [ ] Move the Expo app into `apps/mobile/` (git mv), remove npm lockfile and `reset-project` script
- [ ] Root `package.json` with pnpm + turbo scripts (`dev`, `build`, `lint`, `typecheck`, `test`)
- [ ] `pnpm-workspace.yaml`, `turbo.json`, `tsconfig.base.json` (strict)
- [ ] Verify a single copy of react-native (`pnpm why react-native`) and the app boots from the monorepo

### Shared packages

- [ ] `packages/shared`: permission keys + default role bundles
- [ ] `packages/shared`: status enums + allowed-transition tables
- [ ] `packages/shared`: EN/FIL i18n files + key-parity test
- [ ] `packages/db`: Drizzle setup, `drizzle.config.ts`, migrations folder
- [ ] `packages/db`: local Postgres via docker-compose
- [ ] `packages/db`: seed script (puroks, roles, permissions, document types, categories, committees, inventory, Super Admin)

### API skeleton

- [ ] `apps/api` Express 5 app with Helmet, CORS, pino logging
- [ ] Error middleware + standard error shape
- [ ] `GET /healthz`
- [ ] Vitest + Supertest setup against a test database

### Mobile baseline

- [ ] Install core libraries with `npx expo install` (Clerk, TanStack Query, Zustand, RHF, i18next, Sentry, lucide)
- [ ] NativeWind spike on iOS / Android / web (fallback: StyleSheet tokens, record ADR)
- [ ] Update `app.json`: name, bundle IDs, `web.output` for staff SPA, `web.bundler: metro`
- [ ] Root layout with providers (Query, i18n, Sentry)

### Delivery

- [ ] `eas.json` profiles: development, preview (APK / TestFlight), production
- [ ] GitHub Actions CI: install, lint, typecheck, test (with Postgres service)
- [ ] Husky + lint-staged + Prettier + ESLint
- [ ] `.env.example` for API and mobile; secrets in EAS / host only
- [ ] Deploy API to staging with migrations on deploy
- [ ] Sentry receiving errors from app and API
- [ ] Root `CLAUDE.md` (stack, commands, conventions) + `docs/adr/0001-monorepo.md`

---

## Sprint 1: Auth, consent, profile, permissions

- [ ] Clerk email-code sign-up / sign-in screens (custom, large text)
- [ ] API auth chain: Clerk middleware → load user (lazy upsert) → permission check → verified check
- [ ] `GET /v1/me` (user, profile, role, permissions, verified, consentRequired)
- [ ] Data privacy consent screen + `POST /v1/me/consent` (versioned)
- [ ] Complete profile screen + `PUT /v1/me/profile` + `GET /v1/puroks`
- [ ] Route guard: onboarding → consent → profile → home
- [ ] Pending-verification banner + "Visit the hall to verify" screen
- [ ] `useCan(permission)` hook; staff area hidden for non-staff
- [ ] Language picker on first launch (persisted)
- [ ] Clerk `user.deleted` webhook (scrub PII, revoke tokens)
- [ ] Tests: permission matrix, consent gate, webhook signature
- [ ] QA checklist passed (iOS + Android)

## Sprint 2: Verification, audit, design system

- [ ] Design system components (Button, Text, Card, ListItem, StatusBadge, Timeline, EmptyState, ErrorState, FormField, Screen)
- [ ] Staff web shell (sidebar on desktop, tabs on mobile)
- [ ] `audit()` helper; audit entries on every staff write and on PII views
- [ ] Residents directory (search, filter by purok / verified)
- [ ] Resident detail + Verify action (ID type + last 4 recorded)
- [ ] FIL translations for all existing strings
- [ ] QA checklist passed

## Sprint 3: Push, notifications, announcements

- [ ] Notification permission prompt after onboarding + `POST /v1/devices`
- [ ] APNs / FCM credentials configured via EAS
- [ ] `notify()` outbox + Expo push dispatcher (chunks of 100)
- [ ] Push receipts job (remove invalid tokens)
- [ ] Deep links from push to the right screen
- [ ] Notifications inbox (list, mark read, unread badge)
- [ ] R2 presigned upload helper
- [ ] Announcements manager (create, edit, pin, urgent, image)
- [ ] Urgent announcement broadcast push to all devices
- [ ] Resident announcements feed + detail
- [ ] Verification push now sent when staff verify a resident
- [ ] Tests: chunking, re-dispatch idempotency, token cleanup
- [ ] QA checklist passed

## Sprint 4: Requests (resident side)

- [ ] Document types catalog with `required_fields` definitions
- [ ] Reference number generator `ABC-YYYY-NNNNNN`
- [ ] One-open-request-per-type rule (partial unique index)
- [ ] Idempotency-Key middleware
- [ ] Dynamic request form (purpose + type-specific fields)
- [ ] Submit → reference number confirmation
- [ ] My Requests list
- [ ] Request detail: status timeline + QR code
- [ ] Needs Info: edit and resubmit
- [ ] QA checklist passed

## Sprint 5: Requests (staff side)

- [ ] `transition()` state machine helper (history + audit + notify in one transaction)
- [ ] Requests queue (filters, oldest first, age shown)
- [ ] Process request: allowed actions only, rejection reasons
- [ ] Counter mode (web): search/scan reference → confirm ID → Release
- [ ] Attachments: on-device compression, presigned upload, signed viewing
- [ ] Tests: all valid/invalid transitions, notification + audit per action
- [ ] Maestro flow 2 (request a clearance)
- [ ] QA checklist passed

## Sprint 6: Reports (resident side)

- [ ] Report form: category, description, purok, landmark, hide-identity toggle
- [ ] Optional map pin (`expo-location` + `react-native-maps`)
- [ ] Up to 3 photos, compressed; retry on failed uploads
- [ ] Rate limit 5 reports/day (Upstash); flagged users blocked
- [ ] My Reports list + report detail (timeline, public remarks)
- [ ] QA checklist passed

## Sprint 7: Reports (staff and officials)

- [ ] Committees CRUD + members (Super Admin)
- [ ] Category → default committee mapping
- [ ] Staff reports board (filters) + assign / reassign
- [ ] "Assigned to me" list for officials and Tanod
- [ ] Public remarks vs internal notes
- [ ] Hidden reporter identity enforced in API + audited reveal
- [ ] Close as invalid/duplicate + flag user
- [ ] Tests: identity hiding, Tanod sees only assigned reports
- [ ] Maestro flow 3 (file a report)
- [ ] QA checklist passed

## Sprint 8: Court bookings

- [ ] Migration: `btree_gist` + court exclusion constraint (incl. blocked periods)
- [ ] Map conflict error (23P01) to `BOOKING_CONFLICT`
- [ ] `GET /v1/court/availability`
- [ ] Calendar: month view → day view with hourly slots (6:00–22:00)
- [ ] Select consecutive slots + booking form
- [ ] Settings-driven limits (1–60 days ahead, max 2 active, hours)
- [ ] Staff block dates / slots
- [ ] Test: 50 concurrent bookings for one slot → exactly 1 succeeds
- [ ] QA checklist passed

## Sprint 9: Equipment, holds, booking staff tools

- [ ] Inventory CRUD (staff)
- [ ] Equipment booking: date range + item lines
- [ ] Allocation with row locks (no over-allocation)
- [ ] Pending holds inventory; 48h auto-expire job (advisory lock)
- [ ] Resident cancel (≥24h before start)
- [ ] My Bookings list + booking detail
- [ ] Staff bookings calendar: approve / reject / picked up / returned / completed
- [ ] Test: concurrent over-allocation, 48h expiry
- [ ] Maestro flow 5 (book court / rent equipment)
- [ ] QA checklist passed

## Sprint 10: Lost & Found and Activities

### Lost & Found

- [ ] Board with Lost/Found tabs, search, category/date filters
- [ ] Post item (one photo)
- [ ] "This is mine" claim → staff queue → Returned (no PII shared)
- [ ] Moderation: hide/remove with reason
- [ ] Daily job: warn at 27 days, archive at 30, delete photos 90 days after archive
- [ ] Renew once
- [ ] Maestro flow 4 (lost & found)

### Activities

- [ ] Activities manager (staff CRUD)
- [ ] Resident list + month calendar + detail
- [ ] "Remind me" toggle
- [ ] Daily reminder job (08:00 Asia/Manila, for next-day activities)
- [ ] QA checklist passed

## Sprint 11: Analytics, roles UI, hardening

- [ ] Analytics views: request turnaround, report resolution, booking counts, weekly active users
- [ ] Staff analytics dashboard (web) + CSV export
- [ ] Audit log viewer (Super Admin)
- [ ] Roles & users UI (edit bundles, assign roles, last-Super-Admin guard)
- [ ] Settings screen (booking limits, hours)
- [ ] Home dashboard finalized (6 tiles, announcements, my active items)
- [ ] Profile: delete account, privacy policy, help/user guide, about
- [ ] Accessibility pass (200% text, TalkBack, VoiceOver, contrast)
- [ ] Full Filipino review by a native speaker
- [ ] k6 load test (10k users seeded, p95 < 500ms on staging)
- [ ] Security review (auth on every route, IDOR tests, rate limits, secrets)
- [ ] QA checklist passed

## Sprint 12: Release and evaluation

- [ ] Demo seed data across all statuses
- [ ] Maestro suite for flows 1–7 passing
- [ ] Android APK (EAS preview) distributed
- [ ] iOS TestFlight build distributed to testers
- [ ] SUS evaluation with residents
- [ ] SUS evaluation with officials/staff
- [ ] SUS scores computed and written up
- [ ] Bug bash + fixes (EAS Update channel ready)
- [ ] Sentry: no unresolved crash groups
- [ ] Defense script + backup demo video
