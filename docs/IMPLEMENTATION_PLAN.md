# My Abucaynon App: Implementation Plan (v1)

Builds on [SPEC.md](./SPEC.md). Sprint 0 is planned file by file. Later sprints list tasks, the main technical design, tests, and the QA checklist. Before writing code that touches an Expo, Clerk, Drizzle, or Sentry API, re-check the versioned docs (see AGENTS.md).

---

## 0. Conventions (apply in every sprint)

**Repo layout**
```
/
├─ apps/
│  ├─ mobile/              Expo SDK 58 app (moved from repo root)
│  │  └─ src/
│  │     ├─ app/           routes only (Expo Router)
│  │     ├─ features/<module>/{api.ts, hooks.ts, components/, screens/}
│  │     ├─ components/ui/ design-system primitives
│  │     ├─ lib/           api client, query client, i18n, sentry, push
│  │     └─ stores/        zustand (language, onboarding flags)
│  └─ api/
│     └─ src/
│        ├─ app.ts         express app (no listen; used by Supertest)
│        ├─ server.ts      listen + Sentry init
│        ├─ middleware/    auth, loadUser, requirePermission, requireVerified, validate, idempotency, rateLimit, error
│        ├─ modules/<module>/{routes.ts, service.ts, repo.ts}
│        ├─ jobs/          cron jobs (expire bookings, archive L&F, reminders, push receipts)
│        └─ lib/           audit, notify (outbox + expo push), storage (R2 presign), state machine
├─ packages/
│  ├─ db/                  drizzle schema, migrations, seed, client
│  └─ shared/              zod schemas, enums, transitions, permissions, i18n json, constants
├─ docs/
├─ pnpm-workspace.yaml  turbo.json  tsconfig.base.json  eslint.config.js  .prettierrc
```

**Rules**
- **Validation and types:** every API input is validated with a Zod schema from `packages/shared`. The app uses the same schemas with React Hook Form via `zodResolver`.
- **Status changes:** done only by `transition(entity, from, to, actor, reason?)` in `apps/api/src/lib/state-machine.ts`. That function:
  - validates the change against the allowed-transition tables in `packages/shared/src/status.ts`,
  - writes the history row,
  - writes the audit log,
  - enqueues the notification.

  All four writes happen in one database transaction.
- **Staff writes:** every staff/admin write calls `audit()` inside the same transaction as the change.
- **API errors:** returned as `{ error: { code, message_key, details? } }`. The app translates `message_key` with i18next, so error text is never hard-coded in English.
- **Create endpoints:** every one accepts an `Idempotency-Key` header. The key is stored in Redis for 24h along with the response.
- **Lists:** paginated with a cursor (`?cursor=&limit=20`).
- **Commits:** Conventional Commits. Each piece of work lands as a PR into `main`, and CI must pass.
- **Package installs:** inside `apps/mobile`, use `npx expo install`. Elsewhere, use `pnpm add`.

---

## Sprint 0: Foundation (repo, tooling, environments)

### 0.1 Monorepo restructure
1. Move the Expo app into `apps/mobile/` with `git mv`:
   - Files: `src/`, `assets/`, `app.json`, `tsconfig.json`, `package.json`, `README.md`.
   - Delete `package-lock.json` and `node_modules/`.
   - Remove the `reset-project` script.
2. Root files:
   - `package.json`: name `abucaynon`, `private`, `packageManager: pnpm@<current>`, scripts `dev`, `build`, `lint`, `typecheck`, `test` that call turbo.
   - `pnpm-workspace.yaml`: `packages: ['apps/*', 'packages/*']`. Keep the isolated linker. Switch to `nodeLinker: hoisted` only if a native module fails to resolve (Expo monorepo guide).
   - `turbo.json`: tasks `build`, `lint`, `typecheck`, `test`, `dev` (`persistent`, `cache: false`).
   - `tsconfig.base.json`: `strict`, `noUncheckedIndexedAccess`, `moduleResolution: bundler`.
