# My Abucaynon App: v1 Spec (interview outcome)

> "Every Barangay 91 service in one tap": Capstone, ACLC College Ormoc City.
> Approved v1 spec. The implementation plan is in docs/IMPLEMENTATION_PLAN.md.

## Context

Brgy. 91 Abucay, Tacloban City runs clearances, complaints, lost & found, rentals, and announcements on paper, through walk-ins, and by word of mouth. That means repeat trips, lost records, double bookings, and uneven reach. v1 digitizes six service areas in one Expo app for residents. Staff tools sit in the same codebase and are exported to web for the hall desktop. The app also captures the timing and usage data the capstone needs to answer its research questions and run a SUS evaluation.

**The one thing v1 must do well:** a verified resident can submit a request, report, or booking from home and see its status change without visiting the hall, except for the final pickup or handover.

**Repo today:** a single Expo SDK 58 template at the repo root, installed with npm. It must be restructured into a pnpm + Turborepo monorepo (decided).

---

## 1. Users & roles

Roles are **editable bundles of fixed permission keys**. Keys are defined in code (`packages/shared/permissions.ts`); roles and role↔permission links live in the database. Built-in roles **Resident** and **Super Admin** are locked, and the last Super Admin can't be demoted. Every role change is audit-logged.

Everyone signs up as a Resident, and the Captain or Super Admin promotes staff. The first Super Admin is created by a seed script.

| Seeded role                           | Who                                                 |
| ------------------------------------- | --------------------------------------------------- |
| Resident (unverified / verified flag) | any signed-up user; `verified_at` set by staff      |
| Staff                                 | Secretary, Clerk                                    |
| Official                              | Kagawad, committee head                             |
| Tanod                                 | peace and order; sees only reports assigned to them |
| Super Admin                           | Captain                                             |

### Permission keys (default bundles)

| Permission                                                                                           | Res-unverified | Res-verified | Staff | Official | Tanod | Super Admin |
| ---------------------------------------------------------------------------------------------------- | -------------- | ------------ | ----- | -------- | ----- | ----------- |
| announcements.read / activities.read / lostfound.read                                                | ✓              | ✓            | ✓     | ✓        | ✓     | ✓           |
| requests.create, reports.create, bookings.create, lostfound.post, lostfound.claim, activities.remind | –              | ✓            | ✓     | ✓        | ✓     | ✓           |
| requests.process                                                                                     | –              | –            | ✓     | –        | –     | ✓           |
| residents.verify, residents.view                                                                     | –              | –            | ✓     | –        | –     | ✓           |
| reports.view_all, reports.assign                                                                     | –              | –            | ✓     | –        | –     | ✓           |
| reports.handle_assigned                                                                              | –              | –            | –     | ✓        | ✓     | ✓           |
| reports.view_reporter_identity (when hidden)                                                         | –              | –            | ✓     | –        | –     | ✓           |
| bookings.process, inventory.manage, blocked_dates.manage                                             | –              | –            | ✓     | –        | –     | ✓           |
| lostfound.moderate, lostfound.process_claims                                                         | –              | –            | ✓     | –        | –     | ✓           |
| announcements.manage, announcements.push, activities.manage                                          | –              | –            | ✓     | ✓        | –     | ✓           |
| roles.manage, users.assign_role, committees.manage, settings.manage                                  | –              | –            | –     | –        | –     | ✓           |
| audit.view, analytics.view, analytics.export                                                         | –              | –            | –     | –        | –     | ✓           |

Rules:

- The API checks permissions on **every** call. Verification status lives in our database, not in Clerk metadata.
- When an unverified resident attempts a protected action, the app shows a "Visit the hall to verify" screen with the hall's hours.

## 2. Scope

**In scope for v1:**

- Clearance & certificates. Three document types: Barangay Clearance, Certificate of Residency, Certificate of Indigency.
- Reports and complaints.
- Lost & found.
- Court and equipment bookings.
- Announcements.
- Activities, with day-before reminders.
- Notifications inbox and push notifications.
- Profile, EN/FIL i18n, consent, help.
- Staff tools on mobile and web.
- Audit log, analytics, and CSV export.

