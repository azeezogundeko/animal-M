# Bison — Project Overview

Bison is a sports viewing centre management and fan subscription platform built for the Nigerian market. It connects fans to local viewing centres for live sports events, with NFC-based entry validation, subscription plans, and a digital wallet system.

## Roles

- **Fan** — subscribes to centres, uses NFC/HCE tap for entry, manages wallet
- **Owner** — manages their viewing centre(s), events, subscription plans, entry logs
- **Admin** — platform-level management, oversight, analytics

## Project Structure

```
becon/
├── backend/              Express + TypeScript REST API (production)
├── admin dashboard/      Web admin panel (not started)
├── landing page/         Marketing site (not started)
├── mobile/               Flutter app for fans/owners/readers (not started)
├── owner dashboard web/  Web dashboard for venue owners (not started)
├── ui code mobile/       Bison Emerald design system + exported component code
└── ui inspiration/       Reference images and mockups
```

## Tech Decisions

- **Backend:** Express 5 + TypeScript + Supabase (PostgreSQL)
- **Auth:** JWT (access 15min / refresh 30d) + phone-based OTP
- **Mobile:** Flutter (planned)
- **Design:** Bison Emerald — dark mode, emerald accent (#68dba9), "Organic Editorialism" style
- **No SQL ORMs** — raw queries via Supabase client through a base DatabaseManager class

## Key Docs

- `backend/docs/backend architecture.md` — full API reference and DB schema
- `backend/docs/prd.md` — product requirements
- `ui code mobile/stitch_bison_app_prd_design_system_specifications/bison_emerald/DESIGN.md` — design system

## Conventions

- All API responses follow a standard envelope: `{ success, message, data }`
- Errors use custom error classes from `backend/src/core/utils/errors.ts`
- RBAC is enforced via `authorize(...roles)` middleware
- Feature modules live in `backend/src/apps/<module>/` with controllers, services, models, routes, validators
