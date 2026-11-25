# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Current state

There is no code yet. The repo only holds the planning document `Propuesta de Trabajo — Kiosko RH (1).md` (in Spanish), which is the source of truth for scope, constraints and decisions. Read the relevant section before building anything. No stack has been chosen, so there are no build, lint or test commands yet. **Update this file once a stack and commands exist.**

The client, users and UI text are Mexican Spanish. Write user-facing copy in Spanish.

## What this is

A self-service HR kiosk (touchscreen terminals in a manufacturing plant) plus an admin web panel, served from a **single local server with no internet access**. The first deliverable is due in 2 weeks. Whether that means a navigable prototype (Scenario A) or a working MVP (Scenario B) is still undecided. See "Plan de trabajo".

## Hard constraints that shape every technical decision

- **Offline / intranet only.** No cloud APIs (including AI), no CDNs, no external email/SMS, no managed services. Bundle every asset locally. The DB, scheduler and backups all run on the same box.
- **The kiosk is never the source of truth for payroll, attendance or headcount.** That data is a read-only mirror imported from Excel files and the external A3 (maybe also E3) system. Only data the kiosk creates itself lives natively in its DB: vacation requests, tickets, announcements, audit log.
- **Generate PDFs (cartas laborales, recibos de nómina) on demand. Never store them** on the server or cache them on the kiosk. Store only the underlying data.
- **~50 GB of disk.** The audit log and backups are the parts that grow.

## Roles (strict separation)

- **Admin TI**: users/roles, catalogs and field definitions, employee CRUD, integrations and sync monitoring, audit log, system parameters. Must **not** see salary data.
- **Admin RH**: vacation approval inbox, tickets, reports, announcements, employee lookup, letters. Must **not** change system structure.
- **Operador**: self-service over their **own** data only.

## Architecture decisions already made

- **Auth:** PIN or employee number with a numeric on-screen keypad, forced PIN change on first login. Never use NSS (social security number) as the password. Aggressive session timeout with a visible countdown. Biometrics are out of scope; PIN + explicit on-screen confirmation + an audit entry acts as the "signature".
- **Data import pipeline** (Excel/A3 files): watched folder or scheduled pickup → raw staging tables → validation (expected columns, types, duplicates, control totals) → idempotent upsert into final tables → sync log visible in admin. If validation fails, publish nothing (no partial data). MVP runs the import manually.
- **Sensitive changes** (for example a bank account) become requests in an RH validation queue, never direct edits.
- **Stale data:** show "sin conexión" instead of old balances or data.
- **Configurable fields** (Phase 2, possible stretch goal): a field-definition table (key, label, type, required, active, order, options) plus a JSON column on the employee record, with forms and validation rendered from the definitions. A drag-and-drop page builder is explicitly out of scope.
- **"Chatbot"** = a configurable decision tree that ends by creating a classified ticket. No AI involved.
- **Deployment:** Docker Compose if IT allows it, otherwise a native install + reverse proxy (nginx/IIS) + process manager. The kiosk runs a fullscreen kiosk-mode browser.
- **Stack candidates:** Laravel + Filament, Django + Django Admin, Next.js + Postgres, ASP.NET Core + SQL Server. Rule: the framework the developer already knows wins. Laravel/Filament or Django are favored because the admin panel is a large share of the scope.

## MVP scope

In: PIN login + 3 roles; operator home (photo, name, ≤6 large cards); my data; quick links; announcements (image, title, validity dates); vacation request with calendar + informational balance + RH approve/reject; tickets (create/list/comment/close, admin-editable categories); on-demand letter and payslip PDFs; basic admin dashboard; manual Excel import with validation and log; audit of logins and sensitive actions.

Explicitly out (Phase 2/3; do not add without a scope change): biometrics, live A3/E3 integration, export to A3 format, multi-level approvals and complex seniority rules, chatbot/decision tree, AI, no-code page builder, mobile access, email/SMS notifications, digital signatures, read receipts.

## Kiosk UI rules (operator screens)

Touch-first: targets of at least 48×48 px (prefer larger), body text of at least 18 px, titles of at least 24 px, high contrast. At most 5–6 options per screen, no hamburger menus or nested submenus, one path per task. Home and Salir (logout) are always visible. Use plain plant language ("Pedir vacaciones", not "Gestión de solicitudes de ausencia"). Ask for explicit confirmation before any submit, then show an immediate acknowledgment with a folio number. The admin panel is a separate desktop-oriented design.

## Edge cases to handle in design

Employee without a photo; duplicate names; terminated employee still present in the Excel; previous period's Excel uploaded by mistake; Excel format changed silently; a session left open on a shared terminal; network loss; incorrect data flowing into a legal letter. The demo must run without depending on the network.
