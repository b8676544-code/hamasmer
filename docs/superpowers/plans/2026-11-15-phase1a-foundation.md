# Hamasmer CRM – Phase 1A: Foundation (DB, RLS, 2FA, Automations, Email) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A deployed Next.js app with password + email-code login, and a Supabase database that holds all Phase 1 entities, enforces role permissions and 2FA in RLS, runs automations A1/A6/A7/A8, and emails task owners.

**Architecture:** All business rules live in Postgres (triggers, pg_cron, RLS). Next.js (App Router) on Vercel talks to Supabase with the user's JWT; the service-role key is used only server-side for login codes and by the `send-emails` Edge Function. Emails go through an `email_outbox` table drained every minute.

**Tech Stack:** Next.js 15 (App Router, TypeScript), Tailwind 4, shadcn/ui, `@supabase/ssr`, `@supabase/supabase-js` v2, zod, Vitest 3, Supabase (Postgres 15+, pg_cron, pg_net, Vault, Edge Functions/Deno), Resend, Vercel, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-11-15-phase1-private-classes-design.md` (read Part B, sections 11–19, before starting).

**Scope of this plan:** spec sections 3 (data), 4–7 (permissions, login, automations, email), 11–15, 17–19. **Not in this plan** (Plan 1B, written after this one is executed): all business screens (spec section 8), user-invite UI, cities seed, Playwright E2E, data import.

## Global Constraints

- UI language Hebrew, `<html lang="he" dir="rtl">`, font Heebo. Table/column names English snake_case.
- Mobile phones are stored as `05XXXXXXXX` (digits only). Never store formatted phones.
- No physical deletes of business rows: archive with `archived_at`. Only exception: `lead_children` join rows (see Deviations).
- No card details anywhere. `payment_set` is a boolean only.
- Service-role key and Resend key live only in `.env.local` / Vercel env / Supabase secrets. `.env*` is git-ignored. **Never print secret values in chat or commit them.** The user types secrets herself.
- Never write a real person's email address into code, tests, docs or scripts. Test users use the `@hamasmer.test` domain.
- Every RLS policy must depend on `app.user_role()`, which returns NULL unless the session passed the email code.
- Dummy/test data uses modest Jewish names (e.g. family "כהן", child "יוסי").
- Password minimum length 10. Login code: 6 digits, valid 10 minutes, 5 wrong tries → 15-minute lock, new code at most once per 60 s. Verified session lasts 12 hours.
- Timezone for all schedules: `Asia/Jerusalem`. Task emails go out at 07:30 on the due date.
- Supabase region: `eu-central-1`.
- Commits end with: `Co-Authored-By: Claude Fable 5.2 <noreply@anthropic.com>`. If `git config user.name` is empty, ask the user to set her git name and email once (`git config user.name ...`, `git config user.email ...`) before the first commit.
- All commands run from the repo root (the `hamasmer` project folder) in Git Bash.

## Deviations from the spec (decided while planning)

1. `app.current_role()` is named **`app.user_role()`** (`current_role` is a reserved word in Postgres). It also folds in the `is_verified` check.
2. Landline regex is `^0[23489]\d{7}$` (the spec's `0[2-9]` would accept a 9-digit malformed mobile).
3. `lead_children` rows may be deleted by manager/secretary (it is a join table; the delete is audited).
4. A7 cron runs at minute 5 of every hour and acts when the Israel hour is 7 (i.e. 07:05, not 07:00).
5. `login_codes` is keyed by `user_id` (one live code per user) so the lock cannot be bypassed by signing in again.
6. Column-specific triggers are not used (a BEFORE trigger changing `status` would not fire them); triggers fire on every insert/update and compare OLD/NEW.
7. `app_settings` has an extra `app_url` column, used to build links in emails.

## File Structure

```
app/layout.tsx, app/globals.css          root layout (he / rtl / Heebo)
app/(auth)/login/page.tsx                email + password form
app/(auth)/login/actions.ts              signIn / signOut server actions
app/(auth)/verify/page.tsx               6-digit code form
app/(app)/layout.tsx                     authenticated shell (header + sign out)
app/(app)/page.tsx                       placeholder home ("היום שלי" comes in Plan 1B)
app/api/auth/code/request/route.ts       create + email a login code
app/api/auth/code/verify/route.ts        check code, create verified session
middleware.ts                            session refresh + redirects
lib/supabase/server.ts | client.ts | admin.ts | middleware.ts
lib/auth/login-code.ts                   pure: generate / hash / compare code
lib/auth/session.ts                      read session_id from the access token
lib/email/resend.ts                      sendEmail()
lib/phone.ts, lib/il-id.ts               client-side mirrors of the DB rules
lib/errors.ts                            DB error token → Hebrew message
supabase/migrations/2026111501..06_*.sql schema, RLS, audit, automations, cron, email
supabase/functions/send-emails/index.ts  outbox drain
scripts/create-manager.mjs               bootstrap the first manager user
tests/unit/*.test.ts                     pure TS tests
tests/db/setup.ts, helpers.ts, *.test.ts tests against the hamasmer-dev project
.github/workflows/ci.yml
```

---

## Task 0: Prerequisites (the user does these; the agent guides in simple Hebrew)

No code. Do not proceed to Task 3 until all boxes are checked. Explain each step to the user in plain Hebrew, one at a time. She performs every sign-up, password and secret entry herself.

- [ ] **Supabase:** user creates a project named `hamasmer-dev`, region *Central EU (Frankfurt)*, and saves the database password in her password manager. She tells the agent only the **project ref** (the 20-letter id in the project URL; not a secret).
- [ ] **Resend:** user creates an account at resend.com and an API key (name: `hamasmer-dev`, permission: *Sending access*). Until a domain is bought, mail can only be delivered to the address she signed up with; that is expected.
- [ ] **GitHub:** user creates an empty **private** repository `hamasmer-crm` and tells the agent its URL.
- [ ] **Local secrets file:** the agent creates `.env.local` and `.env.test.local` containing only the variable names below; the user pastes the values herself (Supabase dashboard → Project Settings → API).

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
RESEND_API_KEY=
EMAIL_FROM=Hamasmer CRM <onboarding@resend.dev>
```

- [ ] **Supabase CLI login:** user runs `npx supabase login` (opens the browser) and then `npx supabase link --project-ref <ref>` and types the database password when asked.

---

## Task 1: Project scaffold (Next.js, RTL, Vitest, CI, security headers)

**Files:**
- Create: whole Next.js scaffold at repo root, `vitest.config.ts`, `tests/unit/smoke.test.ts`, `.github/workflows/ci.yml`
- Modify: `app/layout.tsx`, `app/globals.css`, `next.config.ts`, `package.json`, `.gitignore`

**Interfaces:**
- Produces: npm scripts `dev`, `build`, `lint`, `typecheck`, `test:unit`, `test:db`; path alias `@/*` → repo root.

- [ ] **Step 1: Scaffold into a temp folder and move to the root** (create-next-app refuses a non-empty folder)

```bash
npx create-next-app@15 _scaffold --typescript --tailwind --eslint --app --no-src-dir --import-alias "@/*" --use-npm --no-turbopack --yes
rm -rf _scaffold/.git
cp -r _scaffold/. . && rm -rf _scaffold
npm install @supabase/supabase-js @supabase/ssr zod server-only
npm install -D vitest dotenv
npx shadcn@latest init -d
npx shadcn@latest add button input label card
```

- [ ] **Step 2: Add ignores** – append to `.gitignore`:

```
.env*
!.env.example
supabase/.temp/
```

- [ ] **Step 3: RTL root layout** – replace `app/layout.tsx`:

```tsx
import type { Metadata } from "next";
import { Heebo } from "next/font/google";
import "./globals.css";

const heebo = Heebo({ subsets: ["hebrew", "latin"], variable: "--font-heebo" });

export const metadata: Metadata = {
  title: "המסמר – ניהול חוגים",
  robots: { index: false, follow: false },
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="he" dir="rtl">
      <body className={`${heebo.variable} font-sans antialiased`}>{children}</body>
    </html>
  );
}
```

In `app/globals.css`, inside the existing `@theme inline { ... }` block set: `--font-sans: var(--font-heebo);`

- [ ] **Step 4: Security headers** – replace `next.config.ts`:

```ts
import type { NextConfig } from "next";

const supabaseHost = process.env.NEXT_PUBLIC_SUPABASE_URL ?? "";
const csp = [
  "default-src 'self'",
  "script-src 'self' 'unsafe-inline'" + (process.env.NODE_ENV === "development" ? " 'unsafe-eval'" : ""),
  "style-src 'self' 'unsafe-inline'",
  "img-src 'self' data:",
  "font-src 'self'",
  `connect-src 'self' ${supabaseHost}`,
  "frame-ancestors 'none'",
  "base-uri 'self'",
  "form-action 'self'",
].join("; ");

const nextConfig: NextConfig = {
  async headers() {
    return [{
      source: "/(.*)",
      headers: [
        { key: "Content-Security-Policy", value: csp },
        { key: "X-Frame-Options", value: "DENY" },
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains" },
      ],
    }];
  },
};
export default nextConfig;
```

- [ ] **Step 5: Vitest config** – create `vitest.config.ts`:

```ts
import path from "node:path";
import { defineConfig } from "vitest/config";

export default defineConfig({
  resolve: { alias: { "@": path.resolve(__dirname) } },
  test: {
    projects: [
      { extends: true, test: { name: "unit", include: ["tests/unit/**/*.test.ts"] } },
      {
        extends: true,
        test: {
          name: "db",
          include: ["tests/db/**/*.test.ts"],
          setupFiles: ["tests/db/setup.ts"],
          fileParallelism: false,
          testTimeout: 30_000,
          hookTimeout: 60_000,
        },
      },
    ],
  },
});
```

Add to `package.json` scripts:

```json
"typecheck": "tsc --noEmit",
"test:unit": "vitest run --project unit",
"test:db": "vitest run --project db"
```

- [ ] **Step 6: Smoke test** – create `tests/unit/smoke.test.ts`:

```ts
import { expect, test } from "vitest";
test("vitest runs", () => { expect(1 + 1).toBe(2); });
```

- [ ] **Step 7: CI** – create `.github/workflows/ci.yml`:

```yaml
name: ci
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test:unit
```

- [ ] **Step 8: Verify**

Run: `npm run lint && npm run typecheck && npm run test:unit && npm run build`
Expected: all succeed; `test:unit` reports 1 passed.

- [ ] **Step 9: Commit**

```bash
git add -A -- . ':!*.docx'
git commit -m "chore: scaffold Next.js app with RTL layout, Vitest and CI"
```

---

## Task 2: Phone and ID helpers (TypeScript)

**Files:**
- Create: `lib/phone.ts`, `lib/il-id.ts`, `tests/unit/phone.test.ts`, `tests/unit/il-id.test.ts`

**Interfaces:**
- Produces: `normalizeIlPhone(input: string | null | undefined): string | null`, `isIlMobile(p: string): boolean`, `isIlPhone(p: string): boolean`, `isValidIlId(id: string): boolean`, `maskId(id: string | null): string | null`. These mirror the SQL functions of Task 3 exactly.

- [ ] **Step 1: Write failing tests**

`tests/unit/phone.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { isIlMobile, isIlPhone, normalizeIlPhone } from "@/lib/phone";

describe("normalizeIlPhone", () => {
  test.each([
    ["050-123 4567", "0501234567"],
    ["+972-50-1234567", "0501234567"],
    ["972501234567", "0501234567"],
    ["03-6191234", "036191234"],
    ["", null],
    ["   ", null],
    [null, null],
  ])("%s -> %s", (input, expected) => {
    expect(normalizeIlPhone(input)).toBe(expected);
  });
});

describe("validation", () => {
  test("mobile", () => {
    expect(isIlMobile("0501234567")).toBe(true);
    expect(isIlMobile("050123456")).toBe(false);
    expect(isIlMobile("036191234")).toBe(false);
  });
  test("any phone", () => {
    expect(isIlPhone("036191234")).toBe(true);
    expect(isIlPhone("0771234567")).toBe(true);
    expect(isIlPhone("0501234567")).toBe(true);
    expect(isIlPhone("051234567")).toBe(false);
    expect(isIlPhone("12345")).toBe(false);
  });
});
```

`tests/unit/il-id.test.ts`:

```ts
import { expect, test } from "vitest";
import { isValidIlId, maskId } from "@/lib/il-id";

test("valid ids", () => {
  expect(isValidIlId("123456782")).toBe(true);
  expect(isValidIlId("000000018")).toBe(true);
  expect(isValidIlId("00018")).toBe(true); // short form, padded to 9
});
test("invalid ids", () => {
  expect(isValidIlId("123456789")).toBe(false);
  expect(isValidIlId("12345678a")).toBe(false);
  expect(isValidIlId("18")).toBe(false); // fewer than 5 digits
  expect(isValidIlId("")).toBe(false);
});
test("mask", () => {
  expect(maskId("123456782")).toBe("•••••6782");
  expect(maskId(null)).toBeNull();
});
```

- [ ] **Step 2: Run** `npm run test:unit` — Expected: FAIL, cannot resolve `@/lib/phone`.

- [ ] **Step 3: Implement**

`lib/phone.ts`:

```ts
export function normalizeIlPhone(input: string | null | undefined): string | null {
  if (input == null || input.trim() === "") return null;
  let d = input.replace(/\D/g, "");
  if (d.startsWith("972")) d = "0" + d.slice(3);
  return d;
}
export const isIlMobile = (p: string) => /^05\d{8}$/.test(p);
export const isIlPhone = (p: string) =>
  isIlMobile(p) || /^0[23489]\d{7}$/.test(p) || /^07\d{8}$/.test(p);