**Out of scope for v1:**

- Non-Abucay residents.
- Offline-first operation.
- In-app identity verification and e-documents. Verification and document release happen at the hall.
- Online payments and fees. No fee fields; a `payments` extension point is noted only.
- Households.
- SMS.
- Phone OTP. Deferred, and needs a paid Clerk plan.
- Activity RSVP and headcount.
- Waray-Waray. Strings are keyed so it can be added later.
- Multi-barangay.
- Store publishing.

## 3. Feature rules

### A. Requests

- **Request fields:** document type, purpose (picked from a list or "other" with text), type-specific fields defined in the catalog's `required_fields` JSON, and up to 2 optional attachments.
- **Reference number:** `ABC-YYYY-NNNNNN`, from a per-year sequence, unique. A QR code on the detail screen encodes the reference number.
- **Status flow:**
  - Submitted → Under Review → Ready for Pickup → Released
  - At Under Review, staff can also move a request to **Needs Info** (resident edits and resubmits, which returns it to Under Review) or **Rejected**.
  - Rejected and Needs Info require a reason, either from a preset list or free text.
  - Released requires staff to confirm the resident's ID at the counter.
- Every transition is written to `request_status_history` with the actor and a timestamp. This history provides the turnaround analytics.
- **Duplicate guard:** at most 1 open request per document type per resident, plus an idempotency key on the submit call.

### B. Reports

- **Categories:** noise, garbage, drainage/flooding, street lights, road damage, peace & order, others. Each category maps to a default **committee**.
- **Report fields:** description, up to 3 photos compressed on the device (about 1280px, JPEG at about 70%), purok (required), map pin and landmark (both optional), and an "Hide my name from the assigned official" toggle.
- **Status flow:**
  - Received → Assigned → In Progress → Resolved → Closed
  - Staff can also mark a report Closed as invalid or a duplicate, with a reason.
- Public remarks are visible to the reporter. Internal notes are visible only to staff, the assigned official, and admins.
- **Assignment:** staff assign a report to an official or Tanod in the category's committee and can reassign it. Tanod only see reports assigned to them.
- **Anti-abuse:** Redis rate limit of 5 reports/day/user. Staff can flag a user, and flagged users are blocked from submitting.

### C. Lost & Found

- **Posting:** verified residents (and staff) can post. A post has type (lost/found), title, description, 1 photo, date, location text, and category.
- **Claims are staff-mediated.** "This is mine" creates a claim. Staff and the poster are notified. Contact details are never exposed between residents, and the handover happens at the hall.
- **Status flow:** Open → Claimed → Returned → Archived.
  - A daily cron archives Open posts after **30 days**. The poster is warned 3 days before and can renew once.
  - Photos on archived posts are deleted after 90 days.
- Staff can hide or remove a post, with a reason.

### D. Bookings

**Court:**

- Booked in **1-hour slots, 6:00–22:00**, as one or more consecutive slots.
- Stored as a `tstzrange` with a **Postgres exclusion constraint** (`btree_gist`). The constraint applies while status is in (pending, approved, in_use), so two active bookings can never overlap.

**Equipment (chairs, tables, tents):**

- Rented by **whole-day date ranges**, picked up and returned at the hall. No delivery.
- Each booking has lines with quantities.
- Over-allocation is prevented inside one transaction:
  1. `SELECT … FOR UPDATE` the `inventory_items` row.
  2. Sum the quantities of overlapping active bookings.
  3. Reject if the sum plus the new quantity exceeds `total_owned`.

**Holds and expiry:**

- A **Pending** booking holds its slot or quantity immediately.
- It auto-expires after **48h** if staff haven't acted (cron), which releases the hold.

**Status flow:** Pending → Approved → In Use / Picked Up → Completed (court) or Returned (equipment).

- Rejected requires a reason.
- Cancelled: residents can cancel up to 24h before the start.
- Expired: set by the cron above.

**Limits (admin-editable `settings`):**

