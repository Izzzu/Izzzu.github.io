---
title: "A private finance assistant: monthly spending aggregation with locally hosted LLM"
date: 2026-10-19 09:00:00 +0000
tags: 
media_subpath: 
---

## Context

I wanted a single overview of my monthly spending. Every bank and credit card app offers one, neatly categorised — but I use several different card and bank providers, so no single tool could tell me how much I spend each month and where most of it goes.

So I thought: why not download the statements and pass them through an LLM? Voilà — that's how the app came to life.

## Requirements

The one hard requirement was a **privately hosted model**. My account statements should never leave my local machine (or server) to reach a public LLM API.

Everything else followed from that constraint.

## How it works

### Statement ingestion

Once a month I download the statements from my bank portals and upload them all at once. The upload doesn't need to be fast — here I trade speed for quality and privacy.

The flow:

1. The app accepts **PDF and CSV** files.
2. **PDFs** are turned into markdown page by page with `PyMuPDF4LLM`, which keeps tables as tables (see the [Appendix](#appendix) for the alternatives I compared). The LLM then reads each page into transactions.
3. **Generic checks** catch the model's mistakes, with no per-bank rules: every amount must be printed in the PDF, and the opening balance plus the movements must add up to the closing balance.
4. **CSVs** skip the LLM: the code detects the columns and formats itself.
5. **Categorisation:** my rules first, then the LLM for whatever is left. When the LLM is unsure, the transaction goes to a "To review" list.

Some categories and rules are predefined, but they evolve with the user as each new report comes in.

```
PDF ──► PyMuPDF4LLM ──► markdown ──► LLM reads the rows ──► checks ──┐
                                                                     ├──► categorise ──► transactions
CSV ──► column detection ────────────────────────────────────────────┘         ▲
                                                                               │
                                                                   rules first, then the LLM
```
/claude add image assets/img/posts/2026-10-19/diagram.png

### The model

The choice for both the categorisation and parsing models settled on `qwen3:8b`, running via Ollama on my own hardware: my Mac locally, a T4 GPU in Azure. It worked well from the start and seems a good trade-off between reasoning quality and resource use.

Other models that I plan to test out when I reach any issues with my current choice:

| Model | Suggested for | RAM (approx.) | Trade-off |
|---|---|---|---|
| `qwen3:8b` (current) | Clean, common merchants; fast runs | ~5 GB (6-7 GB with context) | Probably weakest on messy or unusual merchants |
| `phi4:14b` | Tabular CSV/markdown input | ~9 GB (11-12 GB) | English-centric, so it may do worse on non-English merchants |
| `gpt-oss:20b` | Harder cases needing more reasoning; fast per token | ~13-14 GB (15-16 GB) | Tight on 16 GB; reasoning adds latency |

Useful reference for learning more about available models: https://huggingface.co/blog/daya-shankar/open-source-llm-models-to-run-locally

## Deployment and scaling

The app started as a local web app for private use. I later deployed it so the two of us in the household could use it too.

The usage profile is tiny: each user uploads up to **5 statements a month**, so there's very little data to move around.

It runs on **Azure Container Apps**, set up with Terraform:

- **The app** runs on one small, always-on replica.
- **The model** runs in Ollama on a **serverless T4 GPU**. It scales to zero about 9 minutes after the last request and is billed per second while running — a good fit for bursty, once-a-month usage. I uploaded the model files from my Mac, so Azure never downloads the model.
- **The price is the cold start:** waking the GPU and loading the model takes about 3 minutes, and my first scan from cold took about 6.
- **Cost:** an estimated $20–30 a month. The always-on app replica is the biggest part, not the GPU (1–3 GPU hours a month).

## Users and data

### Onboarding

Sign-in is handled by **Easy Auth**, the built-in authentication of Container Apps, with Microsoft Entra ID. I invite each person by email, and with *assignment required* enabled, only people I invite can sign in. They use their own email address: a Microsoft account signs in with its password, any other address gets a one-time code by email.

One surprise: guests can't consent to an app themselves, so my first invitee saw "Need admin approval" until I granted consent for the whole tenant.

### Per-user isolation

Easy Auth sends `X-MS-CLIENT-PRINCIPAL-ID` with every request — the user's Entra object ID, a stable GUID for that person. Outside requests can't forge this header; the platform sets it. Each user's data lives under `users/{object-id}/finance.db`, and the ID is validated against a GUID pattern.

### Storage

The project began local, storing everything in a single SQLite file — and it stayed that way. It's a lightweight solution, so I kept it when moving to the cloud: each user gets their own SQLite file, the same kind as today's local `data/finance.db`.

```
inside the container (local disk, temporary)            Blob Storage (permanent copy)
/data/users/3f2a…/finance.db   ← your data        ⇄    users/3f2a…/finance.db   (+ versions)
/data/users/9c41…/finance.db   ← partner's data   ⇄    users/9c41…/finance.db   (+ versions)
```

The container's local disk is a fast working copy. The app saves the whole file to Blob Storage a couple of seconds after each change, and Blob versioning keeps every save. Each user's file holds everything that's theirs — transactions, imports, categories, remembered rules, settings and the AI cache.

There's no write lock: the app assumes it's the only replica writing. During a deploy the old and new versions run side by side for about a minute, so I simply don't use the app while deploying — a shortcut that's fine for two people.

Deleting is final. SQLite overwrites deleted rows, and old copies in Blob Storage expire after about a month (14 days as versions, then 14 days in soft delete).

Nothing is shared between users. The only shared parts are the code, the Ollama model (stateless) and the public ECB exchange-rate file.

## Security

Financial data raises the stakes, so security shaped the design from the start rather than being bolted on. A few principles:

- **The data never leaves my infrastructure.** Ollama runs in its own container in my Azure subscription, reachable only from the app. Statements are never sent to a public LLM API. This was the original requirement, and everything else builds on it.
- **Only from home.** The app accepts requests only from my home IP address; everyone else gets a 403 before even reaching the sign-in page.
- **Only invited people can sign in.** There's no open sign-up — the app simply rejects anyone I haven't invited.
- **Users can't reach each other's data.** Isolation keys off `X-MS-CLIENT-PRINCIPAL-ID`. Easy Auth runs as a proxy in front of the app — it strips any client-supplied copy and injects its own after validating the Entra session, so the app only ever sees a trusted value, **provided the container accepts traffic only through the auth proxy**. The app validates the ID against a GUID pattern (blocking path traversal) and scopes every read and write to `users/{object-id}/`, so one account can't address another's file.
- **No keys to leak.** The app reaches Blob Storage with its managed identity, and storage account keys are turned off. The data is encrypted at rest by default.
- **No telemetry.** PyMuPDF4LLM's layout model runs on ONNX Runtime, which sends usage events to Microsoft by default. I turned that off (`ORT_DISABLE_TELEMETRY=1`).
- **One honest caveat:** as the operator, I can read every user's file in Blob Storage. That's fine for a household; otherwise per-user encryption would be the next step.

## Appendix

### Alternatives for statement ingestion

I compared a few ways to turn a PDF statement into something an LLM could categorise:

1. **PDF-to-markdown tool + LLM** (`qwen3:8b`) — the winning approach. The statements are structured tables from machine-generated documents, not scanned docs or images, so a simple PDF-to-markdown conversion works well enough; the LLM then works from that markdown.
2. **Deterministic** — pdfplumber + hardcoded rules and bank profiles (suggested by Claude) — fast and accurate, but barely maintainable: every new bank statement would need to be onboarded in the code.
3. **Vision LLM** — `qwen3-vl:8b`, `ZwZ-8B` — quite slow, several minutes to parse a one-page PDF on my local machine.
4. **OCR → text LLM** — `glm-ocr` → `qwen3:8b` — slow compared to (1), with comparable results.

Approach 1 won. Even when the conversion isn't perfect, the model handles the context well enough downstream.

### PDF into markdown

The goal: parse different kinds of structured PDFs into a format an LLM can understand. I scored the tools without a model on 7 synthetic statements (54 transactions): is each amount under a column header that says money in or out?

| Tool | Amounts under the right header | Verdict |
|------|---|---------|
| [pymupdf4llm](https://pymupdf4llm.readthedocs.io/) | 44/54 | **The one I use.** Keeps tables as tables, and read a real statement that pdfplumber struggled with. It's AGPL-licensed, so I relicensed the app as AGPL-3.0 too |
| [pdfplumber](https://github.com/jsvine/pdfplumber) | 0/54 raw, 54/54 with my own column labelling | My first choice: light and fast, but it needed my own code (tuned on these same statements) to keep amounts under their headers |
| [Docling](https://github.com/docling-project/docling) | 26/54 | Good results with the model, but by far the heaviest (1.3 GB install plus 1 GB of models) |
| [markitdown](https://github.com/microsoft/markitdown) | 11/54 | Tables only sometimes; wrote one statement column by column |