3. Metro: no custom config. SDK 52+ configures monorepos automatically, so only a `metro.config.js` is added if NativeWind needs one.
4. Verify there's a single React Native copy with `pnpm why --depth=10 react-native`, then confirm `pnpm --filter mobile start` boots on a device.

### 0.2 packages/shared
- `src/permissions.ts`: the permission keys listed in SPEC §1, as a const tuple plus a `Permission` type and `DEFAULT_ROLE_BUNDLES`.
- `src/status.ts`: status enums and `TRANSITIONS` maps for requests, reports, bookings, lost & found items, and claims.
- `src/schemas/`: Zod schemas, added one module per sprint.
- `src/i18n/en.json` and `src/i18n/fil.json`, plus a key-parity test.
- Build: TypeScript source consumed directly. Point `"exports"` at `src/index.ts`; Metro and tsx can both read TypeScript source.

### 0.3 packages/db
- Drizzle with `drizzle-orm/neon-serverless` (pooled) or `node-postgres`. Recommended: `node-postgres` against the Neon pooled URL, which works locally and on Railway/Render.
- `drizzle.config.ts` and a `migrations/` folder. Raw SQL migrations handle:
  - `CREATE EXTENSION btree_gist`
  - the exclusion constraints
  - the reference-number sequence
- `src/seed.ts`:
  - puroks (placeholder list until the barangay supplies the real one)
  - roles and permissions from `DEFAULT_ROLE_BUNDLES`
  - document types, report categories, committees, inventory items
  - a demo Super Admin bound to `SEED_SUPERADMIN_CLERK_ID`
- Local database: Docker `postgres:17` through `docker-compose.yml`. Shared databases: Neon branches `staging` and `prod`.

### 0.4 apps/api skeleton
- **Dependencies:** express 5, helmet, cors, @clerk/express, zod, @upstash/ratelimit and @upstash/redis, @sentry/node, pino.
- **Endpoints:** `GET /healthz`, `GET /v1/me` (stubbed until Sprint 1).
- **Errors:** an error middleware that maps Zod and domain errors to the error shape above.
- **Tests:** Vitest and Supertest against `app.ts`, using a test database from Docker (CI uses a Postgres service container).

### 0.5 apps/mobile baseline
- Install with `npx expo install`:
  - core: `nativewind tailwindcss@3 react-native-reanimated`
  - Clerk: `@clerk/expo expo-secure-store expo-web-browser expo-auth-session expo-crypto`
  - data and forms: `@tanstack/react-query zustand react-hook-form @hookform/resolvers`
  - i18n: `i18next react-i18next expo-localization`
  - monitoring: `@sentry/react-native`
  - icons: `lucide-react-native react-native-svg`
- **NativeWind spike (timeboxed to 1 day):** get a `className` rendering on iOS, Android, and web under SDK 58 / RN 0.88. If it fails, fall back to a typed `StyleSheet` token system and record the decision in `docs/adr/`.
- **app.json changes:**
  - `web.output: "single"` for the staff SPA export (re-check the options in the SDK 58 config docs).
  - `web.bundler: "metro"`.
  - Proper name, slug, bundle identifier, and Android package name.
  - Sentry config plugin.
- **Starting screens:** `app/_layout.tsx` with QueryClientProvider, ClerkProvider (Sprint 1), I18nextProvider, and a Sentry wrap, plus a placeholder home screen.

### 0.6 Delivery
- `eas.json` profiles:
  - `development`: dev client
  - `preview`: Android APK and iOS internal build for TestFlight
  - `production`
  - Every profile sets `EXPO_PUBLIC_API_URL` and the Sentry environment.
- **GitHub Actions** (`ci.yml`): pnpm install with cache, then `turbo lint typecheck test`, with a Postgres service.
- **Husky and lint-staged:** Prettier and ESLint on staged files.
- **Environments:** `.env.example` for the API and mobile app. Secrets live in EAS and the host's environment settings, never in git.
- **API hosting:** Railway or Render for staging and prod. A deploy runs on merge to main, with a migration step (`drizzle-kit migrate`) before the API starts.
- **Docs:** `CLAUDE.md` at the root covering stack, commands, conventions, and the "re-check Expo docs" rule (merged with the existing AGENTS.md). Plus `docs/adr/0001-monorepo.md`.

