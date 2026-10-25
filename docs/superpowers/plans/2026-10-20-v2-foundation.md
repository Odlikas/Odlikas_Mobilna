# Odlikaš v2 Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the v2 monorepo and the complete Supabase database (41 tables, row-level security, quotas, pairing, photo retention) that every later Odlikaš v2 feature builds on.

**Architecture:** New top-level `supabase/`, `packages/` and `apps/` folders are added next to the legacy `odlikas_mobilna/` and `functions/` (left untouched; deleted once v2 replaces them). All schema lives in ordered SQL migrations with RLS in the same file as the tables it protects; every migration is proven by pgTAP tests run through `supabase test db`. Pairing is two Postgres RPC functions called from a small Dart package.

**Tech Stack:** Supabase CLI (local Postgres 15 + pgTAP), SQL, Dart 3.6+ pub workspaces, `supabase_flutter`, `package:test`.

**Spec:** "Odlikaš — Backend, Database & AI Cost Architecture", https://claude.ai/code/artifact/6b788518-e690-4c07-b377-f4e3ab935f14 (sections Database, Security/GDPR, App and file structure). Read it before starting.

## Plan series

This is plan 1 of 6. Each produces working, tested software on its own.

1. **Foundation** (this plan): monorepo, schema, RLS, quotas, pairing, retention.
2. Learning engine (`packages/learning`): math generators, answer matcher, quiz engine, unlock rules engine.
3. AI: `ai-gateway`, photo-help cascade, homework check, answer verdicts, cost log, budget kill switch.
4. e-Dnevnik sync in the parent app (`packages/ednevnik`, porting the parser from the Heroku scraper `odlikas-e-dnevnik-api`).
5. Content pipeline: curriculum import (NN 7/2019), bank builder (batch), pack builder.
6. Apps: parent + kid Flutter UI from Figma v2, Kotlin/Swift blocking bridges, RevenueCat `store-webhook`.

## Global Constraints