```

`lib/il-id.ts`:

```ts
export function isValidIlId(id: string): boolean {
  if (!/^\d{5,9}$/.test(id)) return false;
  const s = id.padStart(9, "0");
  let total = 0;
  for (let i = 0; i < 9; i++) {
    let n = Number(s[i]) * (i % 2 === 0 ? 1 : 2);
    if (n > 9) n -= 9;
    total += n;
  }
  return total % 10 === 0;
}
export const maskId = (id: string | null) => (id ? "•••••" + id.slice(-4) : null);
```

- [ ] **Step 4: Run** `npm run test:unit` — Expected: PASS.

- [ ] **Step 5: Commit** `git add lib tests && git commit -m "feat: add Israeli phone and ID helpers"`

---

## Task 3: Core schema, SQL helpers, DB test harness

**Files:**
- Create: `supabase/config.toml` (via `npx supabase init`), `supabase/migrations/2026111501_core_schema.sql`, `tests/db/setup.ts`, `tests/db/helpers.ts`, `tests/db/schema.test.ts`

**Interfaces:**
- Consumes: env vars from `.env.test.local` (Task 0).
- Produces (SQL): tables `branches, profiles, families, children, leads, lead_children, class_types, instructors, groups, registrations, tasks, app_settings, cities, audit_log, email_outbox, login_codes, verified_sessions`; functions `app.normalize_il_phone(text)`, `app.is_il_mobile(text)`, `app.is_il_phone(text)`, `app.is_valid_il_id(text)`, `app.next_workday(date)`; error tokens `PHONE_INVALID`, `MOBILE_INVALID`.
- Produces (TS, `tests/db/helpers.ts`): `admin`, `tag()`, `makeUser(role, opts?)`, `makeFamily(t, over?)`, `makeChild(familyId, over?)`, `branchId()`, `cleanup(t, userIds)`.

- [ ] **Step 1: Init Supabase folder** – `npx supabase init` (answer No to editor questions). Confirm `npx supabase migration list` reaches the linked project.

- [ ] **Step 2: Test harness**

`tests/db/setup.ts`:

```ts
import { config } from "dotenv";
config({ path: ".env.test.local" });
for (const k of ["NEXT_PUBLIC_SUPABASE_URL", "NEXT_PUBLIC_SUPABASE_ANON_KEY", "SUPABASE_SERVICE_ROLE_KEY"]) {
  if (!process.env[k]) throw new Error(`Missing ${k} in .env.test.local`);
}
```

`tests/db/helpers.ts`:

```ts
import { createClient, type SupabaseClient } from "@supabase/supabase-js";

const url = process.env.NEXT_PUBLIC_SUPABASE_URL!;
const anon = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!;
const service = process.env.SUPABASE_SERVICE_ROLE_KEY!;

export const admin = createClient(url, service, { auth: { persistSession: false } });
export const tag = () => "t" + Date.now().toString(36) + Math.random().toString(36).slice(2, 6);

export type Role = "manager" | "secretary" | "ruchi" | "instructor";
export type TestUser = { client: SupabaseClient; id: string; email: string; sessionId: string };

export async function makeUser(role: Role, opts: { verified?: boolean; active?: boolean } = {}): Promise<TestUser> {
  const email = `test-${role}-${tag()}@hamasmer.test`;
  const password = "Test-" + tag() + "-pw1";
  const { data, error } = await admin.auth.admin.createUser({ email, password, email_confirm: true });
  if (error) throw error;
  const id = data.user.id;
  const p = await admin.from("profiles").insert({ id, full_name: `בודקת ${role}`, email, role, is_active: opts.active ?? true });
  if (p.error) throw p.error;
  const client = createClient(url, anon, { auth: { persistSession: false } });
  const s = await client.auth.signInWithPassword({ email, password });
  if (s.error) throw s.error;
  const payload = JSON.parse(Buffer.from(s.data.session!.access_token.split(".")[1], "base64url").toString());
  const sessionId: string = payload.session_id;
  if (opts.verified ?? true) {
    const v = await admin.from("verified_sessions").insert({
      session_id: sessionId, user_id: id, expires_at: new Date(Date.now() + 3600_000).toISOString(),
    });
    if (v.error) throw v.error;
  }
  return { client, id, email, sessionId };
}

export async function branchId(): Promise<string> {
  const { data, error } = await admin.from("branches").select("id").eq("name", "בני ברק").single();
  if (error) throw error;
  return data.id;
}

export async function makeFamily(t: string, over: Record<string, unknown> = {}) {
  const { data, error } = await admin.from("families")
    .insert({ last_name: `כהן-${t}`, contact_name: "רחל", branch_id: await branchId(), mobile_phone: "0501234567", ...over })
    .select().single();
  if (error) throw error;
  return data;
}

export async function makeChild(familyId: string, over: Record<string, unknown> = {}) {
  const { data, error } = await admin.from("children")
    .insert({ family_id: familyId, first_name: "יוסי", ...over }).select().single();
  if (error) throw error;
  return data;
}

/** Removes everything created under tag `t` and the given test users. */
export async function cleanup(t: string, userIds: string[] = []) {
  const fams = (await admin.from("families").select("id").like("last_name", `%-${t}`)).data ?? [];
  const famIds = fams.map((f) => f.id);
  const kids = famIds.length ? (await admin.from("children").select("id").in("family_id", famIds)).data ?? [] : [];
  const kidIds = kids.map((k) => k.id);
  const leads = famIds.length ? (await admin.from("leads").select("id").in("family_id", famIds)).data ?? [] : [];
  const leadIds = leads.map((l) => l.id);
  const regs = kidIds.length ? (await admin.from("registrations").select("id").in("child_id", kidIds)).data ?? [] : [];
  const regIds = regs.map((r) => r.id);

  if (leadIds.length) await admin.from("tasks").delete().in("lead_id", leadIds);
  if (regIds.length) await admin.from("tasks").delete().in("registration_id", regIds);
  if (famIds.length) await admin.from("tasks").delete().in("family_id", famIds);
  if (userIds.length) await admin.from("tasks").delete().in("assignee_id", userIds);
  await admin.from("tasks").delete().like("title", `%${t}%`);
  if (regIds.length) await admin.from("registrations").delete().in("id", regIds);
  if (leadIds.length) await admin.from("lead_children").delete().in("lead_id", leadIds);
  if (leadIds.length) await admin.from("leads").delete().in("id", leadIds);
  if (kidIds.length) await admin.from("children").delete().in("id", kidIds);
  if (famIds.length) await admin.from("families").delete().in("id", famIds);
  await admin.from("groups").delete().like("name", `%-${t}`);
  await admin.from("class_types").delete().like("name", `%-${t}`);
  for (const id of userIds) {
    await admin.from("audit_log").delete().eq("user_id", id);
    await admin.from("verified_sessions").delete().eq("user_id", id);
    await admin.from("login_codes").delete().eq("user_id", id);
    await admin.auth.admin.deleteUser(id); // profiles row cascades
  }
}
```

- [ ] **Step 3: Write the failing test** – `tests/db/schema.test.ts`:

```ts
import { afterAll, describe, expect, test } from "vitest";
import { admin, branchId, cleanup, makeChild, makeFamily, tag } from "./helpers";

const t = tag();
afterAll(() => cleanup(t));

describe("phones", () => {
  test("normalizes on save", async () => {
    const f = await makeFamily(t, { mobile_phone: "+972-50-765 4321", home_phone: "03-6191234" });
    expect(f.mobile_phone).toBe("0507654321");
    expect(f.home_phone).toBe("036191234");
  });
  test("rejects a landline in mobile_phone", async () => {
    const r = await admin.from("families").insert({ last_name: `לוי-${t}`, branch_id: await branchId(), mobile_phone: "036191234" });
    expect(r.error?.message).toContain("MOBILE_INVALID");
  });
  test("rejects garbage in home_phone", async () => {
    const r = await admin.from("families").insert({ last_name: `לוי-${t}`, branch_id: await branchId(), home_phone: "12345" });
    expect(r.error?.message).toContain("PHONE_INVALID");
  });
});

describe("child id number", () => {
  test("accepts valid, pads to 9 digits", async () => {
    const f = await makeFamily(t);
    const c = await makeChild(f.id, { id_number: "00018" });
    expect(c.id_number).toBe("000000018");
  });
  test("rejects bad check digit", async () => {
    const f = await makeFamily(t);
    const r = await admin.from("children").insert({ family_id: f.id, first_name: "דוד", id_number: "123456789" });
    expect(r.error).not.toBeNull();
  });
  test("rejects duplicate id", async () => {
    const f = await makeFamily(t);
    await makeChild(f.id, { id_number: "123456782" });
    const r = await admin.from("children").insert({ family_id: f.id, first_name: "משה", id_number: "123456782" });
    expect(r.error?.code).toBe("23505");
  });
});

test("seed data exists", async () => {
  const b = await admin.from("branches").select("name").order("name");
  expect(b.data?.map((x) => x.name)).toEqual(["בני ברק", "דימונה"]);
  const s = await admin.from("app_settings").select("current_school_year").single();
  expect(s.data?.current_school_year).toBe('תשפ"ז');
});
```

- [ ] **Step 4: Run** `npm run test:db` — Expected: FAIL (relation "branches" does not exist / schema cache error).

- [ ] **Step 5: Write the migration** – `supabase/migrations/2026111501_core_schema.sql`:

```sql
create schema if not exists app;
grant usage on schema app to authenticated, service_role;

-- ---------- helpers ----------
create function app.normalize_il_phone(p text) returns text
language plpgsql immutable as $$
declare d text;
begin
  if p is null or btrim(p) = '' then return null; end if;
  d := regexp_replace(p, '\D', '', 'g');
  if d like '972%' then d := '0' || substr(d, 4); end if;
  return d;
end $$;

create function app.is_il_mobile(p text) returns boolean
language sql immutable as $$ select p ~ '^05\d{8}$' $$;

create function app.is_il_phone(p text) returns boolean
language sql immutable as $$
  select p ~ '^05\d{8}$' or p ~ '^0[23489]\d{7}$' or p ~ '^07\d{8}$'
$$;

create function app.is_valid_il_id(p text) returns boolean
language plpgsql immutable as $$
declare s text; i int; n int; total int := 0;
begin
  if p is null then return true; end if;
  if p !~ '^\d{5,9}$' then return false; end if;
  s := lpad(p, 9, '0');
  for i in 1..9 loop
    n := substr(s, i, 1)::int * (case when i % 2 = 1 then 1 else 2 end);
    if n > 9 then n := n - 9; end if;
    total := total + n;
  end loop;
  return total % 10 = 0;
end $$;

-- Saturday -> Sunday. Phase 2 adds holidays here.
create function app.next_workday(d date) returns date
language sql immutable as $$
  select case when extract(dow from d) = 6 then d + 1 else d end
$$;

create function app.trg_set_updated_at() returns trigger
language plpgsql as $$
begin new.updated_at := now(); return new; end $$;

-- ---------- tables ----------
create table public.branches (
  id uuid primary key default gen_random_uuid(),
  name text not null unique,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);

create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text not null,
  email text not null,
  role text not null check (role in ('manager','secretary','ruchi','instructor')),
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.families (
  id uuid primary key default gen_random_uuid(),
  last_name text not null,
  contact_name text,
  address text,
  city text,
  branch_id uuid not null references public.branches(id),
  affiliation text check (affiliation in ('litai','hasidi','gur','other')),
  email text,
  home_phone text,
  mobile_phone text,
  extra_phone text,
  kosher_phone boolean not null default false,
  notes text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);
create index families_mobile_idx on public.families (mobile_phone);
create index families_home_idx on public.families (home_phone);
create index families_extra_idx on public.families (extra_phone);
create index families_last_name_idx on public.families (last_name);

create table public.children (
  id uuid primary key default gen_random_uuid(),
  family_id uuid not null references public.families(id),
  first_name text not null,
  id_number text check (app.is_valid_il_id(id_number)),
  birth_date date,
  health_fund text,
  school text,
  track text check (track in ('private','health_fund')),
  notes text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);
create unique index children_id_number_uq on public.children (id_number) where id_number is not null;
create index children_family_idx on public.children (family_id);

create table public.class_types (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  monthly_price numeric(8,2) not null check (monthly_price >= 0),
  valid_year text not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz,
  unique (name, valid_year)
);

create table public.instructors (
  id uuid primary key default gen_random_uuid(),
  first_name text not null,
  last_name text not null,
  mobile_phone text,
  email text,
  phone_code text unique,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);

create table public.groups (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  kind text not null default 'private' check (kind in ('private')),
  branch_id uuid not null references public.branches(id),
  weekday smallint check (weekday between 0 and 5),
  start_time time,
  duration_minutes int check (duration_minutes > 0),
  location text,
  instructor_id uuid references public.instructors(id),
  phone_code char(3) unique check (phone_code ~ '^\d{3}$'),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);