- Book 1–60 days ahead.
- At most 2 active court bookings and 1 active equipment booking per resident.

Staff can **block dates or slots** (events, maintenance). Blocked dates are stored as rows covered by the same exclusion constraint.

### E/F. Announcements & Activities

- **Announcements:** title, body, optional image, category (advisory / emergency / health / events / general), pinned flag, urgent flag.
  - Marking an announcement urgent sends a push to **all** registered device tokens.
- **Activities:** title, details, venue, start and end, image, and a list and calendar view.
  - Residents can toggle **"Remind me"**. A cron sends a push the day before to residents who turned it on. There is no public headcount.
- Content is shown in whatever language staff wrote it in.

### G. Cross-cutting

- **Notifications:** every status change writes an `notifications` inbox row and sends an Expo push. Tapping a notification deep-links to the related screen.
- **Status freshness:** TanStack Query refetches on screen focus, app foreground, and pull-to-refresh. No websockets and no polling.
- **Profile:**
  - Fields: name, birthdate, sex, purok (from a list), street/house no., phone (contact only), email (from Clerk), verification status, language preference.
- **Consent:**
  - The RA 10173 consent screen is required before the profile is created.
  - The `consents` table records the policy version and timestamp.
  - A new policy version re-prompts users.
- **Account deletion** (also required by the App Store): Clerk user deleted, PII in our database scrubbed, records kept against a pseudonymous ID.
- **Audit log:** append-only. Records every staff/admin write, plus _views_ of resident PII (directory and detail) and reporter identity.
- **Analytics:** request turnaround, report resolution time, booking counts, weekly active users, and SUS-relevant usage. Super Admin can export to CSV.
- **Accessibility:**
  - Respects system font scaling, 48dp minimum tap targets, WCAG AA contrast, screen-reader labels.
  - Icons always have text labels.
- **Language:** EN and FIL via i18next. Keys live in `packages/shared/i18n`.

## 4. Data model (core entities)

- **users**:
  - Columns: `id` uuid, `clerk_user_id` unique, `role_id`, `verified_at`, `verified_by`, `flagged_at`, `deleted_at`, `locale`.
  - Created by a lazy upsert on the first `/me` call. A Clerk webhook handles deletion.
- **profiles** (1:1 with users): `first_name`, `middle_name`, `last_name`, `suffix`, `birthdate`, `sex`, `purok_id`, `address_line`, `phone`.
- **puroks**: id, name.
- **roles** (`is_system`), **permissions** (seeded from code), **role_permissions**.
- **committees**, **committee_members** (user, committee), **report_categories** (default committee).
- **consents**: user, `policy_version`, `accepted_at`.
- **document_types**: code, names in EN and FIL, `required_fields` JSON, active flag.
- **document_requests** (reference number unique), **request_status_history**, **request_attachments**.
- **reports**: purok, lat/lng, landmark, `hide_identity`. Plus **report_photos**, **report_assignments**, **report_status_history**, **report_comments** (`visibility`: public or internal).
- **lost_found_items**, **lost_found_claims**.
- **inventory_items**: name, `total_owned`, active.
- **bookings**:
  - Columns: `kind` (court or equipment), user, status, purpose, `time_range` tstzrange (court) or `date_range` daterange (equipment), and the status timestamps.
  - Related tables: **booking_lines** (item, qty), **booking_status_history**, **blocked_periods**.
- **announcements**, **activities**, **activity_reminders** (user, activity, unique).
- **device_tokens**: user, Expo token unique, platform, `last_seen`.
- **notifications**: user, type, entity reference, title and body keys, `read_at`, push status, receipt id. This table doubles as the push outbox.
- **audit_logs**: actor, action, entity type and id, before/after JSON diff, IP, timestamp. Append-only; there is no update or delete route.
- **settings**: key/value, admin-editable booking limits and hours.
- **Future (not built):** `payments` (polymorphic to request or booking), `households`, `barangays` (multi-tenant).

## 5. Architecture

