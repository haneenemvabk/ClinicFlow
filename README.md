# ClinicFlow

A reusable, production-ready clinic website and clinic management system.
Built as a white-label product: one codebase, customizable for any clinic
without touching source code.

**Stack:** React 18 + TypeScript + Vite + Tailwind CSS (frontend) · Supabase
(PostgreSQL database, auth, row-level security, server-side booking
functions) (backend).

---

## What it does

### Public website
- Home page with hero, services overview, doctors preview, how-booking-works,
  testimonials (demo content), FAQ preview, CTAs
- About page (story, mission, values, facilities)
- Doctors listing (searchable/filterable) and full doctor profile pages
- Services listing (category filter) and service detail pages
- **Real appointment booking** — 5-step wizard with live availability,
  server-side validation, confirmation codes
- Contact form (persisted, rate-limited)
- FAQ page, Privacy Policy, Terms of Service, 404 page
- SEO: per-page titles/descriptions, canonical URLs, Open Graph,
  structured data (`MedicalClinic`), `robots.txt`
- Fully responsive (320px → 1440px+), accessible (semantic HTML, labels,
  focus states, ARIA, reduced-motion support)

### Admin dashboard (`/admin`)
- Login/logout (Supabase email + password auth)
- Dashboard: today's schedule, upcoming, stats, recent messages
- Appointment management: filters (date/doctor/service/status/patient
  search), detail drawer, status changes, internal notes
- Doctor management: create/edit/deactivate/reactivate/delete, with
  confirmation dialogs and validation
- Service management: create/edit/activate/deactivate, doctor-service
  links, price configuration
- Availability management: weekly working hours per doctor (with breaks),
  blocked dates/holidays (whole clinic or per doctor)
- Message management: read/unread/archive/delete, reply by email
- Clinic settings: name, branding colours, logo initials, contact details,
  opening hours, booking rules, social links, SEO description — the entire
  public site rebrands from this one page

---

## Architecture

```
src/
  components/       PublicLayout (header/footer), AdminLayout (sidebar+login),
                    shared UI primitives (Spinner, ErrorState, Badge…)
  contexts/         SettingsContext (clinic identity), AuthContext (admin session)
  lib/              api.ts (all Supabase queries), supabase.ts (client),
                    types.ts, format.ts (date/time helpers), seo.ts, clientKey.ts
  pages/            Public pages
  pages/admin/      Admin dashboard pages
supabase/
  migrations/       SQL schema + policies applied to the database
```

**Key design decisions**

- **All clinic data lives in the database.** Nothing about the clinic
  (name, colours, doctors, services, hours) is hard-coded in the UI.
- **Booking is server-side validated.** The `book_appointment` Postgres
  function validates everything (see below) — the wizard's client-side
  checks are only for UX, not security.
- **Double-booking is impossible.** A unique index *and* an EXCLUDE
  constraint (`btree_gist`) reject overlapping non-cancelled appointments
  for the same doctor, even under concurrent submissions.
- **Patient data is private.** `appointments` and `messages` have no public
  read policy. Patients book via SECURITY DEFINER functions; admins see
  data through RLS policies keyed on `admin_users`.
- **Server-side validation in `book_appointment`:** required fields, email
  and phone formats, valid date (not past, within advance window), minimum
  notice, working hours, breaks, blocked dates, slot alignment to the
  doctor's grid, doctor/service relationship, and overlap with existing
  bookings.
- **Rate limiting.** Booking and contact submissions are limited to 5 per
  hour per anonymous browser key, backed by a database table (works across
  serverless invocations).

---

## Local setup

```bash
npm install          # install dependencies
npm run dev          # start dev server
npm run build        # production build (output in dist/)
npm run typecheck    # TypeScript check
npm run lint         # ESLint
```

### Environment variables

`.env` (already provisioned in this project):

| Variable | Purpose |
|---|---|
| `VITE_SUPABASE_URL` | Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | Public anon key (safe for the browser; access is controlled by RLS) |

No secrets are committed or shipped to the browser. The service role key
never appears in frontend code.

### Database setup

