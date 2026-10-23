# GMRL Draft Site

A web app for the Green Mountain Rotisserie League (GMRL) to run its annual draft remotely: a live timed auction, then the 17-round ULTRA draft. It will be hosted on this Linux machine. It replaces a paper draft, because the league's rules are too unusual for any existing draft site.

- **Spec:** `SPEC.md` (the React single-page-app version) is the main spec and the source of truth. `SPEC-orig.md` is the archived server-rendered Jinja2 plan, kept for comparison only. The old living doc (https://claude.ai/code/artifact/afea4ea0-d688-4edf-8148-728b9e08e1aa) still describes the Jinja2 version and is out of date.
- **Status:** planning. No application code yet. Next step: work out the detailed draft rules (see "Open questions" in the spec) before starting Milestone 0.

## How we work: this is a learning project

The user (the league commissioner, also a manager) is building this mainly to learn how web servers work, and how a React frontend talks to one.

- **Claude writes most of the code. The user must understand all of it.** Explain what each piece does and why, especially the HTTP-level mechanics (requests, responses, status codes, headers, cookies, SSE) that frameworks hide.
- **Go milestone by milestone** (see "Learning roadmap" in the spec). Keep each step small enough to read in one sitting. Don't build ahead of the milestone we're on.
- **Pause for questions** after each meaningful chunk rather than delivering a large finished feature.
- **Offer, don't impose:** when a piece is a good one for the user to write themselves, say so and let them choose.
- **Prefer explicit over magic:** raw SQL with `sqlite3` rather than an ORM; in React, plain `fetch` in a small `api.js` helper rather than React Query or Redux. Explain any new dependency (Python or npm) before adding it.
- **Show how to observe it:** suggest `curl -v`, `curl -N` (for SSE) or the browser's DevTools Network tab to see what's actually on the wire, and the React DevTools extension for components and state.
- **Boring over clever:** plain, readable code. Comments explain *why*, not *what*.

### The loop for every step

1. **Concept first:** explain the idea in a few sentences, including what goes over the wire, before writing code.
2. **User predicts:** ask what they expect to see (curl output, status code, database row). Wrong guesses are the point; use them to show where to look.
3. **Small step:** one function or route, roughly 20–60 lines. Never a whole feature at once.
4. **Walk-through:** go through the code line by line, then have the user explain it back or answer 2–3 questions. Stop if anything is fuzzy.
5. **Run and observe:** `curl -v`, the DevTools Network tab or `sqlite3 draft.db`, and compare with the prediction.
6. **Break it on purpose:** remove the cookie, send a bad PIN, kill the server mid-bid. The user predicts the failure, then checks it.
7. **Commit:** the user reads the diff and writes the commit message. If it doesn't fit in one line, the step was too big.

### Pieces to offer the user

Give a stub and a failing test, then review what they write. Always their choice.

| Milestone | Piece | Why |
| --- | --- | --- |
| 0 | Request handler in bare `http.server` | Building an HTTP response by hand |
| 3 | SQL for filtering and sorting on the Players page | Query parameters to SQL, and why `?` placeholders are safe |
| 4 | PIN check and lockout | Hashing and rate limiting in one small piece |
| 8 | Rules functions and their tests | The user knows the league rules best |
| 9 | Parsing one OnRoto table | Reading real HTML with BeautifulSoup |

### End of each milestone

- **Checkpoint:** the user traces one action end to end (for a bid: click, `POST`, auth, rules check, database write, SSE broadcast, other browser updates). Don't start the next milestone until they can.
- **Learning log:** the user keeps `NOTES.md` with three lines per session: what they learned, what surprised them, what's still unclear. Read it at the start of a session and use it for review questions.

## Stack (planned)

Python 3.12+, FastAPI + Uvicorn as a JSON API (`/api/...`) that also serves the built React files, a React 18 single-page app (function components and hooks, React Router, built with Vite; Node on the dev machine only), Server-Sent Events for live updates, SQLite with an append-only event log, `httpx` + BeautifulSoup for the OnRoto import, and systemd + Cloudflare Tunnel for hosting.

## Secrets: never commit or log these

- OnRoto session links (`...?session_id=...`). They act as a login to the user's OnRoto account.
- The OnRoto username and password the importer uses to log in automatically. Keep them in `.env` only, never in SQLite or logs.
- Team PINs, the commissioner password, and session tokens. Store only hashes.

Keep secrets in a gitignored config or `.env` file.

## Related

- Do not reference, read or reuse `~/Projects/OnrotoDrafter` (the user's earlier C++ draft assistant) for this project. Get league rules and data formats from the user and the spec instead.
- League on OnRoto: league `GMRL`. The user's team is YoungShriver (OnRoto team 11). Page paths are in the "OnRoto import" section of the spec.