```
apps/mobile   Expo SDK 58 + Expo Router: (resident) and (staff) route groups; web export serves staff at the hall
apps/api      Node + Express 5 REST /v1, Zod-validated, Helmet, CORS, Upstash rate limit, @clerk/express
packages/db   Drizzle schema + Drizzle Kit migrations (incl. raw SQL for btree_gist + exclusion constraints)
packages/shared  Zod schemas (Zod 4), DTO types, permission keys, status enums + allowed transitions, i18n keys
```

- **Business logic lives in the API service layer.** Status transitions go through one state-machine table per entity in `packages/shared`, and the API enforces them. The client only reflects state.
- **Third-party services:**

  | Service                           | Responsibility                                                                                                                   |
  | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
  | Clerk                             | Identity only: email code sign-in, with Google optional                                                                          |
  | Neon Postgres                     | Source of truth, pooled connection                                                                                               |
  | Cloudflare R2                     | Photos and attachments via presigned PUT. Private bucket, signed GET for internal docs, public-read only for announcement images |
  | Upstash Redis                     | Rate limits                                                                                                                      |
  | Expo Push                         | Delivery                                                                                                                         |
  | Sentry                            | Errors, tracing, releases                                                                                                        |
  | react-native-maps + expo-location | Optional report pin                                                                                                              |

- **Background work:** no BullMQ in v1. Jobs run on cron (Railway/Render cron or an in-process scheduler with a DB lock):
  - expire pending bookings
  - archive and warn on lost & found posts
  - activity reminders
  - push receipt checks

  Push sends are chunked at 100 per request. That covers an all-user announcement (about 10k tokens, about 100 requests).

- **Idempotency:** `Idempotency-Key` header on every create endpoint. Push deduplicated by notification id.
- **Weak networks:** small JSON payloads with pagination, thumbnails, upload retry with progress, and plain-language errors in EN/FIL.

### Versions (checked with `npm view` on 2026-10-22; re-confirm against the docs when implementing)

- **Expo and React Native (from the repo):** expo ~58.0.16, react-native 0.88.0, react 19.3.0, expo-router ~58.0.24, TypeScript ~6.0.3, expo-notifications 58.1.5.
- **Auth:** **@clerk/expo 4.13.1**. The brief names `@clerk/clerk-expo`, which stops at 2.20.0 and appears to be the old package name; verify with the Clerk docs. @clerk/express 2.1.81.
- **Database:** drizzle-orm 0.45.4, drizzle-kit 0.31.11.
- **Monitoring:** @sentry/react-native 8.31.0, @sentry/node 11.8.0.
- **Client libraries:** nativewind 4.2.7 (check SDK 58 / RN 0.88 compatibility), i18next 26.5.4, @tanstack/react-query 5.104.5, zustand 5.0.15, zod 4.6.7, react-native-maps 1.29.11.
- **Server and tooling:** express 5.3.0, turbo 2.11.10.

## 6. Screen map

```
(onboarding)  language → intro slides → sign-in/sign-up (email code) → consent → complete profile → pending-verification
(resident) tabs: Home | Services | Notifications | Profile
  Home: 6 service tiles, pinned/urgent announcements, my active items
  services/requests: catalog → form → my requests → detail (timeline, ref no., QR)
  services/reports: new → my reports → detail (timeline, public remarks)
  services/lost-found: board [Lost|Found, search, filters] → item → post → my posts & claims
  services/bookings: court|equipment → calendar → form → my bookings → detail
  announcements: feed → detail     activities: list|calendar → detail (Remind me)
  profile: details & verification, language, notifications, privacy policy, help/guide, about, delete account, sign out
(staff) role-gated, responsive for web: Dashboard (queue counts)
  requests queue → process | reports → detail (assign, status, remarks, internal notes)
  lost-found moderation & claims | bookings (calendar, approve/reject, block dates) | inventory
  announcements manager | activities manager | residents directory & verification
  roles & users | committees | settings | audit log | analytics & CSV export   (Super Admin only)
```

## 7. Sprint roadmap (2-week sprints, ~12 sprints; ASSUMED timeline)

