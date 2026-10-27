# CLAUDE.md: My Abucaynon App

@AGENTS.md

"Every barangay service in one tap": a capstone project (ACLC College Ormoc City). It's a mobile-first Barangay Management System for residents and officials of **Brgy. Abucay, Tacloban City, Leyte**.

## Source of truth: read before working
- [docs/SPEC.md](docs/SPEC.md): approved v1 scope, roles and permission matrix, feature rules, data model, assumptions, and risks.
- [docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md): conventions plus the technical design for each sprint.
- [PLAN.md](PLAN.md): the progress checklist.
  - **When you finish a task, change its `- [ ]` to `- [x]` in PLAN.md.** Only tick an item when it's done; never tick ahead.
  - Work only on what the user asks for, in sprint order unless told otherwise.

If a request conflicts with SPEC.md (for example, something listed as out of scope), say so and ask before building it. Don't silently swap any tool in the stack below; propose the change and let the user decide.

## Current state vs target
- **Now:** a single Expo SDK 58 app at the repo root, installed with npm (`package-lock.json`).
- **Target** (PLAN.md Sprint 0): a pnpm + Turborepo monorepo.
  ```
  apps/mobile      Expo app (routes in src/app/, features in src/features/<module>/)
  apps/api         Express 5 REST API (/v1), modules/<module>/{routes,service,repo}.ts
  packages/db      Drizzle schema, migrations, seed
  packages/shared  Zod schemas, status enums + transition tables, permission keys, i18n (en/fil)
  ```
  Until the restructure is done, use npm and `npx expo …` at the root.

## Stack (pinned decisions)
- **Language:** TypeScript strict everywhere.
- **App:**
  - Expo SDK 58, React Native 0.88, React 19.3, Expo Router.
  - Styling: NativeWind 4 with Tailwind 3; use a StyleSheet token system instead if the Sprint 0 spike fails.
  - Data: TanStack Query, Zustand, React Hook Form + Zod.
  - i18next (EN + FIL), lucide-react-native.
- **Auth:** Clerk, using **`@clerk/expo`** (not the old `@clerk/clerk-expo`) and `@clerk/express`.
  - Sign-in is an **email code**. Phone OTP is deferred because Clerk's free plan has no SMS.
- **API:** Node + Express 5, Zod validation, Helmet, CORS, Upstash Redis rate limits.
- **Data and infrastructure:**
  - PostgreSQL on Neon, with Drizzle ORM + Drizzle Kit.
  - Cloudflare R2 for files (presigned uploads).
  - Expo Push via expo-notifications.
  - Sentry on the app and the API.
- **Background jobs:** no BullMQ in v1. Jobs are cron functions guarded by Postgres advisory locks.
- **Delivery:** EAS Build/Update with an Android APK and iOS TestFlight. No store publishing in v1.

## Architecture rules
- **Staff/admin tools** live in the same Expo app under a role-gated `(staff)` route group. That group is also exported to web for the hall desktop.
- **Permissions:** keys are fixed in `packages/shared/permissions.ts`, and roles are editable bundles of those keys stored in the database.
  - The API checks the permission **and** the resident's verified status on every call. Our database is the source of truth, never Clerk metadata.
  - System roles (Resident, Super Admin) can't be deleted, and the last Super Admin can't be demoted.
- **Business logic lives in the API service layer.** The client only reflects state.
- **Status changes** go only through `transition()` (`apps/api/src/lib/state-machine.ts`), which validates against `TRANSITIONS` in packages/shared. In one database transaction it writes the history row, the audit entry, and the notification.
- **Audit:** every staff/admin write calls `audit()` in the same transaction. Viewing resident PII or a hidden reporter's identity also writes an audit entry.
- **Double booking is prevented in Postgres, not just in the UI.**
  - Court: a `btree_gist` exclusion constraint on `tstzrange`. Blocked periods are rows covered by the same constraint.
  - Equipment: `SELECT … FOR UPDATE` on inventory rows (locked in id order), then a quantity check.
  - SQLSTATE `23P01` maps to `BOOKING_CONFLICT`.
- **Create endpoints** take an `Idempotency-Key` header. Lists use cursor pagination.
- **Errors** use the shape `{ error: { code, message_key, details? } }`. The app translates `message_key`, so never hard-code English error text.
- **Status freshness:** push notification plus a TanStack Query refetch on focus, foreground, or pull-to-refresh. No websockets and no polling.

## Domain rules (quick reference; details in SPEC.md)
- **Unverified residents** can only browse announcements, activities, and lost & found. Every submit action sends them to a "Visit the hall to verify" screen. Verification happens in person, and staff tick Verified.
- **Documents (v1):** Barangay Clearance, Certificate of Residency, Certificate of Indigency.
  - No fees in the app.
  - Reference number format: `ABC-YYYY-NNNNNN`.
  - One open request per document type per resident.
- **Reports:**
  - Up to 3 photos; purok required; map pin optional.
  - The reporter can hide their name from the assigned official (staff still see it).
  - Limit of 5 reports per day.
  - Assignment goes by committee, and Tanod see only reports assigned to them.
- **Lost & Found:** claims are staff-mediated and residents never see each other's PII. Posts auto-archive after 30 days and can be renewed once.
- **Bookings:**
  - Court in 1-hour slots from 6:00–22:00.
  - Equipment by whole days, picked up at the hall.
  - Pending bookings hold the slot and expire after 48h.
  - Book 1–60 days ahead; cancel at least 24h before.
  - At most 2 active court bookings and 1 active equipment booking per resident.
- **Activities:** a "Remind me" toggle sends a push the day before. No RSVP.
- **Privacy (RA 10173):** versioned consent before the profile is created. Deleting an account scrubs PII and keeps records under a pseudonymous ID.
- **Barangay facts:**
  - Puroks: **Purok 1–9**.
  - Hall hours: **Mon–Fri, 8:00–12:00 and 13:00–17:00**.
  - Timezone: **Asia/Manila**.

## UI / UX rules
- **Bilingual:** every user-facing string goes through i18next keys and must exist in both `en` and `fil`. Waray-Waray comes later, so nothing should be hard-coded.
- **Built for seniors and low digital literacy:**
  - Respect system font scaling; minimum 48dp tap targets; WCAG AA contrast.
  - Accessibility labels on everything; icons always paired with text.
- **Weak connections:** paginated lists, compressed images (around 1280px, JPEG at about 70%), upload retry, and clear error, empty, and loading states.

## Commands
```bash
# now (single app, npm)
npx expo start
npx expo lint
npx tsc --noEmit
npx expo install <pkg>        # never npm install for app deps

# after the Sprint 0 monorepo restructure (pnpm)
pnpm dev | pnpm lint | pnpm typecheck | pnpm test   # via turbo
pnpm --filter mobile exec expo install <pkg>
pnpm --filter db exec drizzle-kit generate          # then migrate
```
Run lint and typecheck before saying a task is done, and run tests once they exist.

## Workflow
- Branch from `main` and use Conventional Commits. Commit or push only when the user asks.
- Never put secrets in git. Env vars go in `.env.example`; real values live in EAS and the host's environment settings.
- Before using an Expo, Clerk, Drizzle, or Sentry API, check its current docs (see AGENTS.md). Don't rely on memory.
- Team: 3 members, with **Larkken on QA**. Each sprint ends with Larkken's QA checklist on Android and iOS builds.
