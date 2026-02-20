# VenuePro (n8n + Postgres)

VenuePro is a booking CRM built on **n8n workflows** with a **PostgreSQL** backend. You manage and operate the CRM through the **n8n web UI** (browser-based HTML interface): create/edit workflows, run automations, and view execution history—while data is stored in Postgres.

## Architecture

- **n8n**: Workflow engine + web UI (the primary “app interface”)
- **PostgreSQL**: Persistent CRM data (clients, leads, bookings, notes, etc.)
- *(Optional)* SMTP/Email, Google Calendar, WhatsApp/SMS providers, Stripe, etc.

## Requirements

- Docker + Docker Compose (recommended), or
- Self-hosted n8n and a reachable PostgreSQL instance

## Quick Start (Docker Compose)

1. Clone:
   ```bash
   git clone https://github.com/AndyJay72/VenuePro.git
   cd VenuePro
   ```

2. Create an `.env` file:
   - Copy `.env.example` to `.env` (if you have one)
   - At minimum you will need values like:
     - `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
     - `N8N_BASIC_AUTH_USER`, `N8N_BASIC_AUTH_PASSWORD` (recommended)
     - `N8N_HOST`, `N8N_PROTOCOL`, `N8N_PORT` (optional)

3. Start services:
   ```bash
   docker compose up -d
   ```

4. Open the n8n UI:
   - `http://localhost:5678`

## Using the HTML UI (n8n Web Interface)

In the n8n UI you can:
- Import provided VenuePro workflows (JSON)
- Configure credentials (Postgres, email, calendar, etc.)
- Run workflows manually or via triggers (Webhook / Schedule)
- View workflow executions and logs

### Importing Workflows
If this repo includes workflow exports:
1. Go to **n8n → Workflows**
2. Click **Import from File**
3. Select the `.json` workflow export(s)
4. Open each workflow and update:
   - Postgres credentials
   - Any environment variables
   - Webhook URLs (if used)

## Database (PostgreSQL)

VenuePro stores CRM entities in Postgres. Typical tables may include:
- `contacts`
- `leads`
- `venues` (optional)
- `bookings`
- `booking_notes`
- `activities` / `tasks`

### Connecting n8n to Postgres
1. In n8n, go to **Credentials → New**
2. Choose **Postgres**
3. Fill in host/user/password/db (from `.env` / compose)
4. Use **Postgres nodes** in workflows for:
   - `INSERT` new leads/inquiries
   - `UPDATE` booking stages
   - `SELECT` upcoming tours / holds
   - Reporting queries

## Recommended n8n Workflow Modules (Example)

- **Inbound inquiry capture**
  - Trigger: Webhook
  - Validate fields (name, email, event date, guest count)
  - Insert lead into Postgres
  - Notify staff (email/Slack)

- **Booking pipeline**
  - Trigger: Manual / Schedule
  - Find leads without follow-up
  - Create tasks / send reminders
  - Update stage fields in Postgres

- **Calendar sync**
  - Trigger: On booking confirmed
  - Create/update Google Calendar event
  - Store calendar event ID back in Postgres

## Security Notes

- Enable **Basic Auth** for n8n (or put it behind a reverse proxy with auth)
- Use strong passwords and keep `.env` out of git
- If exposing webhooks publicly, use:
  - Signed secrets / tokens
  - IP allowlists (where possible)
  - HTTPS only

## What I need from you to tailor this correctly

Reply with:
1. Are you running **n8n via Docker** or **n8n Cloud**?
2. Which UI port are you using (default `5678`)?
3. Do you already have:
   - `docker-compose.yml`?
   - workflow JSON exports (where are they in the repo)?
   - a database schema file (SQL) or do workflows create tables?

If you paste your `docker-compose.yml` (and any `schema.sql`), I can rewrite the README to match your exact setup and add precise env vars and import steps.