| #   | Sprint goal                                                                                      |
| --- | ------------------------------------------------------------------------------------------------ |
| 0   | Monorepo restructure, CI, environments (dev/staging/prod), Neon, Sentry, EAS profiles, CLAUDE.md |
| 1   | Auth (Clerk email), `/me` upsert, consent, profile, roles/permissions engine, staff web shell    |
| 2   | Residents directory and verification, audit log, i18n scaffolding, design system                 |
| 3   | Announcements and push infrastructure (tokens, outbox, deep links), notifications inbox          |
| 4   | Requests: catalog, form, reference number, QR, status machine                                    |
| 5   | Requests: staff queue and processing, attachments (R2)                                           |
| 6   | Reports: resident flow, photos, location                                                         |
| 7   | Reports: assignment and committees, remarks and notes, anti-abuse                                |
| 8   | Bookings: court with exclusion constraint, calendar, blocked periods                             |
| 9   | Equipment and inventory, holds and expiry cron, booking staff tools                              |
| 10  | Lost & found plus claims and archive cron; activities plus reminders                             |
| 11  | Analytics and export, roles UI, accessibility and FIL pass, k6 load test, security review        |
| 12  | Hardening, seed and demo data, Maestro E2E, SUS evaluation, defense builds                       |

**Definition of done for each sprint:**

- Lint, typecheck, and tests pass in CI.
- Larkken (QA) runs the sprint's test checklist on an Android APK and the iOS TestFlight build.
- Strings exist in both EN and FIL.
- Every new staff action writes an audit entry.

## 8. ASSUMPTIONS (no firm answer given)

1. The defense is about 6 months away, with 2-week sprints (you skipped this question).
2. The budget is about $10–25/month plus the $99/yr Apple Developer account (you skipped this question).
3. The team is 3 people, with Larkken on QA (you skipped this question).
4. The admin web build uses a static or single-page web export, not `web.output: "server"`. Admin web is hosted as a static site (Cloudflare Pages, Netlify, or similar).
5. With no fees in the app, the only payments work in v1 is the documented `payments` extension point.
6. Activity reminders use a per-resident "Remind me" toggle, since there's no RSVP.
7. Each document type has 1 open request per resident at a time, and up to 2 attachments.
8. The puroks list, hall hours, and the real document field requirements come from the barangay before Sprint 4.
9. Minimum OS: whatever Expo SDK 58 supports. The app is portrait-only.
10. There are three environments: local, staging, and production. EAS profiles: development, preview (APK/TestFlight), production.
11. Seed data (puroks, categories, committees, inventory, demo accounts) is maintained for the demos and the defense.

## 9. OPEN RISKS / unknowns

- **Phone OTP:** Clerk Hobby has no SMS, and SMS to the Philippines is billed at "market rate." Residents without email may struggle at sign-up. Mitigation: staff help residents sign up during the hall verification visit.
- **Clerk package rename** (`@clerk/clerk-expo` → `@clerk/expo`): the APIs may differ from tutorials.
- **NativeWind and the UI kit on RN 0.88 / SDK 58:** compatibility is unverified. Do a spike in Sprint 0, and fall back to plain StyleSheet with tokens if needed.
- **Free-tier limits:**
  - Neon compute auto-suspends, so the first request after idle is slow.
  - Render's free tier has cold starts.
  - R2 egress is fine, but storage grows with photos.
- **Expo push on iOS** requires APNs credentials through EAS. A push storm during an emergency could hit rate limits.
- **RA 10173:** the barangay must name a Data Protection Officer and approve the privacy policy text, and the adviser must sign off on retention.
- **Digital literacy** of seniors and staff: run SUS early on prototypes, not only at the end.
- **Desktop web staff UX** with React Native components: calendars and tables need responsive layouts, and these are the most likely to slip.
- **Real data from the barangay:** puroks, committees, document fields, inventory counts. Delays here block Sprints 4–9.
- **Scope vs. schedule:** if the timeline turns out to be about 4 months, cut the roles UI (keep seeded roles), analytics export, and maps.
