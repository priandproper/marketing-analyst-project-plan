# Marketing Funnel Analytics — Project

A from-scratch marketing-funnel analytics project built on PostgreSQL:
three tables, a synthetic dataset, analytical SQL, and a published console
— designed so every line can be defended in an interview.

## Start here

📘 **[CURRICULUM.md](CURRICULUM.md)** — the complete, step-by-step
build guide (setup → data model → SQL → judgment traps → console).

## Live site (GitHub Pages)

The published console is served from the `docs/` folder:

**https://priandproper.github.io/marketing-analyst-project-plan/**

Right now `docs/index.html` is a placeholder. To publish the finished
console (Module 8 of the curriculum), replace `docs/index.html` with the
console's HTML and push to `main` — GitHub Pages redeploys automatically.

## Enabling Pages (one-time)

Repo → **Settings** → **Pages** → Source: **Deploy from a branch** →
Branch: `main`, Folder: `/docs` → **Save**. The URL goes live in ~1 minute.

## Repository layout

| Path | What it is |
|---|---|
| `CURRICULUM.md` | The full teaching curriculum. |
| `docs/index.html` | The deployed site (placeholder → console). |
| `generate_data.py` | *(You build this in Module 3)* synthetic-data generator. |
| `*.sql` | *(You build these)* the analytical queries. |