The database schema, RLS policies, booking functions and demo data have
already been applied (see `supabase/migrations/` for the exact SQL). For a
fresh Supabase project, apply the migration files in filename order via the
Supabase dashboard (SQL editor) or MCP tooling.

Migrations:

1. `clinicflow_01_admin_and_settings` — helpers, `admin_users`, `clinic_settings`
2. `clinicflow_02_doctors_services` — `doctors`, `services`, `doctor_services`
3. `clinicflow_03_availability_appointments` — `doctor_availability`, `blocked_dates`, `appointments`, `messages`, `faqs` (+ anti-double-booking unique index)
4. `clinicflow_04_overlap_constraint` — EXCLUDE constraint for overlapping bookings
5. `clinicflow_05_booking_engine` — `get_available_slots`, `book_appointment`, `send_contact_message`, `rate_limit_check`, `rate_limit_hits`
6–7. Seed data (clinic settings, demo doctors/services/FAQs/appointments/messages)

> Note: the demo admin account is created through the Supabase signup API
> (bcrypt-hashed password), then linked in `admin_users`. SQL-inserted auth
> users are not supported by GoTrue sign-in.

### Demo admin account

| | |
|---|---|
| Email | `admin@clinicflow.demo` |
| Password | `DemoAdmin!2024` |

**Change this password before any real deployment** (Supabase dashboard →
Authentication → Users). The demo clinic, doctors, patients and messages
are all fictional sample data.

---

## Customizing for a new clinic (no code changes)

1. Sign in to `/admin` with an admin account.
2. **Settings page** — set clinic name, tagline, logo initials, brand
   colours, address, phone, email, emergency line, about text, opening
   hours, currency, booking rules (min notice, advance window), social
   links, SEO description. The entire public site updates from these
   values, including colours (CSS variables are set from the database row).
3. **Doctors page** — add the clinic's real doctors (name, title, specialty,
   biography, photo URL, qualifications, languages, slot length).
4. **Services page** — add services with duration, price, category, and
   link each service to the doctors who offer it.
5. **Availability page** — set each doctor's weekly hours and breaks, plus
   holidays/blocked dates (whole-clinic or per-doctor).
6. Public pages, booking wizard and dashboard immediately reflect the new
   data.

For a brand-new deployment (new database), apply the migrations, sign up
the first admin via the API, then follow steps 2–5 above.

### Deploying

1. `npm run build` — produces the static site in `dist/`.
2. Host `dist/` on any static host (Vercel, Netlify, Cloudflare Pages,
   S3…). Configure the host to serve `index.html` for unknown paths (SPA
   fallback).
3. Set `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` in the host's
   environment variables at build time.
4. Turn on email confirmation requirements / SMTP in Supabase Auth if
   desired. The demo admin's password must be changed.
5. Update `robots.txt` / sitemap with the production domain.

---

## Security notes

- Row-level security is enabled on **every** table; policies are
  deny-by-default.
- Public (anon) access is limited to active doctors, services,
  availability, FAQs, and the settings row. Appointments and messages are
  invisible to anon.
- All writes to operational tables require an authenticated clinic admin
  (`is_clinic_admin()` helper, SECURITY DEFINER).
- Booking/contact go through validated SECURITY DEFINER functions with
  fixed `search_path` (resists search-path hijacking), parameterized
  inputs (no SQL injection surface), and database-enforced constraints.
- Passwords are bcrypt-hashed by Supabase Auth; no passwords exist in
  application code.
- Rate limiting on booking and contact endpoints (5/hour/browser key).
- The frontend renders no raw HTML from user input (React escaping only),
  so stored XSS is not present in the UI.

## Troubleshooting

- **Booking wizard shows no slots** — check the doctor has availability for
  that weekday (Admin → Availability) and the date is within the advance
  window (Admin → Settings).
- **"This time slot has just been taken"** — expected when two people race
  for one slot; the loser is told to pick another.
- **Admin login fails** — verify the account exists in Supabase →
  Authentication → Users and is linked in `admin_users`.
- **Public pages look empty** — RLS: only *active* doctors/services are
  public. Deactivated items are visible in the admin dashboard only.