create table public.leads (
  id uuid primary key default gen_random_uuid(),
  family_id uuid not null references public.families(id),
  source text not null default 'other' check (source in ('ad','word_of_mouth','returning','phone_system','other')),
  status text not null default 'interested' check (status in ('interested','callback','contract_sent','registered','not_relevant')),
  callback_date date,
  not_relevant_reason text,
  contract_signed boolean not null default false,
  payment_set boolean not null default false,
  tzintuk_done boolean not null default false,
  contract_sent_at timestamptz,
  notes text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);
create index leads_family_idx on public.leads (family_id);
create index leads_status_idx on public.leads (status);

create table public.lead_children (
  id uuid primary key default gen_random_uuid(),
  lead_id uuid not null references public.leads(id) on delete cascade,
  child_id uuid not null references public.children(id),
  class_type_id uuid references public.class_types(id),
  group_id uuid references public.groups(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (lead_id, child_id)
);

create table public.registrations (
  id uuid primary key default gen_random_uuid(),
  child_id uuid not null references public.children(id),
  group_id uuid references public.groups(id),
  school_year text not null,
  class_type_id uuid references public.class_types(id),
  first_lesson_date date,
  status text not null default 'active' check (status in ('awaiting_contract','active','frozen','finished','left')),
  payment_method text check (payment_method in ('credit','standing_order','check','transfer','cash')),
  expected_payment_date date,
  source_lead_id uuid references public.leads(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);
create unique index registrations_lead_child_uq on public.registrations (source_lead_id, child_id)
  where source_lead_id is not null;
create index registrations_child_idx on public.registrations (child_id);
create index registrations_group_idx on public.registrations (group_id);

create table public.tasks (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  description text,
  assignee_id uuid not null references public.profiles(id),
  urgency text not null default 'medium' check (urgency in ('high','medium','low')),
  due_date date not null,
  status text not null default 'open' check (status in ('open','done','cancelled')),
  source text not null default 'manual' check (source in ('A1','A6','A7','A8','manual')),
  family_id uuid references public.families(id),
  lead_id uuid references public.leads(id),
  registration_id uuid references public.registrations(id),
  completed_at timestamptz,
  created_by uuid references public.profiles(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  archived_at timestamptz
);
create unique index tasks_open_lead_uq on public.tasks (source, lead_id)
  where status = 'open' and source in ('A1','A7');
create unique index tasks_open_reg_uq on public.tasks (source, registration_id)
  where status = 'open' and source in ('A6','A8');
create index tasks_assignee_due_idx on public.tasks (assignee_id, due_date) where status = 'open';

create table public.app_settings (
  id int primary key default 1 check (id = 1),
  current_school_year text not null,
  default_task_assignee_id uuid references public.profiles(id),
  app_url text not null default 'http://localhost:3000',
  updated_at timestamptz not null default now()
);

create table public.cities (
  id uuid primary key default gen_random_uuid(),
  name text not null unique
);

create table public.audit_log (
  id bigint generated always as identity primary key,
  table_name text not null,
  row_id uuid,
  action text not null check (action in ('insert','update','delete')),
  old_data jsonb,
  new_data jsonb,
  user_id uuid,
  at timestamptz not null default now()
);
create index audit_log_row_idx on public.audit_log (table_name, row_id);
create index audit_log_at_idx on public.audit_log (at);

create table public.email_outbox (
  id uuid primary key default gen_random_uuid(),
  to_email text not null,
  subject text not null,
  body_html text not null,
  task_id uuid references public.tasks(id) on delete cascade,
  send_after timestamptz not null default now(),
  status text not null default 'pending' check (status in ('pending','sent','failed','cancelled')),
  attempts int not null default 0,
  last_error text,
  sent_at timestamptz,
  created_at timestamptz not null default now()
);
create index email_outbox_due_idx on public.email_outbox (send_after) where status = 'pending';

create table public.login_codes (
  user_id uuid primary key references auth.users(id) on delete cascade,
  session_id uuid not null,
  code_hash text not null,
  expires_at timestamptz not null,
  attempts int not null default 0,
  locked_until timestamptz,
  created_at timestamptz not null default now()
);

create table public.verified_sessions (
  session_id uuid primary key,
  user_id uuid not null references auth.users(id) on delete cascade,
  expires_at timestamptz not null,
  created_at timestamptz not null default now()
);

-- ---------- updated_at on every table that has it ----------
do $do$
declare t text;
begin
  foreach t in array array['branches','profiles','families','children','class_types','instructors',
                           'groups','leads','lead_children','registrations','tasks','app_settings'] loop
    execute format('create trigger set_updated_at before update on public.%I
                    for each row execute function app.trg_set_updated_at()', t);
  end loop;
end $do$;

-- ---------- phone normalisation ----------
create function app.trg_family_phones() returns trigger
language plpgsql as $$
begin
  new.mobile_phone := app.normalize_il_phone(new.mobile_phone);
  new.home_phone   := app.normalize_il_phone(new.home_phone);
  new.extra_phone  := app.normalize_il_phone(new.extra_phone);
  if new.mobile_phone is not null and not app.is_il_mobile(new.mobile_phone) then
    raise exception 'MOBILE_INVALID';
  end if;
  if (new.home_phone is not null and not app.is_il_phone(new.home_phone))
     or (new.extra_phone is not null and not app.is_il_phone(new.extra_phone)) then
    raise exception 'PHONE_INVALID';
  end if;
  return new;
end $$;
create trigger family_phones before insert or update on public.families
  for each row execute function app.trg_family_phones();

create function app.trg_instructor_phone() returns trigger
language plpgsql as $$
begin
  new.mobile_phone := app.normalize_il_phone(new.mobile_phone);
  if new.mobile_phone is not null and not app.is_il_mobile(new.mobile_phone) then
    raise exception 'MOBILE_INVALID';
  end if;
  return new;
end $$;
create trigger instructor_phone before insert or update on public.instructors
  for each row execute function app.trg_instructor_phone();

create function app.trg_child_id_pad() returns trigger
language plpgsql as $$
begin
  if new.id_number is not null then
    new.id_number := nullif(btrim(new.id_number), '');
    if new.id_number ~ '^\d{5,9}$' then new.id_number := lpad(new.id_number, 9, '0'); end if;
  end if;
  return new;
end $$;
create trigger child_id_pad before insert or update on public.children
  for each row execute function app.trg_child_id_pad();

-- ---------- RLS on, no policies yet (Task 4 adds them) ----------
do $do$
declare t text;
begin
  foreach t in array array['branches','profiles','families','children','class_types','instructors','groups',
                           'leads','lead_children','registrations','tasks','app_settings','cities',
                           'audit_log','email_outbox','login_codes','verified_sessions'] loop
    execute format('alter table public.%I enable row level security', t);
  end loop;
end $do$;

-- ---------- seed ----------
insert into public.branches (name) values ('בני ברק'), ('דימונה');
insert into public.app_settings (id, current_school_year) values (1, 'תשפ"ז');
```

- [ ] **Step 6: Apply** – `npx supabase db push` — Expected: "Applying migration 2026111501_core_schema.sql... Finished".

- [ ] **Step 7: Run** `npm run test:db` — Expected: PASS (7 tests).

- [ ] **Step 8: Commit** `git add supabase tests && git commit -m "feat(db): core schema, phone/ID rules and DB test harness"`

---

## Task 4: Roles, verified sessions and RLS policies

**Files:**
- Create: `supabase/migrations/2026111502_rls.sql`, `tests/db/rls.test.ts`

**Interfaces:**
- Consumes: tables from Task 3; `makeUser` from `tests/db/helpers.ts`.
- Produces (SQL): `app.is_verified() returns boolean`, `app.user_role() returns text` (NULL when not verified or inactive), RPC `public.my_session() returns json` → `{ "verified": boolean, "role": text | null }`, RPC `public.find_family_by_phone(p text) returns setof families`, views `children_view`, `families_view`, `groups_view`; error tokens `TASK_STATUS_ONLY`, `RESTORE_MANAGER_ONLY`.

- [ ] **Step 1: Write the failing test** – `tests/db/rls.test.ts`:

```ts
import { afterAll, beforeAll, describe, expect, test } from "vitest";
import { admin, branchId, cleanup, makeChild, makeFamily, makeUser, tag, type TestUser } from "./helpers";

const t = tag();
let manager: TestUser, secretary: TestUser, ruchi: TestUser, instructor: TestUser,
    unverified: TestUser, inactive: TestUser;
let familyId: string;

beforeAll(async () => {
  [manager, secretary, ruchi, instructor] = await Promise.all([
    makeUser("manager"), makeUser("secretary"), makeUser("ruchi"), makeUser("instructor"),
  ]);
  unverified = await makeUser("secretary", { verified: false });
  inactive = await makeUser("secretary", { active: false });
  const f = await makeFamily(t);
  familyId = f.id;
  await makeChild(f.id, { id_number: "123456782" });
  await admin.from("tasks").insert([
    { title: `משימה לרוחי ${t}`, assignee_id: ruchi.id, due_date: "2026-12-01" },
    { title: `משימה למזכירה ${t}`, assignee_id: secretary.id, due_date: "2026-12-01" },
  ]);
});
afterAll(() => cleanup(t, [manager, secretary, ruchi, instructor, unverified, inactive].map((u) => u.id)));

const count = async (u: TestUser, table: string) =>
  (await u.client.from(table).select("id").like(table === "tasks" ? "title" : "last_name", `%${t}%`)).data?.length ?? -1;

describe("families", () => {
  test("manager and secretary see them", async () => {
    expect(await count(manager, "families")).toBe(1);
    expect(await count(secretary, "families")).toBe(1);
  });
  test("ruchi, instructor, unverified and inactive see nothing", async () => {
    for (const u of [ruchi, instructor, unverified, inactive]) expect(await count(u, "families")).toBe(0);
  });
  test("secretary can insert and update, nobody can delete", async () => {
    const ins = await secretary.client.from("families")
      .insert({ last_name: `לוי-${t}`, branch_id: await branchId() }).select().single();
    expect(ins.error).toBeNull();
    const upd = await secretary.client.from("families").update({ city: "בני ברק" }).eq("id", ins.data!.id).select();
    expect(upd.data?.length).toBe(1);
    await manager.client.from("families").delete().eq("id", ins.data!.id);
    const still = await admin.from("families").select("id").eq("id", ins.data!.id);
    expect(still.data?.length).toBe(1);
  });
  test("ruchi cannot insert", async () => {
    const r = await ruchi.client.from("families").insert({ last_name: `לוי-${t}`, branch_id: await branchId() });
    expect(r.error).not.toBeNull();
  });
});

describe("tasks", () => {
  test("ruchi sees only her own", async () => { expect(await count(ruchi, "tasks")).toBe(1); });
  test("secretary sees all", async () => { expect(await count(secretary, "tasks")).toBe(2); });
  test("ruchi may change status but not title", async () => {
    const mine = (await ruchi.client.from("tasks").select("id").like("title", `%${t}%`).single()).data!;
    const bad = await ruchi.client.from("tasks").update({ title: "אחר" }).eq("id", mine.id);
    expect(bad.error?.message).toContain("TASK_STATUS_ONLY");
    const ok = await ruchi.client.from("tasks").update({ status: "done" }).eq("id", mine.id).select().single();
    expect(ok.data?.status).toBe("done");
    expect(ok.data?.completed_at).not.toBeNull();
  });
});

describe("reference and admin tables", () => {
  test("secretary reads class_types but cannot insert", async () => {
    const r = await secretary.client.from("class_types").insert({ name: `נגרות-${t}`, monthly_price: 300, valid_year: 'תשפ"ז' });
    expect(r.error).not.toBeNull();
    const m = await manager.client.from("class_types").insert({ name: `נגרות-${t}`, monthly_price: 300, valid_year: 'תשפ"ז' });
    expect(m.error).toBeNull();
    const read = await secretary.client.from("class_types").select("id").like("name", `%-${t}`);
    expect(read.data?.length).toBe(1);
  });
  test("only service role touches login tables", async () => {
    for (const u of [manager, secretary]) {
      expect((await u.client.from("verified_sessions").select("session_id")).data?.length).toBe(0);
      expect((await u.client.from("login_codes").select("user_id")).data?.length).toBe(0);
    }
  });
  test("secretary cannot update settings, manager can", async () => {
    const s = await secretary.client.from("app_settings").update({ current_school_year: "x" }).eq("id", 1).select();
    expect(s.data?.length ?? 0).toBe(0);
    const before = (await admin.from("app_settings").select("current_school_year").single()).data!.current_school_year;
    const m = await manager.client.from("app_settings").update({ current_school_year: before }).eq("id", 1).select();
    expect(m.data?.length).toBe(1);
  });
});

describe("session and helpers", () => {
  test("my_session", async () => {
    expect((await secretary.client.rpc("my_session")).data).toEqual({ verified: true, role: "secretary" });
    expect((await unverified.client.rpc("my_session")).data).toEqual({ verified: false, role: null });
    expect((await inactive.client.rpc("my_session")).data).toEqual({ verified: true, role: null });
  });
  test("expired verification hides data", async () => {
    await admin.from("verified_sessions").update({ expires_at: new Date(Date.now() - 1000).toISOString() })
      .eq("session_id", instructor.sessionId);
    expect((await instructor.client.rpc("my_session")).data.verified).toBe(false);
  });
  test("find_family_by_phone respects RLS", async () => {
    const hit = await secretary.client.rpc("find_family_by_phone", { p: "050-1234567" });
    expect(hit.data?.some((f: { id: string }) => f.id === familyId)).toBe(true);
    const miss = await ruchi.client.rpc("find_family_by_phone", { p: "050-1234567" });
    expect(miss.data?.length ?? 0).toBe(0);
  });
  test("children_view masks the id", async () => {
    const r = await secretary.client.from("children_view").select("*").eq("family_id", familyId).single();
    expect(r.data?.id_masked).toBe("•••••6782");
    expect(r.data).not.toHaveProperty("id_number");
  });
});

describe("archive", () => {
  test("secretary archives, only manager restores", async () => {
    const a = await secretary.client.from("families").update({ archived_at: new Date().toISOString() }).eq("id", familyId).select();
    expect(a.data?.length).toBe(1);
    const r = await secretary.client.from("families").update({ archived_at: null }).eq("id", familyId);
    expect(r.error?.message).toContain("RESTORE_MANAGER_ONLY");
    const m = await manager.client.from("families").update({ archived_at: null }).eq("id", familyId).select();
    expect(m.data?.length).toBe(1);
  });
});
```

- [ ] **Step 2: Run** `npm run test:db -- rls` — Expected: FAIL (manager sees 0 families; `my_session` not found).

- [ ] **Step 3: Write the migration** – `supabase/migrations/2026111502_rls.sql`:

```sql
create function app.is_verified() returns boolean
language sql stable security definer set search_path = '' as $$
  select exists (
    select 1 from public.verified_sessions v
    where v.session_id = nullif(auth.jwt() ->> 'session_id', '')::uuid
      and v.user_id = auth.uid()
      and v.expires_at > now()
  )
$$;

create function app.user_role() returns text
language sql stable security definer set search_path = '' as $$
  select p.role from public.profiles p
  where p.id = auth.uid() and p.is_active and app.is_verified()
$$;

create function public.my_session() returns json
language sql stable security definer set search_path = '' as $$
  select json_build_object('verified', app.is_verified(), 'role', app.user_role())
$$;
revoke all on function public.my_session() from public, anon;
grant execute on function public.my_session() to authenticated;

create function public.find_family_by_phone(p text) returns setof public.families
language sql stable security invoker set search_path = '' as $$
  select f.* from public.families f
  where f.archived_at is null
    and app.normalize_il_phone(p) in (f.mobile_phone, f.home_phone, f.extra_phone)
$$;
revoke all on function public.find_family_by_phone(text) from public, anon;
grant execute on function public.find_family_by_phone(text) to authenticated, service_role;

-- staff tables: manager + secretary read/insert/update
do $do$
declare t text;
begin
  foreach t in array array['families','children','leads','lead_children','registrations','groups','instructors'] loop
    execute format($f$create policy staff_select on public.%I for select to authenticated
      using ((select app.user_role()) in ('manager','secretary'))$f$, t);
    execute format($f$create policy staff_insert on public.%I for insert to authenticated
      with check ((select app.user_role()) in ('manager','secretary'))$f$, t);
    execute format($f$create policy staff_update on public.%I for update to authenticated
      using ((select app.user_role()) in ('manager','secretary'))
      with check ((select app.user_role()) in ('manager','secretary'))$f$, t);
  end loop;
  -- reference tables: both read, manager writes
  foreach t in array array['class_types','branches','cities'] loop
    execute format($f$create policy ref_select on public.%I for select to authenticated
      using ((select app.user_role()) in ('manager','secretary'))$f$, t);
    execute format($f$create policy ref_insert on public.%I for insert to authenticated
      with check ((select app.user_role()) = 'manager')$f$, t);
    execute format($f$create policy ref_update on public.%I for update to authenticated
      using ((select app.user_role()) = 'manager') with check ((select app.user_role()) = 'manager')$f$, t);
  end loop;
end $do$;

create policy lead_children_delete on public.lead_children for delete to authenticated
  using ((select app.user_role()) in ('manager','secretary'));

-- tasks
create policy tasks_select on public.tasks for select to authenticated using (
  (select app.user_role()) in ('manager','secretary')
  or ((select app.user_role()) = 'ruchi' and assignee_id = (select auth.uid())));
create policy tasks_insert on public.tasks for insert to authenticated
  with check ((select app.user_role()) in ('manager','secretary'));
create policy tasks_update on public.tasks for update to authenticated using (
  (select app.user_role()) in ('manager','secretary')
  or ((select app.user_role()) = 'ruchi' and assignee_id = (select auth.uid())))
  with check (
  (select app.user_role()) in ('manager','secretary')
  or ((select app.user_role()) = 'ruchi' and assignee_id = (select auth.uid())));

-- profiles
create policy profiles_select on public.profiles for select to authenticated using (
  (select app.user_role()) in ('manager','secretary')
  or ((select app.user_role()) is not null and id = (select auth.uid())));
create policy profiles_update on public.profiles for update to authenticated
  using ((select app.user_role()) = 'manager') with check ((select app.user_role()) = 'manager');

-- settings, audit, outbox
create policy settings_select on public.app_settings for select to authenticated
  using ((select app.user_role()) in ('manager','secretary'));
create policy settings_update on public.app_settings for update to authenticated
  using ((select app.user_role()) = 'manager') with check ((select app.user_role()) = 'manager');
create policy audit_select on public.audit_log for select to authenticated
  using ((select app.user_role()) = 'manager');
create policy outbox_select on public.email_outbox for select to authenticated
  using ((select app.user_role()) = 'manager');
-- login_codes, verified_sessions: no policies => service role only.

-- task guard: completed_at + ruchi may change status only
create function app.trg_tasks_before() returns trigger
language plpgsql security definer set search_path = '' as $$
begin
  if tg_op = 'UPDATE' and app.user_role() = 'ruchi' then
    if (to_jsonb(new) - array['status','completed_at','updated_at'])
       is distinct from (to_jsonb(old) - array['status','completed_at','updated_at']) then
      raise exception 'TASK_STATUS_ONLY';
    end if;
  end if;
  if new.status = 'done' and (tg_op = 'INSERT' or old.status <> 'done') then
    new.completed_at := now();
  elsif new.status <> 'done' then
    new.completed_at := null;
  end if;
  if tg_op = 'INSERT' and new.created_by is null then new.created_by := auth.uid(); end if;
  return new;
end $$;
create trigger tasks_before before insert or update on public.tasks
  for each row execute function app.trg_tasks_before();

-- restore from archive: manager (or service role) only
create function app.trg_restore_guard() returns trigger
language plpgsql security definer set search_path = '' as $$
begin
  if old.archived_at is not null and new.archived_at is null
     and auth.uid() is not null and app.user_role() is distinct from 'manager' then
    raise exception 'RESTORE_MANAGER_ONLY';
  end if;
  return new;
end $$;
do $do$
declare t text;
begin
  foreach t in array array['branches','families','children','class_types','instructors','groups',
                           'leads','registrations','tasks'] loop
    execute format('create trigger restore_guard before update on public.%I
                    for each row execute function app.trg_restore_guard()', t);
  end loop;
end $do$;

-- views (run with the caller's permissions)
create view public.children_view with (security_invoker = true) as
select c.id, c.family_id, c.first_name, c.birth_date, c.health_fund, c.school, c.track, c.notes,
       c.created_at, c.updated_at, c.archived_at,
       case when c.id_number is null then null else '•••••' || right(c.id_number, 4) end as id_masked,
       extract(year from age(c.birth_date))::int as age,
       coalesce(r.status, l.status) as current_status,
       r.group_id as current_group_id
from public.children c
left join lateral (
  select rg.status, rg.group_id from public.registrations rg
  where rg.child_id = c.id and rg.archived_at is null
  order by (rg.status = 'active') desc, rg.created_at desc limit 1) r on true
left join lateral (
  select ld.status from public.lead_children lc join public.leads ld on ld.id = lc.lead_id
  where lc.child_id = c.id and ld.archived_at is null
  order by ld.created_at desc limit 1) l on true;

create view public.families_view with (security_invoker = true) as
select f.*, (
  select count(distinct ch.id) from public.children ch
  join public.registrations rg on rg.child_id = ch.id
  where ch.family_id = f.id and rg.status = 'active' and rg.archived_at is null)::int as active_children_count
from public.families f;

create view public.groups_view with (security_invoker = true) as
select g.*, (
  select count(*) from public.registrations rg
  where rg.group_id = g.id and rg.status = 'active' and rg.archived_at is null)::int as active_count
from public.groups g;
```

- [ ] **Step 4: Apply and run** – `npx supabase db push && npm run test:db` — Expected: all DB tests PASS.

- [ ] **Step 5: Commit** `git add supabase tests && git commit -m "feat(db): role-based RLS gated on verified sessions"`

---

## Task 5: Audit log

**Files:**
- Create: `supabase/migrations/2026111503_audit.sql`, `tests/db/audit.test.ts`

**Interfaces:**
- Consumes: `audit_log` table, `audit_select` policy.
- Produces: trigger `audit` on all business tables. `audit_log.new_data` holds only changed columns on update; `user_id` is the acting user (NULL for service role).

- [ ] **Step 1: Write the failing test** – `tests/db/audit.test.ts`:

```ts
import { afterAll, beforeAll, expect, test } from "vitest";
import { cleanup, makeFamily, makeUser, tag, type TestUser } from "./helpers";

const t = tag();
let manager: TestUser, secretary: TestUser;
beforeAll(async () => { [manager, secretary] = await Promise.all([makeUser("manager"), makeUser("secretary")]); });
afterAll(() => cleanup(t, [manager.id, secretary.id]));

test("update is logged with only the changed columns and the acting user", async () => {
  const f = await makeFamily(t, { city: "דימונה" });
  await secretary.client.from("families").update({ city: "בני ברק" }).eq("id", f.id);
  const log = await manager.client.from("audit_log").select("*")
    .eq("table_name", "families").eq("row_id", f.id).eq("action", "update");
  expect(log.data?.length).toBe(1);
  expect(log.data![0].new_data).toEqual({ city: "בני ברק" });
  expect(log.data![0].old_data).toEqual({ city: "דימונה" });
  expect(log.data![0].user_id).toBe(secretary.id);
});

test("a no-op update is not logged", async () => {
  const f = await makeFamily(t, { city: "דימונה" });
  await secretary.client.from("families").update({ city: "דימונה" }).eq("id", f.id);
  const log = await manager.client.from("audit_log").select("id").eq("row_id", f.id).eq("action", "update");
  expect(log.data?.length).toBe(0);
});

test("secretary cannot read the audit log", async () => {
  const log = await secretary.client.from("audit_log").select("id").limit(1);
  expect(log.data?.length).toBe(0);
});
```

- [ ] **Step 2: Run** `npm run test:db -- audit` — Expected: FAIL (0 log rows).

- [ ] **Step 3: Write the migration** – `supabase/migrations/2026111503_audit.sql`:

```sql
create function app.trg_audit() returns trigger
language plpgsql security definer set search_path = '' as $$
declare o jsonb; n jsonb;
begin
  if tg_op = 'INSERT' then
    insert into public.audit_log (table_name, row_id, action, new_data, user_id)
    values (tg_table_name, new.id, 'insert', to_jsonb(new), auth.uid());
    return new;
  elsif tg_op = 'DELETE' then
    insert into public.audit_log (table_name, row_id, action, old_data, user_id)
    values (tg_table_name, old.id, 'delete', to_jsonb(old), auth.uid());
    return old;
  end if;
  select jsonb_object_agg(e.key, e.value) into n
  from jsonb_each(to_jsonb(new)) e
  where e.key <> 'updated_at' and (to_jsonb(old) -> e.key) is distinct from e.value;
  if n is null then return new; end if;
  select jsonb_object_agg(k, to_jsonb(old) -> k) into o from jsonb_object_keys(n) k;
  insert into public.audit_log (table_name, row_id, action, old_data, new_data, user_id)
  values (tg_table_name, new.id, 'update', o, n, auth.uid());
  return new;
end $$;

do $do$
declare t text;
begin
  foreach t in array array['branches','profiles','families','children','class_types','instructors','groups',
                           'leads','lead_children','registrations','tasks'] loop
    execute format('create trigger audit after insert or update or delete on public.%I
                    for each row execute function app.trg_audit()', t);
  end loop;
end $do$;
```

- [ ] **Step 4: Apply and run** – `npx supabase db push && npm run test:db` — Expected: PASS.

- [ ] **Step 5: Commit** `git add supabase tests && git commit -m "feat(db): audit log for all business tables"`

---

## Task 6: Automations A1, A6, A8

**Files:**
- Create: `supabase/migrations/2026111504_automations.sql`, `tests/db/automations.test.ts`

**Interfaces:**
- Consumes: `app.next_workday`, `tasks` unique partial indexes, `app_settings`.
- Produces: `app.default_assignee() returns uuid`; triggers `leads_before`, `leads_after`, `registrations_after`; error tokens `LEAD_NO_CHILDREN`, `LEAD_FLAGS_MISSING`, `NO_ASSIGNEE`. Task titles: A1 `לחזור למשפחת <last_name>`, A6 `לשבץ את <first_name> <last_name> לקבוצה`, A8 `לבדוק אם הכסף הגיע – <first_name> <last_name>`.

- [ ] **Step 1: Write the failing test** – `tests/db/automations.test.ts`:

```ts
import { afterAll, beforeAll, describe, expect, test } from "vitest";
import { admin, branchId, cleanup, makeChild, makeFamily, makeUser, tag, type TestUser } from "./helpers";

const t = tag();
let manager: TestUser, secretary: TestUser;
let prevAssignee: string | null;

beforeAll(async () => {
  [manager, secretary] = await Promise.all([makeUser("manager"), makeUser("secretary")]);
  prevAssignee = (await admin.from("app_settings").select("default_task_assignee_id").single()).data!.default_task_assignee_id;
  await admin.from("app_settings").update({ default_task_assignee_id: secretary.id }).eq("id", 1);
});
afterAll(async () => {
  await admin.from("app_settings").update({ default_task_assignee_id: prevAssignee }).eq("id", 1);
  await cleanup(t, [manager.id, secretary.id]);
});

const newLead = async (over: Record<string, unknown> = {}) => {
  const f = await makeFamily(t, { home_phone: "036191234" });
  const r = await secretary.client.from("leads").insert({ family_id: f.id, ...over }).select().single();
  if (r.error) throw r.error;
  return { family: f, lead: r.data };
};
const tasksOf = async (col: "lead_id" | "registration_id", id: string, source: string) =>
  (await admin.from("tasks").select("*").eq(col, id).eq("source", source).order("created_at")).data ?? [];

describe("A1 callback date", () => {
  test("creates a task with phones, assigned to the default assignee", async () => {
    const { lead, family } = await newLead({ status: "callback", callback_date: "2026-12-02" });
    const [task] = await tasksOf("lead_id", lead.id, "A1");
    expect(task.title).toBe(`לחזור למשפחת ${family.last_name}`);
    expect(task.due_date).toBe("2026-12-02");
    expect(task.assignee_id).toBe(secretary.id);
    expect(task.description).toContain("0501234567");
    expect(task.description).toContain("036191234");
    expect(task.family_id).toBe(family.id);
  });
  test("changing the date moves the same task; Saturday moves to Sunday", async () => {
    const { lead } = await newLead({ callback_date: "2026-12-02" });
    await secretary.client.from("leads").update({ callback_date: "2026-12-05" }).eq("id", lead.id); // Saturday
    const tasks = await tasksOf("lead_id", lead.id, "A1");
    expect(tasks.length).toBe(1);
    expect(tasks[0].due_date).toBe("2026-12-06");
  });
  test("clearing the date or closing the lead cancels the task", async () => {
    const a = await newLead({ callback_date: "2026-12-02" });
    await secretary.client.from("leads").update({ callback_date: null }).eq("id", a.lead.id);
    expect((await tasksOf("lead_id", a.lead.id, "A1"))[0].status).toBe("cancelled");
    const b = await newLead({ callback_date: "2026-12-02" });
    await secretary.client.from("leads").update({ status: "not_relevant" }).eq("id", b.lead.id);
    expect((await tasksOf("lead_id", b.lead.id, "A1"))[0].status).toBe("cancelled");
  });
  test("a completed task is not recreated by an unrelated edit", async () => {
    const { lead } = await newLead({ callback_date: "2026-12-02" });
    await admin.from("tasks").update({ status: "done" }).eq("lead_id", lead.id);
    await secretary.client.from("leads").update({ notes: "דיברנו", status: "interested" }).eq("id", lead.id);
    expect((await tasksOf("lead_id", lead.id, "A1")).length).toBe(1);
  });
});

describe("A6 three flags create registrations", () => {
  const flags = { contract_signed: true, payment_set: true, tzintuk_done: true };
  test("full flow", async () => {
    const { lead, family } = await newLead({ callback_date: "2026-12-02" });
    const kid1 = await makeChild(family.id, { first_name: "יוסי" });
    const kid2 = await makeChild(family.id, { first_name: "דוד" });
    const group = (await admin.from("groups").insert({ name: `נגרות א-${t}`, branch_id: await branchId(), weekday: 0 }).select().single()).data!;
    await secretary.client.from("lead_children").insert([
      { lead_id: lead.id, child_id: kid1.id, group_id: group.id },
      { lead_id: lead.id, child_id: kid2.id },
    ]);
    const upd = await secretary.client.from("leads").update(flags).eq("id", lead.id).select().single();
    expect(upd.data?.status).toBe("registered");

    const regs = (await admin.from("registrations").select("*").eq("source_lead_id", lead.id)).data!;
    expect(regs.length).toBe(2);
    expect(regs.every((r) => r.status === "active" && r.school_year === 'תשפ"ז')).toBe(true);
    const withGroup = regs.find((r) => r.child_id === kid1.id)!;
    const noGroup = regs.find((r) => r.child_id === kid2.id)!;
    expect(withGroup.group_id).toBe(group.id);
    expect((await tasksOf("registration_id", withGroup.id, "A6")).length).toBe(0);
    const [a6] = await tasksOf("registration_id", noGroup.id, "A6");
    expect(a6.title).toBe(`לשבץ את דוד ${family.last_name} לקבוצה`);
    expect(a6.status).toBe("open");
    expect((await tasksOf("lead_id", lead.id, "A1"))[0].status).toBe("cancelled");

    // assigning a group closes the A6 task
    await secretary.client.from("registrations").update({ group_id: group.id }).eq("id", noGroup.id);
    expect((await tasksOf("registration_id", noGroup.id, "A6"))[0].status).toBe("done");

    // untick + retick does not duplicate
    await secretary.client.from("leads").update({ payment_set: false }).eq("id", lead.id);
    await secretary.client.from("leads").update({ payment_set: true }).eq("id", lead.id);
    expect((await admin.from("registrations").select("id").eq("source_lead_id", lead.id)).data!.length).toBe(2);
    expect((await tasksOf("registration_id", noGroup.id, "A6")).length).toBe(1);
  });
  test("a lead without children cannot be registered", async () => {
    const { lead } = await newLead();
    const r = await secretary.client.from("leads").update(flags).eq("id", lead.id);
    expect(r.error?.message).toContain("LEAD_NO_CHILDREN");
  });
  test("status cannot be set to registered by hand", async () => {
    const { lead } = await newLead();
    const r = await secretary.client.from("leads").update({ status: "registered" }).eq("id", lead.id);
    expect(r.error?.message).toContain("LEAD_FLAGS_MISSING");
  });
});

describe("A8 expected payment date", () => {
  test("create, move, cancel", async () => {
    const f = await makeFamily(t);
    const kid = await makeChild(f.id, { first_name: "משה" });
    const reg = (await secretary.client.from("registrations")
      .insert({ child_id: kid.id, school_year: 'תשפ"ז', expected_payment_date: "2026-12-10" }).select().single()).data!;
    let tasks = await tasksOf("registration_id", reg.id, "A8");
    expect(tasks.length).toBe(1);
    expect(tasks[0].title).toBe(`לבדוק אם הכסף הגיע – משה ${f.last_name}`);
    expect(tasks[0].due_date).toBe("2026-12-10");
    // a manual registration without a group does not create an A6 task
    expect((await tasksOf("registration_id", reg.id, "A6")).length).toBe(0);

    await secretary.client.from("registrations").update({ expected_payment_date: "2026-12-15" }).eq("id", reg.id);
    tasks = await tasksOf("registration_id", reg.id, "A8");
    expect(tasks.length).toBe(1);
    expect(tasks[0].due_date).toBe("2026-12-15");

    await secretary.client.from("registrations").update({ status: "left" }).eq("id", reg.id);
    expect((await tasksOf("registration_id", reg.id, "A8"))[0].status).toBe("cancelled");
  });
});

test("falls back to an active manager when no default assignee is set", async () => {
  await admin.from("app_settings").update({ default_task_assignee_id: null }).eq("id", 1);
  const { lead } = await newLead({ callback_date: "2026-12-02" });
  const [task] = await tasksOf("lead_id", lead.id, "A1");
  const p = await admin.from("profiles").select("role").eq("id", task.assignee_id).single();
  expect(p.data?.role).toBe("manager");
  await admin.from("app_settings").update({ default_task_assignee_id: secretary.id }).eq("id", 1);
});
```

- [ ] **Step 2: Run** `npm run test:db -- automations` — Expected: FAIL (no tasks created).

- [ ] **Step 3: Write the migration** – `supabase/migrations/2026111504_automations.sql`:

```sql
create function app.default_assignee() returns uuid
language plpgsql stable security definer set search_path = '' as $$
declare v uuid;
begin
  select s.default_task_assignee_id into v from public.app_settings s
  join public.profiles p on p.id = s.default_task_assignee_id and p.is_active
  where s.id = 1;
  if v is null then
    select p.id into v from public.profiles p
    where p.role = 'manager' and p.is_active order by p.created_at limit 1;
  end if;
  if v is null then raise exception 'NO_ASSIGNEE'; end if;
  return v;
end $$;

-- ---------- leads ----------
create function app.trg_leads_before() returns trigger
language plpgsql security definer set search_path = '' as $$
declare all_new boolean; all_old boolean;
begin
  all_new := new.contract_signed and new.payment_set and new.tzintuk_done;
  all_old := tg_op = 'UPDATE' and old.contract_signed and old.payment_set and old.tzintuk_done;

  if tg_op = 'UPDATE' and all_new and not all_old then
    if not exists (select 1 from public.lead_children lc where lc.lead_id = new.id) then
      raise exception 'LEAD_NO_CHILDREN';
    end if;
    new.status := 'registered';
  end if;

  if new.status = 'registered' and (tg_op = 'INSERT' or old.status <> 'registered') and not all_new then
    raise exception 'LEAD_FLAGS_MISSING';
  end if;

  if new.status = 'contract_sent' and (tg_op = 'INSERT' or old.status <> 'contract_sent') then
    new.contract_sent_at := now();
  end if;
  return new;
end $$;
create trigger leads_before before insert or update on public.leads
  for each row execute function app.trg_leads_before();

create function app.trg_leads_after() returns trigger
language plpgsql security definer set search_path = '' as $$
declare fam public.families; v_due date;
begin
  -- A1
  if new.callback_date is null or new.status in ('registered','not_relevant') then
    update public.tasks set status = 'cancelled'
    where source = 'A1' and lead_id = new.id and status = 'open';
  elsif tg_op = 'INSERT' or new.callback_date is distinct from old.callback_date then
    select * into fam from public.families where id = new.family_id;
    v_due := app.next_workday(new.callback_date);
    update public.tasks set due_date = v_due
    where source = 'A1' and lead_id = new.id and status = 'open';
    if not found then
      insert into public.tasks (title, description, assignee_id, urgency, due_date, source, lead_id, family_id)
      values ('לחזור למשפחת ' || fam.last_name,
              'טלפון בבית: ' || coalesce(fam.home_phone, '—') || ' · פלאפון: ' || coalesce(fam.mobile_phone, '—'),
              app.default_assignee(), 'medium', v_due, 'A1', new.id, new.family_id);
    end if;
  end if;

  -- A6: create a registration per child, once
  if tg_op = 'UPDATE' and new.status = 'registered' and old.status <> 'registered' then
    insert into public.registrations (child_id, group_id, class_type_id, school_year, status, source_lead_id)
    select lc.child_id, lc.group_id, lc.class_type_id,
           (select s.current_school_year from public.app_settings s where s.id = 1), 'active', new.id
    from public.lead_children lc where lc.lead_id = new.id
    on conflict (source_lead_id, child_id) where source_lead_id is not null do nothing;
  end if;
  return new;
end $$;
create trigger leads_after after insert or update on public.leads
  for each row execute function app.trg_leads_after();

-- ---------- registrations ----------
create function app.trg_registrations_after() returns trigger
language plpgsql security definer set search_path = '' as $$
declare v_first text; v_last text; v_family uuid; v_due date;
begin
  select c.first_name, f.last_name, f.id into v_first, v_last, v_family
  from public.children c join public.families f on f.id = c.family_id where c.id = new.child_id;

  -- A6 task: registration born from a lead without a group
  if tg_op = 'INSERT' and new.source_lead_id is not null and new.group_id is null then
    insert into public.tasks (title, assignee_id, urgency, due_date, source, registration_id, family_id)
    values ('לשבץ את ' || v_first || ' ' || v_last || ' לקבוצה', app.default_assignee(), 'medium',
            app.next_workday((now() at time zone 'Asia/Jerusalem')::date), 'A6', new.id, v_family);
  end if;
  if tg_op = 'UPDATE' and new.group_id is not null and old.group_id is null then
    update public.tasks set status = 'done'
    where source = 'A6' and registration_id = new.id and status = 'open';
  end if;

  -- A8
  if new.expected_payment_date is null or new.status in ('left','finished') then
    update public.tasks set status = 'cancelled'
    where source = 'A8' and registration_id = new.id and status = 'open';
  elsif tg_op = 'INSERT' or new.expected_payment_date is distinct from old.expected_payment_date then
    v_due := app.next_workday(new.expected_payment_date);
    update public.tasks set due_date = v_due
    where source = 'A8' and registration_id = new.id and status = 'open';
    if not found then
      insert into public.tasks (title, assignee_id, urgency, due_date, source, registration_id, family_id)
      values ('לבדוק אם הכסף הגיע – ' || v_first || ' ' || v_last, app.default_assignee(), 'medium',
              v_due, 'A8', new.id, v_family);
    end if;
  end if;
  return new;
end $$;
create trigger registrations_after after insert or update on public.registrations
  for each row execute function app.trg_registrations_after();
```

- [ ] **Step 4: Apply and run** – `npx supabase db push && npm run test:db` — Expected: PASS.

- [ ] **Step 5: Commit** `git add supabase tests && git commit -m "feat(db): automations A1, A6 and A8"`

---

## Task 7: Automation A7 and scheduled jobs

**Files:**
- Create: `supabase/migrations/2026111505_a7_and_cron.sql`, `tests/db/a7.test.ts`

**Interfaces:**
- Consumes: `leads.contract_sent_at` (set by `leads_before`), `app.default_assignee()`.
- Produces: `app.run_a7(p_force boolean default false) returns int` (number of tasks created); RPC `public.run_a7_now() returns int` (service role only, forces a run); cron jobs `a7-hourly`, `nightly-cleanup`. A7 title: `משפחת <last_name> עדיין לא חתמה`, urgency `high`.

- [ ] **Step 1: Write the failing test** – `tests/db/a7.test.ts`:

```ts
import { afterAll, beforeAll, expect, test } from "vitest";
import { admin, cleanup, makeFamily, makeUser, tag, type TestUser } from "./helpers";

const t = tag();
let manager: TestUser;
beforeAll(async () => { manager = await makeUser("manager"); });
afterAll(() => cleanup(t, [manager.id]));

const daysAgo = (n: number) => new Date(Date.now() - n * 86_400_000).toISOString();
const a7 = async (leadId: string) =>
  (await admin.from("tasks").select("*").eq("lead_id", leadId).eq("source", "A7")).data ?? [];

test("contract_sent_at is stamped on the transition", async () => {
  const f = await makeFamily(t);
  const lead = (await admin.from("leads").insert({ family_id: f.id }).select().single()).data!;
  expect(lead.contract_sent_at).toBeNull();
  const upd = (await admin.from("leads").update({ status: "contract_sent" }).eq("id", lead.id).select().single()).data!;
  expect(upd.contract_sent_at).not.toBeNull();
});

test("creates one high-urgency task after 3 days, never twice", async () => {
  const f = await makeFamily(t);
  const old = (await admin.from("leads").insert({ family_id: f.id, status: "contract_sent" }).select().single()).data!;
  const fresh = (await admin.from("leads").insert({ family_id: f.id, status: "contract_sent" }).select().single()).data!;
  const signed = (await admin.from("leads").insert({ family_id: f.id, status: "contract_sent", contract_signed: true }).select().single()).data!;
  await admin.from("leads").update({ contract_sent_at: daysAgo(4) }).in("id", [old.id, signed.id]);

  expect((await admin.rpc("run_a7_now")).error).toBeNull();
  const tasks = await a7(old.id);
  expect(tasks.length).toBe(1);
  expect(tasks[0].title).toBe(`משפחת ${f.last_name} עדיין לא חתמה`);
  expect(tasks[0].urgency).toBe("high");
  expect((await a7(fresh.id)).length).toBe(0);
  expect((await a7(signed.id)).length).toBe(0);

  await admin.from("tasks").update({ status: "done" }).eq("id", tasks[0].id);
  await admin.rpc("run_a7_now");
  expect((await a7(old.id)).length).toBe(1);
});

test("run_a7_now is not callable by users", async () => {
  const r = await manager.client.rpc("run_a7_now");
  expect(r.error).not.toBeNull();
});
```

- [ ] **Step 2: Run** `npm run test:db -- a7` — Expected: FAIL (function `run_a7_now` not found).

- [ ] **Step 3: Write the migration** – `supabase/migrations/2026111505_a7_and_cron.sql`:

```sql
create extension if not exists pg_cron;
create extension if not exists pg_net with schema extensions;

create function app.run_a7(p_force boolean default false) returns int
language plpgsql security definer set search_path = '' as $$
declare v_count int;
begin
  if not p_force and extract(hour from now() at time zone 'Asia/Jerusalem') <> 7 then
    return 0;
  end if;
  insert into public.tasks (title, assignee_id, urgency, due_date, source, lead_id, family_id)
  select 'משפחת ' || f.last_name || ' עדיין לא חתמה', app.default_assignee(), 'high',
         app.next_workday((now() at time zone 'Asia/Jerusalem')::date), 'A7', l.id, l.family_id
  from public.leads l join public.families f on f.id = l.family_id
  where l.status = 'contract_sent' and not l.contract_signed and l.archived_at is null
    and l.contract_sent_at <= now() - interval '3 days'
    and not exists (select 1 from public.tasks t where t.source = 'A7' and t.lead_id = l.id);
  get diagnostics v_count = row_count;
  return v_count;
end $$;

create function public.run_a7_now() returns int
language sql security definer set search_path = '' as $$ select app.run_a7(true) $$;
revoke all on function public.run_a7_now() from public, anon, authenticated;
grant execute on function public.run_a7_now() to service_role;

select cron.schedule('a7-hourly', '5 * * * *', $$ select app.run_a7(false) $$);

select cron.schedule('nightly-cleanup', '30 0 * * *', $$
  delete from public.verified_sessions where expires_at < now() - interval '1 day';
  delete from public.login_codes where expires_at < now() - interval '1 day';
  delete from public.audit_log where at < now() - interval '24 months';
$$);
```

- [ ] **Step 4: Apply and run** – `npx supabase db push && npm run test:db` — Expected: PASS.

- [ ] **Step 5: Commit** `git add supabase tests && git commit -m "feat(db): automation A7 and scheduled cleanup"`

---

## Task 8: Email outbox and the `send-emails` Edge Function

**Files:**
- Create: `supabase/migrations/2026111506_email_outbox.sql`, `supabase/functions/send-emails/index.ts`, `tests/db/outbox.test.ts`

**Interfaces:**
- Consumes: `tasks`, `profiles.email`, `app_settings.app_url`, Vault secrets `send_emails_url`, `cron_secret`.
- Produces: trigger `tasks_email` on `tasks`; RPC `public.claim_pending_emails(p_limit int) returns setof email_outbox` (service role only; each claim increments `attempts` and pushes `send_after` 5 minutes ahead); cron job `send-emails`; Edge Function `send-emails` (POST, header `Authorization: Bearer <CRON_SECRET>`). Email link format: `<app_url>/t/<task id>` (the `/t/[id]` redirect page is built in Plan 1B).

- [ ] **Step 1: Write the failing test** – `tests/db/outbox.test.ts`:

```ts
import { afterAll, beforeAll, expect, test } from "vitest";
import { admin, cleanup, makeUser, tag, type TestUser } from "./helpers";

const t = tag();
let secretary: TestUser, manager: TestUser;
beforeAll(async () => { [secretary, manager] = await Promise.all([makeUser("secretary"), makeUser("manager")]); });
afterAll(() => cleanup(t, [secretary.id, manager.id]));

const outbox = async (taskId: string) =>
  (await admin.from("email_outbox").select("*").eq("task_id", taskId)).data ?? [];
const newTask = async (due: string, title = `משימה <b> ${t}`) =>
  (await secretary.client.from("tasks").insert({ title, assignee_id: secretary.id, due_date: due }).select().single()).data!;

test("future task is queued for 07:30 Israel time on the due date", async () => {
  const task = await newTask("2027-01-12"); // winter: UTC+2
  const [row] = await outbox(task.id);
  expect(row.to_email).toBe(secretary.email);
  expect(new Date(row.send_after).toISOString()).toBe("2027-01-12T05:30:00.000Z");
  expect(row.subject).toContain("משימה");
  expect(row.body_html).toContain("&lt;b&gt;");
  expect(row.body_html).toContain(`/t/${task.id}`);
});

test("task due today or earlier is queued for now", async () => {
  const task = await newTask("2020-01-01");
  const [row] = await outbox(task.id);
  expect(new Date(row.send_after).getTime()).toBeLessThanOrEqual(Date.now() + 5_000);
});

test("date and assignee changes move the pending email; done cancels it", async () => {
  const task = await newTask("2027-01-12");
  await secretary.client.from("tasks").update({ due_date: "2027-01-13", assignee_id: manager.id }).eq("id", task.id);
  let rows = await outbox(task.id);
  expect(rows.length).toBe(1);
  expect(new Date(rows[0].send_after).toISOString()).toBe("2027-01-13T05:30:00.000Z");
  expect(rows[0].to_email).toBe(manager.email);
  await secretary.client.from("tasks").update({ status: "done" }).eq("id", task.id);
  rows = await outbox(task.id);
  expect(rows[0].status).toBe("cancelled");
});

test("claim_pending_emails returns due rows once and counts attempts", async () => {
  const task = await newTask("2020-01-01");
  const first = await admin.rpc("claim_pending_emails", { p_limit: 500 });
  expect(first.data.some((r: { task_id: string }) => r.task_id === task.id)).toBe(true);
  const second = await admin.rpc("claim_pending_emails", { p_limit: 500 });
  expect(second.data.some((r: { task_id: string }) => r.task_id === task.id)).toBe(false);
  const [row] = await outbox(task.id);
  expect(row.attempts).toBe(1);
  // tidy up so the real function never sends test mail
  await admin.from("email_outbox").update({ status: "cancelled" }).eq("task_id", task.id);
});

test("users cannot claim emails", async () => {
  expect((await manager.client.rpc("claim_pending_emails", { p_limit: 1 })).error).not.toBeNull();
});
```

Note: `claim_pending_emails` in the fourth test may also claim real pending rows in the dev project; that only delays them 5 minutes. Never run DB tests against `hamasmer-prod`. Test users have `@hamasmer.test` addresses, so even if the Edge Function picks one up, Resend rejects it and the row ends as `failed`.

- [ ] **Step 2: Run** `npm run test:db -- outbox` — Expected: FAIL (no outbox rows).

- [ ] **Step 3: Write the migration** – `supabase/migrations/2026111506_email_outbox.sql`:

```sql
create function app.html_escape(p text) returns text
language sql immutable as $$
  select replace(replace(replace(coalesce(p, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')
$$;

create function app.trg_tasks_email() returns trigger
language plpgsql security definer set search_path = '' as $$
declare v_email text; v_send timestamptz; v_subject text; v_body text; v_url text;
begin
  if new.status <> 'open' then
    update public.email_outbox set status = 'cancelled'
    where task_id = new.id and status = 'pending';
    return new;
  end if;
  if tg_op = 'UPDATE'
     and new.due_date = old.due_date and new.assignee_id = old.assignee_id
     and new.title = old.title and new.description is not distinct from old.description
     and old.status = 'open' then
    return new;
  end if;

  select p.email into v_email from public.profiles p where p.id = new.assignee_id;
  select s.app_url into v_url from public.app_settings s where s.id = 1;
  v_send := greatest(now(), (new.due_date + time '07:30') at time zone 'Asia/Jerusalem');
  v_subject := 'משימה: ' || new.title;
  v_body := '<div dir="rtl" style="font-family:Arial,sans-serif;font-size:16px">'
         || '<h2>' || app.html_escape(new.title) || '</h2>'
         || '<p>' || app.html_escape(new.description) || '</p>'
         || '<p>תאריך יעד: ' || to_char(new.due_date, 'DD/MM/YYYY') || '</p>'
         || '<p><a href="' || v_url || '/t/' || new.id || '">לפתיחת המשימה במערכת</a></p>'
         || '</div>';

  update public.email_outbox
  set to_email = v_email, subject = v_subject, body_html = v_body, send_after = v_send
  where task_id = new.id and status = 'pending';
  if not found and tg_op = 'INSERT' then
    insert into public.email_outbox (to_email, subject, body_html, task_id, send_after)
    values (v_email, v_subject, v_body, new.id, v_send);
  end if;
  return new;
end $$;
create trigger tasks_email after insert or update on public.tasks
  for each row execute function app.trg_tasks_email();

create function public.claim_pending_emails(p_limit int) returns setof public.email_outbox
language sql security definer set search_path = '' as $$
  update public.email_outbox o
  set attempts = o.attempts + 1, send_after = now() + interval '5 minutes'
  where o.id in (
    select id from public.email_outbox
    where status = 'pending' and send_after <= now()
    order by send_after limit p_limit for update skip locked)
  returning o.*
$$;
revoke all on function public.claim_pending_emails(int) from public, anon, authenticated;
grant execute on function public.claim_pending_emails(int) to service_role;

select cron.schedule('send-emails', '* * * * *', $$
  select net.http_post(
    url := (select decrypted_secret from vault.decrypted_secrets where name = 'send_emails_url'),
    headers := jsonb_build_object(
      'Content-Type', 'application/json',
      'Authorization', 'Bearer ' || (select decrypted_secret from vault.decrypted_secrets where name = 'cron_secret')),
    body := '{}'::jsonb)
  where exists (select 1 from vault.decrypted_secrets where name = 'send_emails_url');
$$);
```

- [ ] **Step 4: Apply and run** – `npx supabase db push && npm run test:db` — Expected: PASS.

- [ ] **Step 5: Edge Function** – `supabase/functions/send-emails/index.ts`:

```ts
import { createClient } from "jsr:@supabase/supabase-js@2";

const MAX_ATTEMPTS = 3;

Deno.serve(async (req) => {
  if (req.headers.get("Authorization") !== `Bearer ${Deno.env.get("CRON_SECRET")}`) {
    return new Response("forbidden", { status: 403 });
  }
  const db = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
  const { data: rows, error } = await db.rpc("claim_pending_emails", { p_limit: 20 });
  if (error) return new Response(error.message, { status: 500 });

  let sent = 0, failed = 0;
  for (const row of rows ?? []) {
    let problem: string | null = null;
    try {
      const res = await fetch("https://api.resend.com/emails", {
        method: "POST",
        headers: { Authorization: `Bearer ${Deno.env.get("RESEND_API_KEY")}`, "Content-Type": "application/json" },
        body: JSON.stringify({ from: Deno.env.get("EMAIL_FROM"), to: row.to_email, subject: row.subject, html: row.body_html }),
      });
      if (!res.ok) problem = `resend ${res.status}: ${(await res.text()).slice(0, 300)}`;
    } catch (e) {
      problem = String(e).slice(0, 300);
    }
    if (problem === null) {
      await db.from("email_outbox").update({ status: "sent", sent_at: new Date().toISOString(), last_error: null }).eq("id", row.id);
      sent++;
    } else {
      await db.from("email_outbox")
        .update({ last_error: problem, ...(row.attempts >= MAX_ATTEMPTS ? { status: "failed" } : {}) })
        .eq("id", row.id);
      failed++;
    }
  }
  return Response.json({ sent, failed });
});
```

- [ ] **Step 6: Deploy and wire secrets** (the user types secret values herself; the agent never echoes them)

```bash
npx supabase functions deploy send-emails --no-verify-jwt
```

User runs, replacing the placeholders with her own values (the cron secret is any long random string she generates, e.g. from her password manager):

```bash
npx supabase secrets set RESEND_API_KEY=<her key> CRON_SECRET=<random string> "EMAIL_FROM=Hamasmer CRM <onboarding@resend.dev>"
```

Then, in the Supabase dashboard → SQL Editor, the user runs (same random string; function URL is `https://<project-ref>.supabase.co/functions/v1/send-emails`):

```sql
select vault.create_secret('<random string>', 'cron_secret');
select vault.create_secret('https://<project-ref>.supabase.co/functions/v1/send-emails', 'send_emails_url');
```

- [ ] **Step 7: Verify** – the function rejects callers without the secret: `curl -s -o /dev/null -w "%{http_code}" -X POST https://<project-ref>.supabase.co/functions/v1/send-emails` → `403`. The full "task → inbox" check happens in Task 9 Step 11, once a real manager user exists.

- [ ] **Step 8: Commit** `git add supabase tests && git commit -m "feat: email outbox and send-emails edge function"`

---

## Task 9: Login with email code (app side)

**Files:**
- Create: `lib/supabase/server.ts`, `lib/supabase/client.ts`, `lib/supabase/admin.ts`, `lib/supabase/middleware.ts`, `middleware.ts`, `lib/auth/login-code.ts`, `lib/auth/session.ts`, `lib/email/resend.ts`, `lib/errors.ts`, `app/(auth)/login/page.tsx`, `app/(auth)/login/actions.ts`, `app/(auth)/verify/page.tsx`, `app/(auth)/verify/verify-form.tsx`, `app/api/auth/code/request/route.ts`, `app/api/auth/code/verify/route.ts`, `app/(app)/layout.tsx`, `app/(app)/page.tsx`, `scripts/create-manager.mjs`, `tests/unit/login-code.test.ts`, `tests/unit/errors.test.ts`, `.env.example`
- Delete: `app/page.tsx` (replaced by `app/(app)/page.tsx`)

**Interfaces:**
- Consumes: RPC `my_session()`, tables `login_codes`, `verified_sessions` (service role), env vars from Task 0.
- Produces: `createClient()` (server, async), `createBrowserClient()` (client), `createAdminClient()` (server only); `generateCode(): string`, `hashCode(code: string, sessionId: string): string`, `codesMatch(a: string, b: string): boolean`; `getSessionId(accessToken: string): string | null`; `sendEmail({to, subject, html}): Promise<void>`; `dbErrorMessage(message: string | undefined): string`; routes `POST /api/auth/code/request` → `200 {ok:true}` | `401` | `423 {error:"LOCKED"}` | `429 {error:"TOO_SOON"}`; `POST /api/auth/code/verify` body `{code}` → `200 {ok:true}` | `400 {error:"WRONG_CODE"|"EXPIRED"}` | `423 {error:"LOCKED"}`.

- [ ] **Step 1: Write failing unit tests**

`tests/unit/login-code.test.ts`:

```ts
import { expect, test } from "vitest";
import { codesMatch, generateCode, hashCode } from "@/lib/auth/login-code";
import { getSessionId } from "@/lib/auth/session";

test("code is 6 digits", () => {
  for (let i = 0; i < 200; i++) expect(generateCode()).toMatch(/^\d{6}$/);
});
test("hash depends on code and session", () => {
  const h = hashCode("123456", "s1");
  expect(h).toMatch(/^[0-9a-f]{64}$/);
  expect(hashCode("123456", "s1")).toBe(h);
  expect(hashCode("123456", "s2")).not.toBe(h);
  expect(hashCode("123457", "s1")).not.toBe(h);
});
test("codesMatch", () => {
  expect(codesMatch("abc", "abc")).toBe(true);
  expect(codesMatch("abc", "abd")).toBe(false);
  expect(codesMatch("abc", "ab")).toBe(false);
});
test("getSessionId reads the JWT payload", () => {
  const payload = Buffer.from(JSON.stringify({ session_id: "abc-123" })).toString("base64url");
  expect(getSessionId(`x.${payload}.y`)).toBe("abc-123");
  expect(getSessionId("garbage")).toBeNull();
});
```

`tests/unit/errors.test.ts`:

```ts
import { expect, test } from "vitest";
import { dbErrorMessage } from "@/lib/errors";

test("maps known tokens to Hebrew", () => {
  expect(dbErrorMessage("MOBILE_INVALID")).toBe("מספר הפלאפון לא תקין. צריך להיות 10 ספרות ולהתחיל ב-05.");
  expect(dbErrorMessage('new row violates ... "LEAD_NO_CHILDREN"')).toBe("אי אפשר לרשום פנייה בלי ילדים. הוסיפי ילד לפנייה.");
});
test("unknown errors get the generic message", () => {
  expect(dbErrorMessage("something odd")).toBe("משהו השתבש, נסי שוב.");
  expect(dbErrorMessage(undefined)).toBe("משהו השתבש, נסי שוב.");
});
```

- [ ] **Step 2: Run** `npm run test:unit` — Expected: FAIL (modules not found).

- [ ] **Step 3: Implement the pure modules**

`lib/auth/login-code.ts`:

```ts
import { createHash, randomInt, timingSafeEqual } from "node:crypto";

export const CODE_TTL_MS = 10 * 60_000;
export const RESEND_WAIT_MS = 60_000;
export const MAX_ATTEMPTS = 5;
export const LOCK_MS = 15 * 60_000;
export const VERIFIED_TTL_MS = 12 * 3_600_000;

export const generateCode = () => String(randomInt(0, 1_000_000)).padStart(6, "0");
export const hashCode = (code: string, sessionId: string) =>
  createHash("sha256").update(`${code}:${sessionId}`).digest("hex");
export function codesMatch(a: string, b: string): boolean {
  const x = Buffer.from(a), y = Buffer.from(b);
  return x.length === y.length && timingSafeEqual(x, y);
}
```

`lib/auth/session.ts`:

```ts
export function getSessionId(accessToken: string): string | null {
  try {
    const payload = JSON.parse(Buffer.from(accessToken.split(".")[1], "base64url").toString());
    return typeof payload.session_id === "string" ? payload.session_id : null;
  } catch {
    return null;
  }
}
```

`lib/errors.ts`:

```ts
const MESSAGES: Record<string, string> = {
  MOBILE_INVALID: "מספר הפלאפון לא תקין. צריך להיות 10 ספרות ולהתחיל ב-05.",
  PHONE_INVALID: "מספר הטלפון לא תקין.",
  LEAD_NO_CHILDREN: "אי אפשר לרשום פנייה בלי ילדים. הוסיפי ילד לפנייה.",
  LEAD_FLAGS_MISSING: "כדי לרשום צריך לסמן חוזה חתום, אמצעי תשלום וצינתוקים.",
  TASK_STATUS_ONLY: "אפשר לשנות רק את סטטוס המשימה.",
  RESTORE_MANAGER_ONLY: "רק המנהלת יכולה לשחזר מהארכיון.",
  NO_ASSIGNEE: "לא הוגדרה אחראית על משימות אוטומטיות.",
  children_id_number_uq: "תעודת הזהות הזו כבר קיימת אצל ילד אחר.",
  children_id_number_check: "מספר תעודת הזהות לא תקין.",
};
const GENERIC = "משהו השתבש, נסי שוב.";

export function dbErrorMessage(message: string | undefined): string {
  if (!message) return GENERIC;
  for (const [token, text] of Object.entries(MESSAGES)) if (message.includes(token)) return text;
  return GENERIC;
}
```

- [ ] **Step 4: Run** `npm run test:unit` — Expected: PASS.

- [ ] **Step 5: Supabase clients**

`lib/supabase/server.ts`:

```ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!, {
    cookies: {
      getAll: () => cookieStore.getAll(),
      setAll: (list) => {
        try { list.forEach(({ name, value, options }) => cookieStore.set(name, value, options)); }
        catch { /* called from a Server Component: the middleware refreshes cookies */ }
      },
    },
  });
}
```

`lib/supabase/client.ts`:

```ts
import { createBrowserClient as create } from "@supabase/ssr";
export const createBrowserClient = () =>
  create(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!);
```

`lib/supabase/admin.ts`:

```ts
import "server-only";
import { createClient } from "@supabase/supabase-js";
export const createAdminClient = () =>
  createClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!, {
    auth: { persistSession: false, autoRefreshToken: false },
  });
```

`lib/supabase/middleware.ts`:

```ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

const PUBLIC = ["/login", "/api/auth/"];

export async function updateSession(request: NextRequest) {
  let response = NextResponse.next({ request });
  const supabase = createServerClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!, {
    cookies: {
      getAll: () => request.cookies.getAll(),
      setAll: (list) => {
        list.forEach(({ name, value }) => request.cookies.set(name, value));
        response = NextResponse.next({ request });
        list.forEach(({ name, value, options }) => response.cookies.set(name, value, options));
      },
    },
  });
  const { data: { user } } = await supabase.auth.getUser();
  const path = request.nextUrl.pathname;
  const isPublic = PUBLIC.some((p) => path.startsWith(p));
  const go = (to: string) => {
    const url = request.nextUrl.clone();
    url.pathname = to;
    url.search = to === "/login" && !isPublic && path !== "/" ? `?next=${encodeURIComponent(path)}` : "";
    return NextResponse.redirect(url);
  };

  if (!user) return isPublic ? response : go("/login");
  if (path.startsWith("/api/auth/")) return response;

  const { data } = await supabase.rpc("my_session");
  const verified = data?.verified === true && data?.role != null;
  if (!verified) return path === "/verify" ? response : go("/verify");
  if (path === "/login" || path === "/verify") return go("/");
  return response;
}
```

`middleware.ts`:

```ts
import type { NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/middleware";

export const middleware = (request: NextRequest) => updateSession(request);
export const config = { matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|ico)$).*)"] };
```

- [ ] **Step 6: Email sender** – `lib/email/resend.ts`:

```ts
import "server-only";

export async function sendEmail({ to, subject, html }: { to: string; subject: string; html: string }) {
  const res = await fetch("https://api.resend.com/emails", {
    method: "POST",
    headers: { Authorization: `Bearer ${process.env.RESEND_API_KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({ from: process.env.EMAIL_FROM, to, subject, html }),
  });
  if (!res.ok) throw new Error(`resend ${res.status}`);
}
```

- [ ] **Step 7: Code routes**

`app/api/auth/code/request/route.ts`:

```ts
import { NextResponse } from "next/server";
import { CODE_TTL_MS, RESEND_WAIT_MS, generateCode, hashCode } from "@/lib/auth/login-code";
import { getSessionId } from "@/lib/auth/session";
import { sendEmail } from "@/lib/email/resend";
import { createAdminClient } from "@/lib/supabase/admin";
import { createClient } from "@/lib/supabase/server";

export async function POST() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  const { data: { session } } = await supabase.auth.getSession();
  const sessionId = session ? getSessionId(session.access_token) : null;
  if (!user || !user.email || !sessionId) return NextResponse.json({ error: "UNAUTHENTICATED" }, { status: 401 });

  const admin = createAdminClient();

  // A session that was verified once and expired must log in again (12-hour rule).
  const { data: old } = await admin.from("verified_sessions").select("expires_at").eq("session_id", sessionId).maybeSingle();
  if (old) {
    await supabase.auth.signOut();
    return NextResponse.json({ error: "SESSION_EXPIRED" }, { status: 401 });
  }

  const now = Date.now();
  const { data: row } = await admin.from("login_codes").select("*").eq("user_id", user.id).maybeSingle();
  if (row?.locked_until && new Date(row.locked_until).getTime() > now) {
    return NextResponse.json({ error: "LOCKED" }, { status: 423 });
  }
  if (row && now - new Date(row.created_at).getTime() < RESEND_WAIT_MS) {
    return NextResponse.json({ error: "TOO_SOON" }, { status: 429 });
  }
  const lockExpired = row?.locked_until != null;
  const code = generateCode();
  const { error } = await admin.from("login_codes").upsert({
    user_id: user.id,
    session_id: sessionId,
    code_hash: hashCode(code, sessionId),
    expires_at: new Date(now + CODE_TTL_MS).toISOString(),
    attempts: lockExpired ? 0 : row?.attempts ?? 0,
    locked_until: null,
    created_at: new Date(now).toISOString(),
  });
  if (error) return NextResponse.json({ error: "SERVER" }, { status: 500 });

  try {
    await sendEmail({
      to: user.email,
      subject: `קוד הכניסה שלך: ${code}`,
      html: `<div dir="rtl" style="font-family:Arial,sans-serif;font-size:16px">
        <p>קוד הכניסה למערכת המסמר:</p>
        <p style="font-size:32px;letter-spacing:6px"><b>${code}</b></p>
        <p>הקוד תקף ל-10 דקות. אם לא ניסית להיכנס, אפשר להתעלם מהמייל.</p></div>`,
    });
  } catch {
    return NextResponse.json({ error: "SERVER" }, { status: 500 });
  }
  return NextResponse.json({ ok: true });
}
```

`app/api/auth/code/verify/route.ts`:

```ts
import { NextResponse } from "next/server";
import { z } from "zod";
import { LOCK_MS, MAX_ATTEMPTS, VERIFIED_TTL_MS, codesMatch, hashCode } from "@/lib/auth/login-code";
import { getSessionId } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";
import { createClient } from "@/lib/supabase/server";

const Body = z.object({ code: z.string().regex(/^\d{6}$/) });

export async function POST(request: Request) {
  const parsed = Body.safeParse(await request.json().catch(() => null));
  if (!parsed.success) return NextResponse.json({ error: "WRONG_CODE" }, { status: 400 });

  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  const { data: { session } } = await supabase.auth.getSession();
  const sessionId = session ? getSessionId(session.access_token) : null;
  if (!user || !sessionId) return NextResponse.json({ error: "UNAUTHENTICATED" }, { status: 401 });

  const admin = createAdminClient();
  const now = Date.now();
  const { data: row } = await admin.from("login_codes").select("*").eq("user_id", user.id).maybeSingle();
  if (!row) return NextResponse.json({ error: "EXPIRED" }, { status: 400 });
  if (row.locked_until && new Date(row.locked_until).getTime() > now) {
    return NextResponse.json({ error: "LOCKED" }, { status: 423 });
  }
  if (new Date(row.expires_at).getTime() < now || row.session_id !== sessionId) {
    return NextResponse.json({ error: "EXPIRED" }, { status: 400 });
  }

  if (!codesMatch(row.code_hash, hashCode(parsed.data.code, sessionId))) {
    const attempts = row.attempts + 1;
    const locked = attempts >= MAX_ATTEMPTS;
    await admin.from("login_codes").update({
      attempts,
      locked_until: locked ? new Date(now + LOCK_MS).toISOString() : null,
    }).eq("user_id", user.id);
    return NextResponse.json({ error: locked ? "LOCKED" : "WRONG_CODE" }, { status: locked ? 423 : 400 });
  }

  const { error } = await admin.from("verified_sessions").insert({
    session_id: sessionId, user_id: user.id, expires_at: new Date(now + VERIFIED_TTL_MS).toISOString(),
  });
  if (error) return NextResponse.json({ error: "SERVER" }, { status: 500 });
  await admin.from("login_codes").delete().eq("user_id", user.id);
  return NextResponse.json({ ok: true });
}
```

- [ ] **Step 8: Pages**

`app/(auth)/login/actions.ts`:

```ts
"use server";
import { redirect } from "next/navigation";
import { getSessionId } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";
import { createClient } from "@/lib/supabase/server";

export async function signIn(_prev: { error: string } | null, formData: FormData) {
  const supabase = await createClient();
  const { error } = await supabase.auth.signInWithPassword({
    email: String(formData.get("email") ?? "").trim(),
    password: String(formData.get("password") ?? ""),
  });
  if (error) return { error: "המייל או הסיסמה לא נכונים." };
  redirect("/verify");
}

export async function signOut() {
  const supabase = await createClient();
  const { data: { session } } = await supabase.auth.getSession();
  const sessionId = session ? getSessionId(session.access_token) : null;
  if (sessionId) await createAdminClient().from("verified_sessions").delete().eq("session_id", sessionId);
  await supabase.auth.signOut();
  redirect("/login");
}
```

`app/(auth)/login/page.tsx`:

```tsx
"use client";
import { useActionState } from "react";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { signIn } from "./actions";

export default function LoginPage() {
  const [state, action, pending] = useActionState(signIn, null);
  return (
    <main className="flex min-h-screen items-center justify-center p-4">
      <Card className="w-full max-w-sm">
        <CardHeader><CardTitle>כניסה למערכת המסמר</CardTitle></CardHeader>
        <CardContent>
          <form action={action} className="space-y-4">
            <div className="space-y-2">
              <Label htmlFor="email">מייל</Label>
              <Input id="email" name="email" type="email" dir="ltr" autoComplete="username" required />
            </div>
            <div className="space-y-2">
              <Label htmlFor="password">סיסמה</Label>
              <Input id="password" name="password" type="password" dir="ltr" autoComplete="current-password" required />
            </div>
            {state?.error && <p role="alert" className="text-sm text-red-600">{state.error}</p>}
            <Button type="submit" className="w-full" disabled={pending}>{pending ? "נכנסת…" : "כניסה"}</Button>
          </form>
        </CardContent>
      </Card>
    </main>
  );
}
```

`app/(auth)/verify/verify-form.tsx`:

```tsx
"use client";
import { useEffect, useRef, useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

const TEXT: Record<string, string> = {
  WRONG_CODE: "הקוד לא נכון. נסי שוב.",
  EXPIRED: "הקוד פג תוקף. בקשי קוד חדש.",
  LOCKED: "יותר מדי ניסיונות. אפשר לנסות שוב בעוד 15 דקות.",
  TOO_SOON: "קוד כבר נשלח. אפשר לבקש קוד חדש בעוד דקה.",
  SERVER: "משהו השתבש, נסי שוב.",
};

export function VerifyForm() {
  const router = useRouter();
  const [message, setMessage] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);
  const asked = useRef(false);

  async function requestCode() {
    const res = await fetch("/api/auth/code/request", { method: "POST" });
    if (res.status === 401) return router.replace("/login");
    if (res.ok) return setMessage("שלחנו קוד למייל שלך.");
    const { error } = await res.json();
    setMessage(TEXT[error] ?? TEXT.SERVER);
  }
  useEffect(() => {
    if (asked.current) return;
    asked.current = true;
    void requestCode();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  async function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setBusy(true);
    const code = String(new FormData(e.currentTarget).get("code") ?? "").trim();
    const res = await fetch("/api/auth/code/verify", {
      method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ code }),
    });
    setBusy(false);
    if (res.ok) { router.replace("/"); router.refresh(); return; }
    if (res.status === 401) return router.replace("/login");
    const { error } = await res.json();
    setMessage(TEXT[error] ?? TEXT.SERVER);
  }

  return (
    <form onSubmit={onSubmit} className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="code">קוד בן 6 ספרות</Label>
        <Input id="code" name="code" dir="ltr" inputMode="numeric" pattern="\d{6}" maxLength={6}
               autoComplete="one-time-code" autoFocus required className="text-center text-2xl tracking-widest" />
      </div>
      {message && <p role="status" className="text-sm">{message}</p>}
      <Button type="submit" className="w-full" disabled={busy}>{busy ? "בודקת…" : "אישור"}</Button>
      <Button type="button" variant="ghost" className="w-full" onClick={requestCode}>שלחו לי קוד חדש</Button>
    </form>
  );
}
```

`app/(auth)/verify/page.tsx`:

```tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { VerifyForm } from "./verify-form";

export default function VerifyPage() {
  return (
    <main className="flex min-h-screen items-center justify-center p-4">
      <Card className="w-full max-w-sm">
        <CardHeader><CardTitle>אימות בקוד מהמייל</CardTitle></CardHeader>
        <CardContent><VerifyForm /></CardContent>
      </Card>
    </main>
  );
}
```

`app/(app)/layout.tsx`:

```tsx
import { Button } from "@/components/ui/button";
import { createClient } from "@/lib/supabase/server";
import { signOut } from "../(auth)/login/actions";

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  const { data: profile } = await supabase.from("profiles").select("full_name").eq("id", user!.id).maybeSingle();
  return (
    <div className="min-h-screen">
      <header className="flex items-center justify-between border-b px-4 py-3">
        <span className="font-bold">המסמר</span>
        <div className="flex items-center gap-3">
          <span className="text-sm">{profile?.full_name}</span>
          <form action={signOut}><Button variant="outline" size="sm" type="submit">יציאה</Button></form>
        </div>
      </header>
      <main className="p-4">{children}</main>
    </div>
  );
}
```

`app/(app)/page.tsx` (and delete `app/page.tsx`):

```tsx
export default function HomePage() {
  return <h1 className="text-2xl font-bold">היום שלי</h1>;
}
```

`.env.example`: the five variable names from Task 0, with empty values.

- [ ] **Step 9: Bootstrap script** – `scripts/create-manager.mjs` (the user runs it herself and types her own details in the terminal; nothing is passed as an argument, stored or printed):

```js
import { createClient } from "@supabase/supabase-js";
import { config } from "dotenv";
import { createInterface } from "node:readline/promises";

config({ path: ".env.local" });
const rl = createInterface({ input: process.stdin, output: process.stdout });
const email = (await rl.question("Manager email: ")).trim();
const fullName = (await rl.question("Full name (Hebrew is fine): ")).trim();
const password = await rl.question("Password (10+ characters, will be visible while typing): ");
rl.close();
if (password.length < 10) { console.error("Password too short."); process.exit(1); }

const admin = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY, {
  auth: { persistSession: false },
});
const { data, error } = await admin.auth.admin.createUser({ email, password, email_confirm: true });
if (error) { console.error(error.message); process.exit(1); }
const p = await admin.from("profiles").insert({ id: data.user.id, full_name: fullName, email, role: "manager" });
if (p.error) { console.error(p.error.message); process.exit(1); }
console.log("Manager created.");
```

- [ ] **Step 10: Supabase Auth settings** (user, in the dashboard → Authentication): turn **off** "Allow new users to sign up"; set minimum password length to **10**. Custom SMTP for invite mails is configured in Plan 1B together with the invite screen.

- [ ] **Step 11: Verify by hand**

Run: `npm run lint && npm run typecheck && npm run test:unit && npm run build`, then the user runs `node scripts/create-manager.mjs` in her terminal (using the address she signed up to Resend with, so mail can reach her), then `npm run dev`.
Expected, in the browser pane at `http://localhost:3000`:
1. `/` redirects to `/login`.
2. Wrong password shows "המייל או הסיסמה לא נכונים."
3. Correct password goes to `/verify`; a code arrives by email; a wrong code shows "הקוד לא נכון. נסי שוב."
4. The correct code opens "היום שלי" with her name in the header.
5. "יציאה" returns to `/login`, and `/` redirects to `/login` again.
6. Outbox end to end: in the Supabase table editor insert a `tasks` row assigned to her, due today. Within 2 minutes her inbox receives "משימה: …" and the `email_outbox` row shows `status = sent`.

- [ ] **Step 12: Commit** `git add -A -- . ':!*.docx' && git commit -m "feat: password + email-code login with 12-hour verified sessions"`

---

## Task 10: GitHub, Vercel deploy, hand-over

**Files:**
- Create: `README.md` (replace the scaffold README), `supabase/README.md`

**Interfaces:**
- Produces: a Vercel production URL; `app_settings.app_url` set to it.

- [ ] **Step 1: README** – replace `README.md` with: what the project is (one paragraph, Hebrew), how to run (`npm install`, `.env.local` variable names, `npm run dev`), how to test (`npm run test:unit`, `npm run test:db` — dev project only, never prod), how to apply migrations (`npx supabase db push`), and links to the spec and both plan files. `supabase/README.md`: the list of Vault secrets and function secrets by **name only**, and the cron jobs (`a7-hourly`, `nightly-cleanup`, `send-emails`).

- [ ] **Step 2: Push** – `git remote add origin <repo URL from Task 0> && git push -u origin main`. Confirm the `ci` workflow is green on GitHub.

- [ ] **Step 3: Vercel** (user, in the Vercel dashboard): Add New → Project → import `hamasmer-crm`; add the five environment variables from Task 0 (she pastes the values); Deploy. Region setting: Functions → `fra1` (Frankfurt), next to the database.

- [ ] **Step 4: Point emails at the deployed app** – user runs in the Supabase SQL editor: `update public.app_settings set app_url = 'https://<her-vercel-url>' where id = 1;` Then sets the same URL in Supabase → Authentication → URL Configuration → Site URL.

- [ ] **Step 5: Verify** – repeat Task 9 Step 11 items 1–5 on the Vercel URL. Run `npm run test:db` once more — Expected: all PASS.

- [ ] **Step 6: Commit and push** `git add README.md supabase/README.md && git commit -m "docs: project and Supabase readme" && git push`

---

## Self-review notes

- **Spec coverage:** §3 → Task 3; §4, §13 → Task 4; §5, §14 → Tasks 4, 9; §6, §15 → Tasks 6, 7; §7 → Task 8; audit (§3, §15) → Task 5; §17 error mapping → Task 9 (`lib/errors.ts`); §18 unit + DB tests → Tasks 2–9; §19 headers/noindex/secrets → Tasks 1, 8, 9. Deferred to Plan 1B: §8 screens, invite flow and custom SMTP, `set-password`, cities seed, failed-email banner, `/t/[id]` redirect, Playwright E2E, DB tests in CI.
- **Open product question** (spec §10.1, tzintuk consent): does not block this plan; a yes/no column can be added in Plan 1B.
