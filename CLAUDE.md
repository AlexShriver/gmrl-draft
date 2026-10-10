# GMRL Draft Site

A web app for the Green Mountain Rotisserie League (GMRL) to run its annual draft remotely: a live timed auction, then the 17-round ULTRA draft. It will be hosted on this Linux machine. It replaces a paper draft, because the league's rules are too unusual for any existing draft site.

- **Spec:** `SPEC.md` is a snapshot. The living doc is the source of truth: https://claude.ai/code/artifact/afea4ea0-d688-4edf-8148-728b9e08e1aa. If they disagree, ask which is current, and re-export `SPEC.md` after doc changes.
- **Status:** planning. No application code yet. Next step: work out the detailed draft rules (see "Open questions" in the spec) before starting Milestone 0.

## How we work: this is a learning project

The user (the league commissioner, also a manager) is building this mainly to learn how web servers work.

- **Claude writes most of the code. The user must understand all of it.** Explain what each piece does and why, especially the HTTP-level mechanics (requests, responses, status codes, headers, cookies, SSE) that frameworks hide.
- **Go milestone by milestone** (see "Learning roadmap" in the spec). Keep each step small enough to read in one sitting. Don't build ahead of the milestone we're on.
- **Pause for questions** after each meaningful chunk rather than delivering a large finished feature.
- **Offer, don't impose:** when a piece is a good one for the user to write themselves, say so and let them choose.
- **Prefer explicit over magic:** raw SQL with `sqlite3` rather than an ORM, plain JavaScript rather than a frontend framework. Explain any new dependency before adding it.
- **Show how to observe it:** suggest `curl -v`, `curl -N` (for SSE) or the browser's DevTools Network tab to see what's actually on the wire.

## Stack (planned)

Python 3.12+, FastAPI + Uvicorn, Jinja2 server-rendered pages, a little plain JavaScript, Server-Sent Events for live updates, SQLite with an append-only event log, `httpx` + BeautifulSoup for the OnRoto import, and systemd + Cloudflare Tunnel for hosting.

## Secrets: never commit or log these

- OnRoto session links (`...?session_id=...`). They act as a login to the user's OnRoto account.
- The OnRoto username and password the importer uses to log in automatically. Keep them in `.env` only, never in SQLite or logs.
- Team PINs, the commissioner password, and session tokens. Store only hashes.

Keep secrets in a gitignored config or `.env` file.

## Related

- Do not reference, read or reuse `~/Projects/OnrotoDrafter` (the user's earlier C++ draft assistant) for this project. Get league rules and data formats from the user and the spec instead.
- League on OnRoto: league `GMRL`. The user's team is YoungShriver (OnRoto team 11). Page paths are in the "OnRoto import" section of the spec.