**Done when:**
- CI is green.
- The app boots on an Android device, an iOS device, and in a web browser from the monorepo.
- `/healthz` is deployed to staging.
- A test error appears in Sentry from both the app and the API.

---

## Sprint 1: Auth, consent, profile, permissions
**API**
- Middleware chain, in order:
  1. `clerkMiddleware()`
  2. `requireAuth`
  3. `loadUser`: lazy-upserts the `users` row by `clerk_user_id` and loads the role's permissions. Cached in Redis for 60s and invalidated when the role changes.
  4. `requirePermission(key)`
  5. `requireVerified`, where needed.
- **Endpoints:**
  - `GET /v1/me` returns the user, profile, role, permissions, `verified`, and `consentRequired`.
  - `POST /v1/me/consent`
  - `PUT /v1/me/profile`
  - `GET /v1/puroks`
- **Clerk webhook** `user.deleted`: scrub PII, set `deleted_at`, revoke device tokens. Verified with svix.

**Mobile**
- Clerk email-code sign-in and sign-up with custom screens (larger text than Clerk's built-ins).
- Route guard with Expo Router protected routes, which picks the screen from `/me`:
  - not signed in → onboarding
  - consent required → consent screen
  - no profile → complete profile
  - otherwise → the home screen
- Pending-verification banner, plus a `useCan(permission)` hook. The `(staff)` group is hidden unless the user has any staff permission.
- Language picker on first launch, persisted with zustand and SecureStore/AsyncStorage.

**Tests**
- Permission middleware matrix test: every seeded role × a sample of endpoints.
- Consent gate.
- Webhook signature rejection.

**QA**
- Sign up, sign in, and sign out on iOS and Android.
- A new policy version re-prompts consent.
- An unverified user sees the verify prompt on protected actions.

## Sprint 2: Verification, audit, design system, i18n
- **Staff:** residents directory (search by name or purok, filter by verified), resident detail, and a "Verify" action that requires the ID type and the last 4 characters of the ID number to be entered for the audit trail. The resident gets a push notification (fully wired in Sprint 3).
- **Audit:** `audit(tx, {actor, action, entity, before, after})`. Viewing a resident's PII logs a `view` entry.
- **Design system:** Button, Text (scaled), Card, ListItem, StatusBadge, Timeline, EmptyState, ErrorState, FormField, and Screen (safe area plus pull-to-refresh).
- **Staff web shell:** a responsive sidebar layout at ≥1024px and tabs below that.
- **i18n:** FIL translations for every string written so far, plus a CI test that EN and FIL have the same keys.

## Sprint 3: Push infrastructure, notifications, announcements
**Push and notifications**
- **Device registration:** `expo-notifications` permission is requested after onboarding (not on first launch), then `POST /v1/devices` registers the token. Configure APNs and FCM through EAS credentials.
- **`notify(tx, userIds, type, entityRef, i18nKeys)`:** inserts `notifications` rows inside the transaction. After commit, a dispatcher sends them to Expo Push in chunks of 100 and stores the ticket ids.
- **Push receipts job:** checks receipts every 15 minutes and deletes `DeviceNotRegistered` tokens.
- **Deep links:** the push `data.url` (for example `/requests/<id>`) is handled with a notification-response listener and `router.push`.
- **Inbox:** list, mark as read, and an unread badge.

**Announcements**
- Staff can create, edit, pin, and mark urgent. The image is uploaded through an R2 presigned PUT.
- Urgent announcements fan out to all tokens. Use a `broadcast` notification type with one row, and send push to tokens directly, so the database doesn't get 10k rows.
- Residents get a feed (pinned first) and a detail screen.

**Tests:** chunking, idempotent re-dispatch, and invalid token cleanup (mock Expo).

## Sprint 4: Requests (resident side)
- **Reference numbers:** sequence `request_ref_seq`, formatted `ABC-YYYY-NNNNNN`. The per-year reset is handled by a `(year, counter)` table updated with `UPDATE … RETURNING` inside the transaction.
- **Catalog:** comes from `document_types.required_fields` (JSON field definitions). The form renders those fields dynamically and validates them with a Zod schema built at runtime.
- **Uniqueness:** a partial unique index on `(user_id, document_type_id) WHERE status NOT IN ('released','rejected')` enforces one open request per type.
- **Screens:** catalog → form → submit (idempotency key) → detail with the timeline and a QR code of the reference number (`react-native-qrcode-svg`) → My Requests list.
- **Needs Info:** the resident edits the flagged fields and resubmits.

## Sprint 5: Requests (staff side) and attachments
- **Queue:** filterable by status and type, oldest first, with SLA age shown.
- **Processing:** buttons for allowed transitions only, from the `TRANSITIONS` table. Reason presets for rejection.
- **Counter mode (web):** search or scan a reference number → confirm ID → Release.
- **Attachments:**
  - Presigned PUT with a size limit and a content-type allowlist.
  - Images are compressed with `expo-image-manipulator` before upload.
  - Viewing an attachment uses a signed GET with a 5-minute expiry.

**Tests:** every valid and invalid transition, notification on each status change, audit on each staff action, and Maestro flow 2.

## Sprint 6: Reports (resident side)
- **Form:**
  - Category, description, purok (required), and an optional pin (`expo-location` + `react-native-maps`; the pin is optional if permission is denied).
  - Landmark text, up to 3 photos, and the hide-identity toggle.
- **Upload flow:** the report is created first, then its photos. If photo uploads fail, the report stays valid and the app shows a "retry photos" button.
- **Rate limit:** Upstash sliding window of 5 per day per user. Flagged users get a 403.
- **Screens:** My Reports, and a report detail with the timeline and public remarks.

## Sprint 7: Reports (staff and officials)
- **Committees:** CRUD (Super Admin only) and member assignment.
- **Staff:** a reports board filtered by category, purok, and status, with assign and reassign to members of the category's committee.
- **Officials and Tanod:** an "Assigned to me" list. Public remarks and internal notes go in separate tabs.
- **Hidden identity:** the response omits reporter fields unless the caller has `reports.view_reporter_identity`, and showing the identity writes an audit entry.
- **Moderation:** Close as invalid or duplicate with a reason, and flag the user.

## Sprint 8: Court bookings
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings ADD CONSTRAINT court_no_overlap
  EXCLUDE USING gist (resource_key WITH =, time_range WITH &&)
  WHERE (kind = 'court' AND status IN ('pending','approved','in_use','blocked'));
-- staff blocks are bookings rows with status='blocked', so the same constraint covers them
```
- **Blocked periods:** stored in the same table (`status = 'blocked'`, included in the constraint's WHERE), so the constraint enforces them too.
- **Overlap errors:** an exclusion violation (SQLSTATE 23P01) maps to `BOOKING_CONFLICT`, and the app refreshes the calendar.
- **Availability:** `GET /v1/court/availability?from&to` returns the slots taken in the range.
- **Calendar UI:** a month view → a day view with 16 hourly slot buttons (free/taken/blocked/mine). Consecutive slots can be selected.
- **Settings checks:** the 1–60 day window, at most 2 active court bookings, and hours from `settings`.

**Tests:** 50 parallel booking requests for the same slot, exactly 1 succeeds (Vitest with concurrent transactions).

## Sprint 9: Equipment, holds, and booking staff tools
**Allocation**, in one transaction:
1. `SELECT … FROM inventory_items WHERE id = ANY($ids) ORDER BY id FOR UPDATE`. Locking in a fixed order avoids deadlocks.
2. Sum the quantities on overlapping active lines (`date_range && $range`).
3. Reject the whole booking if any item would exceed `total_owned`.

**Jobs**
- `expirePendingBookings` runs every 15 minutes: Pending older than 48h → Expired, with a notification.
- Every job uses a Postgres advisory lock so multiple API instances can't run it at the same time.

**Staff tools**
- A bookings calendar with approve/reject and Picked Up → Returned.
- Inventory CRUD.
- Block-dates UI.

**Cancellation:** residents can cancel only if the start is more than 24h away.

**Tests:** concurrent over-allocation and the 48h expiry.

## Sprint 10: Lost & Found and Activities
**Lost & Found**
- Board with Lost/Found tabs, search (Postgres `ILIKE` with a trigram index), and category and date filters.
- Post with one photo.
- Claim → staff queue → mark Returned. The poster is notified without any claimant PII.
- Jobs:
  - `archiveLostFound` runs daily: warns at 27 days, archives at 30, and deletes photos 90 days after archiving.
  - Renew once (`renewed_at` must be null).
- Moderation: hide or remove with a reason.

**Activities**
- Staff CRUD, plus a list and a month calendar.
- A "Remind me" toggle.
- `activityReminders` runs daily at 08:00 Asia/Manila for activities starting the next day.

## Sprint 11: Analytics, roles UI, hardening
**Analytics**
- SQL views:
  - request turnaround (`submitted` → `ready_for_pickup`, `→ released`), as median and p90 by document type
  - report resolution time by category
  - booking counts by resource and month
  - weekly active users (a `last_active_at` column updated at most once per day)
- Charts on the staff web, and CSV export.

**Roles UI** (Super Admin only)
- Edit role bundles and assign roles.
- Guards: system roles can't be deleted, and the last Super Admin can't be demoted.

**Accessibility and polish**
- Accessibility pass: test at 200% font scale, with TalkBack and VoiceOver, and check contrast.
- Full FIL review by a native speaker.
- Help guide screens.

**Load and security**
- k6: a 10k-user dataset seeded, a 300-VU run on the read-heavy endpoints, and a 50-VU run on submit flows. Target p95 < 500ms on staging.
- Security review:
  - OWASP API Top 10 checklist: authorization on every route, IDOR tests (a resident fetching another resident's request gets 404), rate limits.
  - Secrets audit.

## Sprint 12: Release and evaluation
- Demo seed data (`pnpm db:seed:demo`) with realistic records across every status.
- Maestro E2E suite covering flows 1–7. Run on Android in CI if time allows, otherwise run locally and record the results.
- EAS `preview` builds: an APK link and TestFlight internal testers.
- SUS sessions with residents and officials. Use a printed or Google Form questionnaire, and keep the results outside the app.
- Bug bash, an EAS Update channel for last-minute fixes, the defense script, and a backup recorded demo video.

---

## Spike / prerequisite checklist (owner → by sprint)
| Item | Needed by |
|---|---|
| Barangay: real puroks list, hall hours, committee and Kagawad list, inventory counts, document field requirements | S2 (puroks), S4 (documents), S7 (committees), S9 (inventory) |
| Barangay and adviser: privacy policy text, DPO name, retention approval | S1 (draft), S11 (final) |
| Apple Developer account ($99/yr) and Expo account; APNs key via EAS | S3 |
| Clerk app (dev and prod instances), Neon project, R2 bucket, Upstash, Sentry projects, Railway/Render | S0 |
| Confirm timeline, budget, and team roles (spec assumptions 1–3) | Before S1 |

## Verification (whole project)
- Every PR passes `pnpm turbo lint typecheck test`. For the mobile app, that includes `npx expo lint` and `tsc --noEmit`.
- Each sprint ends with an APK and a TestFlight build, and Larkken runs that sprint's QA checklist on both.
- Concurrency tests (S8, S9) and permission matrix tests (S1, S7) are required in CI.
- Before the defense: k6 report, Maestro run, Sentry dashboard with no unresolved crash groups, and the SUS score computed.