- Supabase project region: EU, `eu-central-1` (Frankfurt). Default timezone `Europe/Zagreb`, locale `hr`.
- Every family-owned table has a non-null `family_id uuid references families(id) on delete cascade` and RLS enabled; global content tables have no personal data.
- No column anywhere stores e-Dnevnik credentials (they live only in the parent phone's keystore).
- `id uuid primary key default gen_random_uuid()` and `created_at timestamptz not null default now()` unless the spec table says otherwise.
- `children.grade` is `smallint check (grade between 1 and 8)`; `grades.value` is `check (value between 1 and 5)`.
- `time_requests.approved_min` is `check (approved_min in (15, 30, 60))`.
- `child_settings` defaults: `daily_unlock_min 60`, `grade5_bonus_min 30`, `streak_bonus_min 60`, `photo_help_daily_cap 10`; `parents.kid_facing_name` default `'mama'`.
- Photos: `delete_after = created_at + interval '30 days'`.
- Pairing code: 6 characters from `ABCDEFGHJKMNPQRSTUVWXYZ23456789`, valid 10 minutes, single use; QR payload `odlikas://pair?c=<CODE>`.
- Free quota: 3 quizzes per ISO week per child; Premium daily caps from `child_settings` / `app_config` (photo help 10, homework check 5).
- Only parents decide: kids can never approve time, change rules, or edit settings.
- Do not modify `odlikas_mobilna/` or `functions/`.

## Review Focus

1. **A kid's phone tries to approve its own time request or edit its rules.** Expect: denied by RLS, row unchanged. Test in Task 6.
2. **A pairing code typed with lowercase or spaces (`"ab3 k9p"`), expired, or used twice.** Expect: normalised and accepted while valid; expired/used → error `pairing_code_invalid`. Tests in Task 8 and Task 9.
3. **A second parent (guardian) in the same family, and a parent in another family.** Expect: guardian sees and edits everything in their family; the other family's parent sees zero rows. Test in Task 2.
4. **The same e-Dnevnik grade synced twice.** Expect: one `grades` row and one notification. Test in Task 5.
5. **A photo older than 30 days.** Expect: storage object deleted, `ocr_text` nulled, `deleted_at` set; younger photos untouched. Test in Task 7.

---

### Task 1: Tooling, Supabase init, test helpers

**Files:**
- Create: `supabase/config.toml` (via `supabase init`), `supabase/tests/00_helpers.sql`, `supabase/migrations/20261020000000_test_helpers.sql`
- Modify: `.gitignore` (add `supabase/.temp/`, `supabase/.branches/`)

**Interfaces:**
- Produces (schema `tests`, only used by pgTAP):
  - `tests.create_parent(email text) returns uuid` — inserts an `auth.users` row, returns its id.
  - `tests.create_child_user() returns uuid` — inserts an anonymous `auth.users` row.
  - `tests.as(uid uuid)` — `set local role authenticated` + `set local request.jwt.claims = json_build_object('sub', uid, 'role', 'authenticated')`.
  - `tests.as_service()` — `set local role service_role`.

- [ ] **Step 1: Install tools**

Run: `brew install supabase/tap/supabase deno && brew install --cask orbstack` (OrbStack provides Docker; Docker Desktop also works). Start OrbStack once.
Expected: `supabase --version` and `deno --version` print versions; `docker ps` works.

- [ ] **Step 2: Init and start**

Run: `supabase init && supabase start`
Expected: local API URL and anon/service keys printed.

- [ ] **Step 3: Write `supabase/tests/00_helpers.sql` smoke test**

```sql
begin;
select plan(2);
select ok(tests.create_parent('a@test.hr') is not null, 'creates parent user');
select ok(tests.create_child_user() is not null, 'creates child user');
select * from finish();
rollback;
```

- [ ] **Step 4: Run it, expect failure** — `supabase test db` → FAIL (`schema "tests" does not exist`).

- [ ] **Step 5: Implement the helper migration** (`create extension if not exists pgtap with schema extensions;` plus the four functions above).

- [ ] **Step 6: Run** `supabase db reset && supabase test db` → PASS.

- [ ] **Step 7: Commit** `chore: supabase init and pgTAP helpers`

---

### Task 2: Families and accounts (area A) + RLS core

**Files:**
- Create: `supabase/migrations/20261020000100_accounts.sql`, `supabase/tests/01_accounts.sql`

**Interfaces:**
- Consumes: Task 1 helpers.
- Produces:
  - Enums `plan_kind (free, premium)`, `parent_role (owner, guardian)`, `platform_kind (android, ios)`.
  - Tables `families, parents, children, child_settings, devices, pairing_codes` with the spec's fields.
  - `app.my_family_id() returns uuid` (stable, security definer): the family of `auth.uid()` as parent, else as child (`children.auth_user_id`).
  - `app.is_parent() returns boolean`, `app.my_child_id() returns uuid` (null for parents).
  - Trigger: inserting `children` inserts its `child_settings` row with defaults.
  - RPC `create_family(p_name text, p_display_name text) returns uuid` — creates family + owner parent for `auth.uid()`; errors `already_in_family` if one exists.

RLS: parents select/insert/update/delete all A rows where `family_id = app.my_family_id()`. A child selects only its own `children`, `child_settings`, `devices` rows and its family's `parents.kid_facing_name, display_name`; no writes except `devices.push_token, app_version, blocking_ok, permissions, last_seen_at` on its own device (column-level `grant update (...)`).

- [ ] **Step 1: Write failing tests in `01_accounts.sql`**

Assertions (one `select` each):
- owner can `create_family('Horvat','Marija')` and sees 1 family.
- guardian added to same family sees the same child rows as owner.
- parent of another family sees 0 `children` rows of family 1 (`is((select count(*) from children), 0::bigint)` under `tests.as(other)`).
- inserting a child creates `child_settings` with `daily_unlock_min = 60`, `grade5_bonus_min = 30`, `streak_bonus_min = 60`, `photo_help_daily_cap = 10`.
- `children.grade = 9` raises `check_violation` (`throws_ok(..., '23514')`).
- child user can update own `devices.push_token`; `update child_settings set daily_unlock_min = 600` as child affects 0 rows.
- `create_family` twice for one user throws `already_in_family`.

- [ ] **Step 2: Run** `supabase test db` → FAIL (relations missing).
- [ ] **Step 3: Implement the migration.**
- [ ] **Step 4: Run** `supabase db reset && supabase test db` → PASS.
- [ ] **Step 5: Commit** `feat(db): families, parents, children, devices with RLS`

---

### Task 3: Subscriptions, quotas, config (area B)

**Files:**
- Create: `supabase/migrations/20261020000200_billing_quota.sql`, `supabase/tests/02_billing_quota.sql`

**Interfaces:**
- Produces:
  - Tables `subscriptions` (unique `family_id`), `usage_counters` (PK `child_id, feature, period_start`), `app_config` (PK `key`).
  - Enums `sub_status (trialing, active, grace, canceled, expired)`, `billing_period (month, year)`, `store_kind (app_store, play_store)`, `quota_feature (quiz, photo_help, homework_check)`.
  - Trigger on `subscriptions` keeping `families.plan = 'premium'` when status in (trialing, active, grace), else `'free'`.
  - `app.is_premium(p_family uuid) returns boolean`.
  - RPC `consume_quota(p_child uuid, p_feature quota_feature) returns boolean` (security definer): free + quiz → period = ISO week start (`date_trunc('week', now() at time zone 'Europe/Zagreb')::date`), limit 3; free + photo/homework → false; premium + quiz → true without counting; premium + photo_help → daily limit `child_settings.photo_help_daily_cap`; premium + homework_check → daily limit `app_config['quota.homework_check_daily']` (seed `5`). Increments atomically (`insert ... on conflict do update ... returning count`) and returns whether the call was within the limit. Caller must be that child or a parent of its family.
  - Seed `app_config`: `min_version` `{"android":"2.0.0","ios":"2.0.0"}`, `ai_daily_budget_eur` `5`, `ai_kill_switch` `false`, `quota.homework_check_daily` `5`.

RLS: family reads own `subscriptions` and `usage_counters`; no client writes (only service role / RPC). `app_config` readable by `authenticated`, writable by service role only.

- [ ] **Step 1: Failing tests** — free child: `consume_quota(..,'quiz')` returns true, true, true, then false; premium child: 11th `photo_help` same day returns false; free child `photo_help` returns false; setting a subscription to `active` flips `families.plan` to `premium`, `expired` flips back; parent cannot `insert into subscriptions` (RLS error `42501`).
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(db): subscriptions, quotas, app_config`

---

### Task 4: Shared content (area C)

**Files:**
- Create: `supabase/migrations/20261020000300_content.sql`, `supabase/seed.sql`, `supabase/tests/03_content.sql`

**Interfaces:**
- Produces:
  - `create extension if not exists pg_trgm`.
  - Tables `subjects, curriculum_outcomes, generators, bank_items, content_packs, answer_verdicts, solutions` exactly as the spec lists (unique `curriculum_outcomes.code`, unique `bank_items.content_hash`, unique `solutions.problem_key`, GIN `gin_trgm_ops` index on `solutions.problem_norm`).
  - `find_solution(p_norm text, p_min real default 0.9) returns solutions` — the most similar `live` row with `similarity >= p_min`, increments `hits` and sets `last_hit_at`; null when none. Security definer, callable by `authenticated`.
  - Seed subjects: `MAT Matematika, HRV Hrvatski jezik, ENG Engleski jezik, PID Priroda i društvo, PRI Priroda, BIO Biologija, KEM Kemija, FIZ Fizika, POV Povijest, GEO Geografija, INF Informatika, NJE Njemački jezik`.

RLS: `select` for `authenticated` on all, except `bank_items` only where `status = 'approved'`; no client writes.

- [ ] **Step 1: Failing tests** — authenticated user sees 12 subjects; a `draft` bank item is invisible to a parent, visible to service role; inserting a duplicate `content_hash` throws `23505`; `find_solution('345+278=')` returns the row stored as `'345+278='` and bumps `hits` to 1; `find_solution('completely different')` returns null.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(db): shared content tables and solution lookup`

---

### Task 5: School data, notifications, compliance (areas D + G)

**Files:**
- Create: `supabase/migrations/20261020000400_school_comms.sql`, `supabase/tests/04_school_comms.sql`

**Interfaces:**
- Produces:
  - Tables `schools, child_subjects, grades (unique source_hash), timetable_entries, calendar_events, sync_runs, notifications, weekly_reports, consents, audit_log` per spec. `sync_runs` references the **parent's** device context: column `parent_id uuid references parents(id)` instead of `device_id`.
  - Trigger `grades_after_insert` → inserts `notifications(type 'new_grade', payload {"subject_id","value","child_id"})` for the family; fires only on real inserts (upserts that hit `on conflict do nothing` do not fire).
  - RPC `upsert_grades(p_child uuid, p_rows jsonb) returns int` — inserts rows with `on conflict (source_hash) do nothing`, returns rows inserted; parent-only.

RLS: parents full access in their family; child read-only on its own `grades, timetable_entries, calendar_events, child_subjects, notifications (child_id = me)`; `consents` and `audit_log` parent read, service-role write.

- [ ] **Step 1: Failing tests** — `upsert_grades` with the same row twice → returns 1 then 0; exactly 1 `new_grade` notification; child reads own grades, cannot insert grades (`42501`); grade value 6 throws `23514`; parent of other family sees 0 grades.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(db): school data, notifications, consents`

---

### Task 6: Tasks, learning, unlock (area E)

**Files:**
- Create: `supabase/migrations/20261020000500_daily.sql`, `supabase/tests/05_daily.sql`

**Interfaces:**
- Produces:
  - Tables `task_rules, daily_plans (unique child_id, plan_date), tasks, quiz_sessions, quiz_answers, learning_sessions, unlock_grants, time_requests, usage_daily, streaks` with the spec's enums.
  - RPC `decide_time_request(p_request uuid, p_approved_min smallint) returns unlock_grants` — parent-only; `p_approved_min` null = deny; on approve sets status `approved`, `decided_by`, `decided_at` and inserts `unlock_grants(source 'time_request', minutes p_approved_min, ref_id p_request, grant_date today Europe/Zagreb)`.
  - Realtime publication includes `time_requests` and `unlock_grants`.

RLS:
- Parent: full access in family, except no direct `insert` into `unlock_grants` with source `time_request` (only via RPC).
- Child: select own rows; insert/update own `daily_plans, tasks, quiz_sessions, quiz_answers, learning_sessions, usage_daily, streaks`; insert own `unlock_grants` only with `source in ('daily_tasks','grade5_bonus','streak_bonus')`; insert `time_requests` only with `status = 'pending'` and `approved_min is null`; **no** update/delete on `time_requests` or `task_rules`.

- [ ] **Step 1: Failing tests** — child inserts pending request; child `update time_requests set status='approved'` affects 0 rows; child insert of `unlock_grants` with source `time_request` throws `42501`; parent `decide_time_request(id, 30)` creates a 30-minute grant; `decide_time_request(id, 45)` throws `23514`; child cannot `insert into task_rules`; second `daily_plans` row for same child/date throws `23505`.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(db): daily plans, quizzes, unlock grants, time requests`

---

### Task 7: Photos, AI tables, storage, retention (area F)

**Files:**
- Create: `supabase/migrations/20261020000600_photos_ai.sql`, `supabase/tests/06_photos_ai.sql`

**Interfaces:**
- Produces:
  - Tables `photos (delete_after default now() + interval '30 days'), help_requests, homework_checks, ai_calls, content_reports` per spec; partial index `photos (delete_after) where deleted_at is null`.
  - Storage buckets: `photos` (private), `packs` (public read), `avatars` (private). Storage policies: object path's first segment must equal `app.my_family_id()::text` for read/insert on `photos` and `avatars`.
  - `app.purge_expired_photos() returns int` — for photos with `delete_after < now()` and `deleted_at is null`: deletes `storage.objects` row for `storage_path`, sets `ocr_text = null`, `deleted_at = now()`; returns count.
  - `pg_cron` job `purge-photos` at `0 3 * * *` calling it.

RLS: family read on photos/help/homework; child insert own `photos` and `help_requests` and `content_reports`; `homework_checks` and `ai_calls` written by service role only; `ai_calls` readable by nobody but service role.

- [ ] **Step 1: Failing tests** — photo created 31 days ago (set `created_at`, `delete_after` explicitly) is purged, a 2-day-old photo is not, function returns 1; parent of other family selects 0 photos; child cannot select `ai_calls` (0 rows); storage insert to path `<other_family>/x.jpg` is denied.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(db): photos, AI tables, storage buckets, 30-day purge`

---

### Task 8: Pairing RPCs

**Files:**
- Create: `supabase/migrations/20261020000700_pairing.sql`, `supabase/tests/07_pairing.sql`

**Interfaces:**
- Produces:
  - `create_pairing_code(p_child uuid) returns text` — parent-only; generates a 6-char code from the constraint alphabet (retry on collision), `expires_at = now() + interval '10 minutes'`.
  - `claim_pairing_code(p_code text, p_platform platform_kind, p_app_version text) returns uuid` — caller is an anonymous child-device user; normalises `upper(regexp_replace(p_code, '\s', '', 'g'))`; raises `pairing_code_invalid` if missing, expired or used; sets `children.auth_user_id = auth.uid()` (replacing a previous device: old `devices` row gets `revoked_at = now()`), inserts `devices`, marks code `used_at`, `device_id`; returns device id.

(Spec listed pairing as an Edge Function; RPCs do the same with less code and are testable in pgTAP. Call made by plan author.)

- [ ] **Step 1: Failing tests** — code matches `^[ABCDEFGHJKMNPQRSTUVWXYZ23456789]{6}$`; claiming with `lower` + a space inside succeeds; claiming again throws `pairing_code_invalid`; claiming after `expires_at` set to past throws; a kid user that is not a parent calling `create_pairing_code` throws `42501`; after claim, the child user sees its own `children` row; re-pairing a new phone revokes the old device.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(db): QR/code pairing RPCs`

---

### Task 9: Dart workspace + `packages/core` pairing client

**Files:**
- Create: `pubspec.yaml` (root workspace: `workspace: [packages/core]`, `environment: sdk: ^3.6.0`), `packages/core/pubspec.yaml` (`resolution: workspace`, deps `supabase: ^2.x`, dev `test`, `mocktail`), `packages/core/lib/core.dart`, `packages/core/lib/src/pairing/pairing_payload.dart`, `packages/core/lib/src/pairing/pairing_repository.dart`, `packages/core/test/pairing_payload_test.dart`, `packages/core/test/pairing_repository_test.dart`

**Interfaces:**
- Consumes: Task 8 RPC names and the error `pairing_code_invalid`.
- Produces:
  - `class PairingPayload { final String code; static PairingPayload parse(String input); String toQrString(); }` — accepts either `odlikas://pair?c=AB3K9P` or a typed code; normalises (uppercase, strips whitespace and `-`); throws `FormatException` unless 6 chars from the alphabet.
  - `class PairingRepository { PairingRepository(SupabaseClient client); Future<String> createCode(String childId); Future<String> claim(PairingPayload p, {required String platform, required String appVersion}); }` — `claim` signs in anonymously if no session; maps the Postgres error `pairing_code_invalid` to `PairingException.invalidCode`.

- [ ] **Step 1: Failing tests**

```dart
test('parses QR and typed codes the same', () {
  expect(PairingPayload.parse('odlikas://pair?c=AB3K9P').code, 'AB3K9P');
  expect(PairingPayload.parse(' ab3 k9p ').code, 'AB3K9P');
  expect(PairingPayload.parse('AB3-K9P').code, 'AB3K9P');
});
test('rejects ambiguous characters and wrong length', () {
  expect(() => PairingPayload.parse('AB3K9O'), throwsFormatException); // O not allowed
  expect(() => PairingPayload.parse('AB3K9'), throwsFormatException);
});
test('round-trips QR string', () {
  expect(PairingPayload.parse('AB3K9P').toQrString(), 'odlikas://pair?c=AB3K9P');
});
```

Repository test (mocktail on the RPC call): `claim` on a `PostgrestException(message: 'pairing_code_invalid')` throws `PairingException.invalidCode`.

- [ ] **Step 2: Run** `dart test packages/core` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `dart test packages/core` → PASS. Also run `supabase test db` → PASS (full suite).
- [ ] **Step 5: Commit** `feat(core): pairing payload and repository`
