# WC2026 Daily Match Alerts — WhatsApp

> Automated **WhatsApp** notifications for international football matches involving **FIFA World Cup 2026** nations — built with **n8n**, **Twilio** and the **API-Football** REST API, deployed on **Railway**.

[![n8n](https://img.shields.io/badge/built%20with-n8n-EA4B71?logo=n8n&logoColor=white)](https://n8n.io)
[![Twilio](https://img.shields.io/badge/WhatsApp-Twilio-F22F46?logo=twilio&logoColor=white)](https://www.twilio.com/whatsapp)
[![Deployed on Railway](https://img.shields.io/badge/deployed-Railway-0B0D0E?logo=railway&logoColor=white)](https://railway.app)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

*¿Prefieres español?* → [README.es.md](README.es.md)

[Overview](#overview) • [How it works](#how-it-works) • [Stack](#stack) • [Prerequisites](#prerequisites) • [Getting started](#getting-started) • [Deployment](#deployment-localhost--railway)

## Overview

Every day this workflow sends two automated WhatsApp digests to a list of recipients:

| When (CR time) | What it sends |
| :--- | :--- |
| **07:00 — Morning** | Preview of **today's matches** with kickoff times (local) |
| **23:00 — Night** | **Final results** with goal scorers, minutes and penalty/own-goal tags |

Both digests are filtered to only the matches that matter: national teams that are **World Cup 2026 qualifiers**, in the **World Cup** and **International Friendlies** competitions — youth squads excluded.

> **Use case:** a group of friends and family in Costa Rica who want a clean, spam-free daily summary of World Cup-relevant football, delivered straight to WhatsApp — no app, no feed, no noise.

## How it works

```mermaid
flowchart LR
    subgraph Morning["Morning Pipeline — 07:00 CR"]
        A1["Schedule Trigger\ncron 0 7 * * *"] --> A2["Fetch Today's Matches\nAPI-Football /fixtures"]
        A2 --> A3["Fetch Next UTC Matches\n(timezone overlap)"]
        A3 --> A4["Format Match Summary\nfilter + build message"]
    end
    subgraph Night["Night Pipeline — 23:00 CR"]
        B1["Schedule Trigger\ncron 0 23 * * *"] --> B2["Fetch Today's Results"]
        B2 --> B3["Fetch Next UTC Results"]
        B3 --> B4["Format Results Summary\nfilter + fetch scorers + chunk"]
    end
    A4 --> S["Send WhatsApp Message\nTwilio"]
    B4 --> S
```

### Key design decisions

- **Timezone-correct day boundaries.** Costa Rica is `UTC-6`. A match at 8 pm CR falls on the *next* UTC calendar day, so each pipeline fetches **both** today and tomorrow (UTC), deduplicates, and clips everything back to the Costa Rica calendar day using Luxon. This avoids the classic "missing evening matches" bug.
- **Business filtering in code.** Only leagues `1` (World Cup) and `10` (Friendlies); only WC2026-qualified nations; youth teams (`U17`, `U20`, `U23`…) excluded via regex.
- **Rich results.** The night digest calls `/fixtures/events` per finished match to list scorers with minute, plus `(P)` penalty and `(OG)` own-goal tags, correctly crediting own goals to the opposing side.
- **WhatsApp-safe chunking.** Long result days are split into ≤ 1500-character messages so nothing is truncated by the provider.
- **Fan-out delivery.** A single formatted message is delivered to every recipient in one execution.

## Stack

| Area | Tech / Technique |
| :--- | :--- |
| Orchestration | **n8n** (Schedule triggers, HTTP Request, Code, Twilio nodes) |
| Messaging | **Twilio WhatsApp API** |
| Data source | **API-Football** (`v3.football.api-sports.io`) |
| Scheduling | Cron expressions (`0 7 * * *`, `0 23 * * *`) |
| Logic | JavaScript Code nodes · Luxon date math · regex filters · dedup |
| Deployment | **Railway** (localhost → cloud production) |
| Secrets | Environment variables + n8n Credentials (kept out of code) |

## Prerequisites

Before importing, make sure you have:

1. A running **n8n** instance (self-hosted, Railway, or n8n Cloud).
2. An **API-Football** API key — sign up at [api-football.com](https://www.api-football.com/).
3. A **Twilio** account with the WhatsApp sender enabled ([Sandbox](https://www.twilio.com/console/sms/whatsapp/learn) works for testing).
4. Recipient WhatsApp numbers in **E.164** format (e.g. `+50688889999`).

## Getting started

### 1. Import the workflow

```bash
git clone https://github.com/imkhub1/n8n-wc2026-football-alerts.git
cd n8n-wc2026-football-alerts
```

Open your n8n editor → top-right menu → **Import from File**, then select [`workflow/wc2026-football-alerts.json`](workflow/wc2026-football-alerts.json).

### 2. Configure credentials

This repository ships with all real secrets removed. Replace the following placeholders:

| Placeholder in the JSON | What it is | Where to set the real value |
| :--- | :--- | :--- |
| `YOUR_API_FOOTBALL_KEY` | API-Football key | HTTP Request header `x-apisports-key` (or an n8n **Header Auth** credential) |
| `+10000000001 … 5` | Recipient WhatsApp numbers | The `recipients` array inside the two **Code** nodes |
| Twilio credential | Account SID / Auth Token | n8n **Credentials → Twilio API** (mapped by name `Twilio account`) |

Copy [`.env.example`](.env.example) to `.env` and fill in your values:

```bash
cp .env.example .env
```

> [!TIP]
> Instead of hardcoding the API-Football key directly in the HTTP nodes, create an n8n **Header Auth** credential and reference it — the key will never appear in the exported JSON.

### 3. Activate

Enable the workflow. The two schedule triggers will fire at 07:00 and 23:00 in your configured timezone.

## Deployment: localhost → Railway

This project started on **localhost** for development and was promoted to a **production deployment on [Railway](https://railway.app)** to run 24/7.

### Why Railway?

- One-click n8n deployment with a **persistent volume** and managed **PostgreSQL**.
- A **public HTTPS domain** out of the box — required for n8n's editor, webhooks, and OAuth callbacks.
- Simple environment-variable management for secrets.
- Always-on and cost-effective for a personal automation on a daily cron.

### Local vs. production

| Concern | Localhost (dev) | Railway (production) |
| :--- | :--- | :--- |
| Database | SQLite file | PostgreSQL (managed, persistent) |
| Public access | `localhost:5678` | Public HTTPS domain (`*.up.railway.app`) |
| Webhooks | not reachable | `WEBHOOK_URL` set to the public domain |
| Secrets | `.env` on local machine | Railway environment variables |
| Credential encryption | local key | stable `N8N_ENCRYPTION_KEY` env var |
| Timezone | system default | `GENERIC_TIMEZONE=America/Costa_Rica` |

### Core environment variables

```env
N8N_HOST=your-instance.up.railway.app
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-instance.up.railway.app/
GENERIC_TIMEZONE=America/Costa_Rica

DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=...
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=...

N8N_ENCRYPTION_KEY=<stable-random-32+-chars>
```

> [!WARNING]
> Keep `N8N_ENCRYPTION_KEY` **stable** across redeploys — if it changes, n8n can no longer decrypt previously saved credentials.

See [`.env.example`](.env.example) for the full annotated list.

## Notable techniques

- **Cloud deployment** — promoted a local prototype to a real production service on Railway (managed Postgres, persistent storage, public HTTPS domain).
- **Secrets management** — all credentials externalized to environment variables / n8n Credentials; the repo is safe to publish with zero leaked secrets.
- **Timezone-correct scheduling** — robust handling of the `UTC ↔ UTC-6` day-boundary problem with Luxon, avoiding missed late-evening fixtures.
- **Third-party API integration** — primary and secondary calls to API-Football, including a per-match call to enrich results with scorers.
- **Data shaping** — dedup, business-rule filtering, regex exclusions, and provider-aware message chunking.
- **Notification fan-out** — a single computed payload delivered to N recipients per run.
- **Documentation & DX** — importable export, annotated `.env.example`, bilingual docs, and in-canvas sticky notes explaining each branch.

## Repository structure

```
.
├── workflow/
│   └── wc2026-football-alerts.json   # Importable n8n workflow (secrets scrubbed)
├── docs/
│   └── README.md                     # Screenshot guidance
├── .env.example                      # Annotated environment variables
├── .gitignore
├── LICENSE
├── README.md
└── README.es.md                      # Spanish version
```

## Author

Built by [@imkhub1](https://github.com/imkhub1). Issues and PRs are welcome.