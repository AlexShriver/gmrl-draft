# GMRL Draft Site — Spec (React version)

> **Draft, 2026-10-21.** This version moves the whole frontend to a React single-page app, with FastAPI as a JSON API. It was adapted from the snapshot exported 2026-10-10, which is kept as `SPEC-jinja-server-rendered-2026-10-10.md`.
> **Not yet in the living doc.** The living doc is still the server-rendered version: https://claude.ai/code/artifact/afea4ea0-d688-4edf-8148-728b9e08e1aa. Decide which one is current before starting Milestone 0.
> The doc's two diagrams don't export; they're described in words below.

## Overview and goals

We'll build a web app, hosted on this machine, where all 12 GMRL managers sign in remotely and run the whole draft: first the live auction with a timer, then the 17-round ULTRA draft. It replaces the paper draft.

**Primary goal: learning.** The project is a way to learn how web servers work and how a modern React frontend talks to one. So the spec favors approaches that show the moving parts (HTTP requests, sessions, a database, real-time updates, hosting) over frameworks that hide them. When a library would do something for us, the milestones say what it's doing underneath.

**Why React makes the wire easier to see, not harder.** The React app talks to the server only through `fetch` and `EventSource`. Every interaction is a JSON request or an SSE message you can read in the DevTools Network tab or reproduce with `curl`. The server never sends HTML after the first page load.

**Product goals**

- **Easy for everyone.** Several managers are older. Big text, few buttons, and nothing to install.
- **Remote and live.** Every manager sees bids and picks as they happen, from their own device.
- **Can't lose the draft.** Every action is saved before it's shown, so a crash or restart loses nothing.
- **Enforces league rules.** Salary cap, roster limits and the league's other rules are checked by the server, with a commissioner override.
- **Starts from OnRoto.** Rosters and contracts are imported from the league's OnRoto site so nobody retypes them.

**Out of scope for now:** in-season management, trades outside the draft, stats and scoring. OnRoto keeps doing all of that.

## Draft rules (as currently understood)

The draft has three stages: import keepers, auction until every roster is full, then the ULTRA draft of 17 more rounds. Detailed rules come later. This section records only the general shape the software must support.

1. **Keepers and contracts.** Each team starts with some players already on its roster. Every player has a contract with a salary. Each team has a total salary cap and must be well under it before the draft starts.
2. **Auction phase.** Teams take turns nominating a player. Everyone can bid. A countdown timer runs, and when it hits zero the highest bidder signs that player to a contract for the winning bid. The auction fills each team's 23-man starting roster.
3. **ULTRA draft.** Once every starting roster is full, the ULTRA draft starts: a standard draft in order 1 to 12, until each team has filled 17 bench spots. The salary cap doesn't apply in ULTRA. Each pick gets a contract set by its round (for example, round 1 is $15 for 3 years).

**Rules the app will need to know (details to come)**

- Salary cap amount, and what "well under" means before the draft
- Roster size and position requirements that define a "full" team
- Nomination order, minimum bid, bid increments and timer length (including whether a new bid resets the timer)
- Max bid a team can make, given its remaining cap and open roster spots
- Contract length and terms for auction players and for ULTRA picks
- ULTRA pick order: same order every round, or snake, and whether traded picks apply
- Who is eligible to be drafted in each phase

## Users, roles and sign-in

Each manager signs in by tapping their team in a list and entering a 4-digit PIN. The server then remembers them with a session cookie, so they sign in once per device.

| Role | Who | Can do |
| --- | --- | --- |
| Manager | One per team, 12 total | Watch the draft. Nominate and bid when allowed. Make their own ULTRA picks. |
| Commissioner | You (also a manager) | Everything a manager can do, plus import, start, pause, undo, act for any team, and override rules |
| Spectator | Anyone who chooses Spectator | Watch only. No PIN needed. |

**How sign-in works**

1. When a user opens the site, the React app calls `GET /api/me`. A `401` means nobody is signed in, so it shows the sign-in screen with a list of every team in the league (from `GET /api/teams`).
   1. To log in as this team ("Manager"), the user will have to enter their team's 4-digit PIN. The commissioner gives each manager their PIN before the draft. This prevents anyone from logging in as the wrong team, by accident or on purpose.
   2. There will be a "Spectator" option that does not have permission to draft anyone
   3. When signing in with any team, there will be an option to sign in as a "Commissioner", which requires a password.
2. The sign-in form sends `POST /api/login` with a JSON body. On success the server sets the session cookie and returns `200` with the team. React then switches to the draft room itself; there is no server redirect.

**Security basics** (all part of the learning)

- **PINs:** random 4-digit numbers, generated in the Admin page and shown to the commissioner once. The database stores only a hash (Python's built-in `hashlib.scrypt`), never the PIN itself. The commissioner can reset a forgotten PIN.
- **Limited guesses:** there are only 10,000 possible PINs, so after 5 wrong tries a team is locked for 10 minutes. Every failed try is logged.
- **Commissioner password:** long, hashed the same way, and kept in a config file outside the code, never committed to git.
- **Session cookie:** after a correct PIN, the server creates a long random token (`secrets.token_urlsafe`), stores its hash in `sessions`, and sends it as a cookie marked `HttpOnly`, `Secure` and `SameSite=Lax`. Sessions expire after the draft.
- **The token never touches JavaScript.** Because the cookie is `HttpOnly`, React can't read it, and neither can any injected script. The browser attaches it to every same-origin `fetch` automatically. We don't keep tokens in `localStorage`.
- **Same origin, always:** the React files and the API are served from the same address, so there's no CORS to configure. In development, Vite's proxy gives the same result (see Architecture).
- **JSON-only POSTs:** every state-changing route requires `Content-Type: application/json`. Combined with `SameSite=Lax`, this stops another site from submitting a form that bids as you.
- **More than one device is fine:** a manager can be signed in on a laptop and a phone at once.
- **The server decides:** every state-changing request re-checks who the user is and what they're allowed to do. React hides buttons a team can't use, but that's for convenience only. The server never trusts it.

## Architecture and tech stack

One Python process on this machine serves the built React files, every API call and live update, and stores everything in one SQLite file. Node is used only on the development machine to build the React app. It doesn't run in production.

*Diagram:* Manager browsers ⇄ (HTTPS) ⇄ Cloudflare ⇄ cloudflared tunnel client ⇄ **FastAPI app**, all on this machine. The app does three jobs: it serves the **React bundle** (static files from `web/dist/`), answers the **JSON API** (`/api/...`), and keeps the **SSE stream** open (`/api/events`). Behind the API it calls three internal parts: the **Importer** (reads OnRoto), **Rules** (checks actions) and **SQLite** (`draft.db`).

Managers reach the app through Cloudflare, which forwards traffic down a tunnel the machine opened. So no router ports are opened. The app calls the rules module before accepting any action and writes to SQLite before telling anyone.

**What a web server actually does.** A browser sends an HTTP request: a method (`GET`, `POST`), a path (`/api/players`), headers (including cookies) and sometimes a body. The server matches the path to a function, runs it, and returns a response: a status code (`200`, `401`, `409`), headers, and a body. Everything in this project is a variation on that loop.

**How a single-page app loads.** The first request for any page returns the same nearly empty `index.html`: one `<div id="root">` and a `<script>` tag. The browser downloads the JavaScript bundle, and React draws the page and fetches JSON for it. After that, clicking Draft · Players · Teams · History never loads a new HTML page. React Router changes the URL with the browser's History API and swaps components. You can see this in the Network tab: one document request, then only `fetch` (XHR) requests.

**Development setup.** Two servers run side by side:

- **Uvicorn** on `localhost:8000` runs the FastAPI app.
- **Vite** on `localhost:5173` serves the React source with instant reload. Its `proxy` setting forwards `/api` (including the SSE stream) to port 8000.

The browser only ever talks to 5173, so it sees one origin: cookies just work and there's no CORS. In production, `npm run build` writes plain files to `web/dist/`, FastAPI serves them, and Vite isn't involved at all.

| Layer | Choice | Why, for learning |
| --- | --- | --- |
| Language | Python 3.12+ on the server, JavaScript (JSX) in the browser | Each language where it's strongest |
| Web framework | FastAPI, run by the Uvicorn server | Small, explicit routes. You can see each request's path, method, cookies and body. |
| API format | JSON over plain HTTP | Readable with `curl` and the Network tab. Pydantic models describe each request and response. |
| Frontend | React 18 with function components and hooks | The industry-standard way to build interactive UIs: components, props, state, effects |
| Routing (browser) | React Router | Draft · Players · Teams · History as client-side routes, with filters kept in the URL |
| Build tool | Vite | Fast dev server with a proxy, and a one-command production build. Needs Node, on the dev machine only. |
| Data fetching | Plain `fetch` in a small `api.js` helper | No React Query or Redux yet, so you see every request and how loading and error states are handled |
| Styling | Plain CSS files | Nothing to learn but CSS |
| Live updates | Server-Sent Events (SSE) | One long-lived HTTP response the server keeps writing to, and the simplest real-time option. React subscribes in a `useEffect`. |
| Database | SQLite through Python's built-in `sqlite3` | One file, real SQL, no separate database server. Writing raw SQL teaches more than an ORM. |
| Scraping | `httpx` + BeautifulSoup | Fetch OnRoto pages and parse the HTML tables |
| Hosting | systemd + Cloudflare Tunnel | Runs on this machine and is reachable from anywhere |

**Project layout**

```
server/          FastAPI app, rules, importer, SQLite code (Python)
  tests/         pytest
web/             React app (Vite project)
  src/
    api.js       fetch helper: JSON in and out, turns 401/409 into errors React can show
    App.jsx      routes and the shared layout (top bar + nav)
    pages/       DraftPage, PlayersPage, TeamsPage, HistoryPage, AdminPage, LoginPage
    components/  TopBar, BidPanel, PlayerTable, PlayerPopup, Board, ...
    live.js      the SSE subscription and the state it keeps
  dist/          build output (gitignored), served by FastAPI in production
```

**Stretch options:** after v1 works, rewrite a small piece with the standard library `http.server`, no framework, to see what FastAPI does for you. On the frontend, build one widget (the countdown) with plain DOM calls to see what React does for you.

## Data model

The core of the data model is an append-only `events` table: every nomination, bid, sale, pick, undo and override is one row, and the current draft state is rebuilt from those rows. The other tables hold reference data and sessions.

| Table | Key columns | Purpose |
| --- | --- | --- |
| `teams` | id, name, onroto_id (0–11), manager_name, email, pin_hash, is_commissioner | The 12 teams |
| `players` | id, name, mlb_team, positions, onroto_id | Every draftable player, from the OnRoto player pool |
| `projections` | player_id, kind (hitter / pitcher), stats JSON, value ($$), imported_at | Projected stats from the OnRoto rankings pages. Hitters and pitchers have different stats, so they're stored as JSON and sorted with json_extract. |
| `contracts` | team_id, player_id, salary, years, type (FA or OY), source (keeper / auction / ultra) | Who owns whom, and for how much. Keepers load at import. |
| `drafts` | id, season, phase (setup / auction / ultra / done), settings JSON | One row per season's draft. Settings hold cap, timer length and so on. |
| `events` | seq, draft_id, type, team_id, payload JSON, created_at, actor | **Source of truth.** Append-only, and `seq` gives a strict order. |
| `sessions` | token_hash, team_id, created_at, expires_at | Signed-in browsers |
| `login_attempts` | team_id, attempted_at, succeeded | PIN tries, for the lockout and an audit trail |

**Event types (first cut):** `draft_started`, `nominated`, `bid_placed`, `lot_sold`, `pick_made`, `phase_changed`, `undone` (points at an earlier `seq`), `paused`, `resumed`, `override`.

**Why an event log?** It's easy to recover (replay the rows), easy to undo (add an `undone` event and don't delete anything), and it gives you a full history for settling disputes. It's also a classic pattern (event sourcing) worth learning. Rebuilding state from roughly 1,000 events takes milliseconds, so the server can simply do it at startup and keep the state in memory.

The same idea runs in the browser on a small scale: React loads a full snapshot from `GET /api/state`, then applies each SSE event to it (a reducer: state + event → new state).

## HTTP routes and live updates

The server has two kinds of routes. Everything under `/api/` returns JSON (or the SSE stream). Every other path returns `index.html`, and React Router decides which page to show.

**Pages (client-side routes, handled by React Router)**

| Path | Who | Page |
| --- | --- | --- |
| `/login` | Anyone | Sign-in: every team as a big button, plus Spectator |
| `/` | Signed in | The draft room: current phase, auction or pick panel, board |
| `/players?pos=SS&sort=HR&q=sot` | Signed in | Available players, with filters and sort kept in the URL |
| `/players/:id` | Signed in | One player, as a full page (the same content as the popup), so links can be shared |
| `/teams/:id` | Signed in | One team's roster, money left and players still needed |
| `/history` | Signed in | Every auction sale and ULTRA pick in order |
| `/admin` | Commissioner | Control panel |

**SPA fallback.** If someone refreshes `/teams/4` or opens a shared link, the browser asks the *server* for `/teams/4`. FastAPI has no such API route, so a catch-all returns `index.html` with `200`, and React Router shows the right page. Paths under `/api/` never fall back. An unknown API path is a real `404`.

**API routes (JSON)**

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| GET | `/api/me` | Anyone | The signed-in team and role, or `401` if nobody is signed in |
| GET | `/api/teams` | Anyone | All 12 teams (names only), for the sign-in screen |
| POST | `/api/login` | Anyone | Body `{team_id, pin}` (plus `password` for commissioner). Sets the session cookie and returns `200` with the team. Wrong PIN gives `401` with `{"error": "..."}`, and a locked team gives `429`. |
| POST | `/api/logout` | Signed in | Ends the session and clears the cookie |
| GET | `/api/state` | Signed in | Full current draft state: phase, current lot, high bid, deadline, who's on the clock, your cap room and max bid. React loads this on start and after any reconnect. |
| GET | `/api/teams/{id}` | Signed in | One team's roster by position with projected stats, money left and players still needed |
| GET | `/api/players?pos=SS&sort=HR&q=sot` | Signed in | Available players with projected stats. `pos` filters by position, `sort` sorts by a stat, `q` searches by name |
| GET | `/api/players/{id}` | Signed in | One player: projected stats, plus fantasy team, salary and contract years if drafted |
| GET | `/api/history` | Signed in | Every sale and pick in order |
| GET | `/api/events` | Signed in | SSE stream of live updates |
| POST | `/api/auction/nominate` | Team whose turn it is | Nominates a player with an opening bid |
| POST | `/api/auction/bid` | Any team that can afford it | Places a higher bid |
| POST | `/api/ultra/pick` | Team on the clock | Makes an ULTRA pick |
| POST | `/api/admin/import` | Commissioner | Runs the OnRoto import, or accepts uploaded HTML files |
| POST | `/api/admin/start`, `/pause`, `/resume` | Commissioner | Controls the draft |
| POST | `/api/admin/undo` | Commissioner | Undoes the last action |
| POST | `/api/admin/act-as/{team}` | Commissioner | Nominates, bids or picks for a team |
| POST | `/api/admin/pins/{team}` | Commissioner | Generates or resets a team's PIN and unlocks a locked-out team |
| GET | `/api/export.csv` | Anyone signed in | Draft results download (a plain link, not `fetch`, so the browser saves the file) |

**Every POST handler follows the same five steps:**

1. **Who?** Read the session cookie and look up the team. Nobody signed in means `401`.
2. **Allowed?** Ask the rules module whether this team can do this now. If not, return `409` with `{"error": "Max bid $4"}`, a plain-English reason React shows to the user.
3. **Save.** Append the event in a database transaction.
4. **Apply.** Update the in-memory state.
5. **Broadcast.** Push the event to every open `/api/events` stream.

A single lock around steps 2–5 means two bids arriving at once are handled one after the other, never interleaved.

**How React handles responses.** `api.js` wraps `fetch`: it sends JSON, reads JSON, and turns a non-2xx status into an error carrying the server's message. Components then do one of three things: a `401` sends the user to `/login`, a `409` shows the reason next to the button, and anything else shows a "Something went wrong, try again" notice. A successful bid doesn't update the screen directly. The screen changes when the SSE event comes back, so the bidder and everyone else see exactly the same thing.

**How SSE works.** The browser opens the stream with `new EventSource('/api/events')`. The server keeps that response open and writes lines like `event: bid` and `data: {...}` whenever something happens. If the connection drops, the browser reconnects automatically and sends `Last-Event-ID`, so the server can replay what it missed. It's plain HTTP, which makes it easy to watch with `curl -N`.

**SSE in React.** The shared layout opens one `EventSource` in a `useEffect` and closes it in the effect's cleanup function, so changing pages never opens a second stream. Each message is passed to a reducer that updates the draft state, and that state is shared with every page through React Context. That's why the top bar stays current wherever you are.

## Auction phase

The server owns the auction clock. Browsers only display a countdown to a deadline the server sets, so every manager sees the same result regardless of lag.

*Diagram (see the doc):* Nomination → (opening bid) → **Bidding** (a higher bid resets the timer, looping back) → (timer hits 0) → Sold to top bid → Rosters full? — yes: ULTRA draft; no: the next team nominates.

One player at a time is up for bid (a "lot"). Nominating opens bidding, each higher bid resets the timer, and when the timer expires the lot is sold. Then the next team nominates, until every roster is full.

**Nomination**

- Nomination order rotates through teams, skipping any team whose roster is already full.
- The nominating team picks a player from search and enters an opening bid, which counts as their first bid.

**Bidding**

- Big buttons for **+$1** and **+$5**, plus a box for a custom amount. Each shows exactly what you'll bid: "Bid $23".
- The server rejects a bid if it isn't higher than the current bid, or if it's more than the team's **max bid** (the cap room left after reserving the minimum salary for each open roster spot), or if the team has no open roster spot the player is eligible for. A player eligible at several positions can fill an open spot at any of them. The exact slot rules come with the Rules module.
- A team that can't legally bid sees the bid buttons disabled, with the reason: "Roster full", "Max bid $4" or "Position unavailable".
- While a bid request is in flight, its button shows "Bidding…" and is disabled, so a double-click can't send two bids.

**The timer (a key learning piece)**

- When a bid is accepted, the server stores `deadline = now + N seconds` and broadcasts it.
- Browsers draw the countdown from that deadline. They never decide when the lot is over.
- In React, a `Countdown` component runs a `setInterval` in a `useEffect` (cleared in the cleanup) and redraws from `deadline − now`. It never counts down a number of its own, so a sleeping laptop or a slow tab can't drift.
- A background task on the server wakes at the deadline. If no newer bid arrived, it records `lot_sold` and broadcasts it.
- Late bids: a bid arriving after the deadline is rejected, even if the button was clicked a moment before. A small grace period (for example 1 second) can soften network lag.
- When the commissioner pauses, the remaining time is saved. Resuming sets a new deadline from it.

**What each manager sees**

- Top: the player up for bid, the current high bid and bidder, and a large countdown that turns red under 5 seconds. This top bar shows on every page (see Pages under UI and accessibility).
- Middle: the bid buttons, their own cap room and max bid, and open roster spots.
- Bottom: a running list of recent sales. Full results are on the History page and rosters on the Teams page.

## ULTRA draft phase

The ULTRA draft is 17 rounds × 12 teams = 204 picks, made one at a time in a fixed order. It's simpler than the auction and reuses the same search, rules check and live-update machinery.

- **Order:** 1 to 12 each round. Whether it snakes, and how traded picks apply, is an open question. The order is stored as a list of 204 slots so any of those rules can fill it.
- **On the clock:** a banner across every screen says who's picking, for example "**On the clock: Newfane Mets** (Round 3, Pick 5)". The team on the clock also sees a large search box and a **Draft** button.
- **Confirm step:** "Draft Juan Soto to Newfane Mets? [Yes] [No]". Big buttons, so it's hard to click the wrong one.
- **Pick timer:** optional, off by default. If it's on, it works like the auction timer, but expiry only alerts the commissioner. Nothing is auto-picked.
- **Board:** a 12-column × 17-row grid that fills in live, plus a list of the next five picks.
- **Commissioner:** can pick for any team (for a manager on the phone), skip a team and come back to them, or undo the last pick.

## OnRoto import

The commissioner imports once before the draft, reviews the result and fixes anything wrong. The live draft never talks to OnRoto.

| OnRoto page | Path (league GMRL) | Gives us |
| --- | --- | --- |
| Team roster | `display_roster.pl?GMRL+11+N` (N = 0–11) | Each team's keepers and contracts |
| League pool | `display_mlb_rosters.pl?GMRL+11` | Players available to draft |
| Hitter projections | `team_get_hitter_rankings.pl?GMRL+11` | Projected hitting stats and $ value |
| Pitcher projections | `team_get_pitcher_rankings.pl?GMRL+11` | Projected pitching stats and $ value |
| Draft pick trades | `display_draft_choices.pl?GMRL+11` | Traded picks, for the ULTRA order |
| Final rosters | `display_roster.pl?GMRL+11+all+YEAR` | Past seasons, useful for testing |

- **Session:** every page needs a `session_id` in the URL, and it expires. The importer logs in to OnRoto itself to get a fresh one (see below), so the commissioner just clicks Import.
- **Parsing:** fetch with `httpx` and parse the HTML tables with BeautifulSoup. Save raw HTML to `imports/` so the parser can be rerun and tested without hitting OnRoto again.
- **Projections:** keep these columns and ignore the rest (Pts and the extra info at the end of each row).
  - Pitchers: Player, Team, IP, W, ERA, WHIP, SO, HOSV, $$ (projected value).
  - Hitters: Player, Team, Positions, AB, R, HR, RBI, SB, AVG, $$ (projected value). This list is from memory; confirm it against the page, and which two columns after AVG to skip.
- **Review screen:** after import, the Admin page shows each team's roster, total salary and cap room, and flags anything odd (a team over the cap, unparseable rows).
- **Be polite:** about 15 requests per import, one at a time. Never scrape during the draft.
- **Security:** treat the OnRoto password and every session link as passwords. Never commit them or write them to logs.

**Automatic OnRoto login**

The importer gets its own session ID by logging in the way a browser does, so nobody copies links by hand. At the start of the OnRoto milestone, we watch a real login in the DevTools Network tab (with "Preserve log" on) to learn OnRoto's exact login URL and form fields.

1. **Credentials:** the commissioner's OnRoto username and password live in the gitignored `.env` file. They are never stored in SQLite, shown on a page or written to logs.
2. **Load the login form:** `GET` the OnRoto login page and parse its `<form>` with BeautifulSoup: the `action` URL, the field names and any hidden fields. Reading these from the page instead of hard-coding them keeps the login working if OnRoto tweaks the form.
3. **Submit it:** `POST` the username, password and hidden fields from one `httpx.Client`, which keeps any cookies OnRoto sets.
4. **Find the session ID:** take `session_id` from the redirect's `Location` header, or from a link on the page that comes back.
5. **Check it:** fetch one league page and confirm it is real data, not the login form again. A login page returned with `200 OK` counts as a failure.
6. **Use it, then drop it:** the session ID stays in memory for that one import run and is never saved.

If the login fails (wrong password, a changed form, a CAPTCHA), `/api/admin/import` says so plainly and the Admin page offers two fallbacks. The commissioner can paste a session link, or open each OnRoto page in a browser, save it, and upload the saved HTML files. Uploaded files go to `imports/` and run through the same parser, so this still works if a bot check (like Cloudflare's "Verify you are human") ever blocks scripted requests. A test on 2026-10-10 got `200 OK`, so for now the scripted fetch works.

## Persistence, recovery and undo

Every action is written to SQLite and committed before any browser is told about it. A crash or reboot loses at most an action that was never confirmed.

- **On startup:** load the settings, replay `events` in `seq` order to rebuild state, and resume. If a lot was mid-auction, it comes back **paused**, so the commissioner resumes it and nobody loses a bid to the downtime.
- **Browsers reconnect by themselves** (SSE retry) and React reloads `GET /api/state`, so managers see a short "Reconnecting…" notice, then carry on. The React app is already loaded in the browser, so it keeps showing the last known state while the server is down instead of an error page.
- **Undo:** the commissioner's undo appends an `undone` event that cancels the latest action. Replay skips cancelled events. Nothing is deleted.
- **Durability settings:** SQLite in WAL mode, with `synchronous=FULL` for commits that survive power loss.
- **Backups:** copy `draft.db` every 5 minutes during the draft with SQLite's backup API, keep the last 50 copies, and put a copy somewhere off the machine after the draft.
- **Export:** `/api/export.csv` gives results anyone can open in a spreadsheet, and doubles as a paper fallback. Three details make it open cleanly in Excel:
  - Encoded as UTF-8 with a BOM (Python's `utf-8-sig`), so accented names like Acuña aren't garbled.
  - Sent with `Content-Disposition: attachment; filename="gmrl-draft-YEAR.csv"`, so the browser downloads it instead of showing it.
  - Any cell starting with `=`, `+`, `-` or `@` is escaped, so Excel never runs it as a formula.

## Hosting from this machine

The app runs on this Linux machine as a systemd service, and Cloudflare Tunnel makes it reachable at a public HTTPS address. No router changes are needed, and your home IP stays hidden.

1. **Build the frontend:** `npm run build` in `web/` writes static files to `web/dist/`. Run it before each deploy. File names include a content hash (`index-3f9a1c.js`), so they can be cached forever, while `index.html` is sent with `Cache-Control: no-cache` so browsers always pick up a new build.
2. **The app** listens only on `127.0.0.1:8000`, so nothing on your network can reach it directly. It serves `web/dist/`, the API and the SSE stream.
3. **systemd** (`draft.service`) starts it on boot and restarts it if it crashes. Logs go to `journalctl -u draft`.
4. **cloudflared** (`cloudflared.service`) keeps an outbound tunnel open to Cloudflare, and Cloudflare forwards `draft.<yourdomain>` to `localhost:8000`.
5. **Domain:** about $10 a year, managed in Cloudflare. Tailscale Funnel is a free fallback, but its address is harder to share.

**Draft-day checklist:** wired Ethernet, sleep and suspend disabled, laptop plugged in, a fresh build and backup, and a test sign-in from a phone on cellular data an hour before.

**Known risk:** a home internet or power outage pauses the draft for everyone. The event log means nothing is lost. If this proves flaky, the same code moves to a $5 cloud server.

## UI and accessibility

The site should be usable by a manager who has never seen it, on a laptop, tablet or phone, without instructions.

**Pages.** The site has four pages that share one layout component: a live top bar, then links to Draft · Players · Teams · History. The layout owns the SSE stream, so the top bar stays current wherever you are, and moving between pages is instant because nothing reloads.

- **Top bar, auction:** the player up for bid, the current high bid and bidder, and the timer.
- **Top bar, ULTRA:** who's on the clock, the round and pick, and the pick timer if it's on.

| Page | Path | Shows |
| --- | --- | --- |
| Draft | `/` | The auction panel (bid buttons, your cap room, max bid, open roster spots) and recent sales, or in ULTRA the pick panel and board. |
| Players | `/players` | Available players with projected stats. Filter by position and sort by any stat, alone or within a position, all in the URL (`/players?pos=SS&sort=HR`), so the back button and shared links work. Search filters as you type. Hitters and pitchers show their own stat columns. On your turn, each row has a Nominate button (auction) or Pick button (ULTRA). Nominate asks for the opening bid, then confirms. |
| Teams | `/teams/:id` | A dropdown picks the team. Its roster by position with projected stats, money left and players still needed. |
| History | `/history` | Every auction sale with its price and every ULTRA pick with its round and pick, in order. |

**Player popup.** Every player name, on any page, opens a popup (a modal component) that fetches `GET /api/players/{id}`: projected stats and, if drafted, the fantasy team, salary and contract years. The name is also a real link to `/players/:id`, so a middle-click or a shared link opens it as a full page.

**React components and what each teaches**

| Component | Shows | React idea |
| --- | --- | --- |
| `TopBar` | Phase, lot, high bid, countdown | Reading shared state from Context |
| `Countdown` | Seconds left, red under 5 | `useEffect` with an interval and cleanup |
| `BidPanel` | +$1, +$5, custom amount, reason when disabled | Event handlers, controlled inputs, in-flight state |
| `PlayerTable` | Filterable, sortable player list | Lists and `key`, URL state with `useSearchParams` |
| `PlayerPopup` | One player's details | Fetching on open, loading and error states, focus handling |
| `Board` | 12 × 17 ULTRA grid | Rendering derived data, keeping re-renders cheap |
| `ConfirmDialog` | "Draft Juan Soto to Newfane Mets?" | Reusable components and props |

- **Readable:** base text 18 px or larger, strong contrast, buttons at least 48 px tall, and no text in faint grey.
- **One job per screen:** the draft room shows what's happening now and what *you* can do. Everything else is one click away.
- **Plain words:** "Bid $23", "You can bid up to $41", "Waiting for Peeking Ducks to nominate". No jargon or icon-only buttons.
- **Forgiving:** a confirm step before nominating and before an ULTRA pick. Clear error messages that say what to do next.
- **Always oriented:** the live top bar on every page shows the phase, who's up, and the time left.
- **Accessible React:** real `<button>` and `<a>` elements (never clickable `<div>`s), focus moves into a popup when it opens and back when it closes, and live changes such as a new high bid are announced with an `aria-live` region.
- **Sound (optional):** a chime when it's your turn to nominate or pick.
- **Works everywhere:** a responsive layout and current Chrome, Safari and Edge. No app to install.
- **Test it:** before draft day, run a mock draft with two or three of the least technical managers and watch where they get stuck.

## Learning roadmap

Each milestone adds one web-server or React idea and ends with something you can run and see. Milestones 1–7 get you a working auction, and the rest make it real.

| # | Build | Idea you learn | Done when |
| --- | --- | --- | --- |
| 0 | Tiny server with Python's `http.server`, no framework | What a request and response really are | `curl -v localhost:8000` shows your headers and body |
| 1 | FastAPI hello world returning JSON | Routing, JSON responses, status codes, Pydantic models | `curl -v localhost:8000/api/teams` returns 12 fake teams as JSON, and an unknown path returns `404` |
| 2 | Vite + React app that shows those teams | How a SPA loads, components and props, `useEffect` + `fetch`, the dev proxy and same-origin | A browser shows 12 teams drawn by React, and View Source shows an almost empty `index.html` |
| 3 | SQLite tables (with fake projections), the Players API and page | SQL, query parameters, controlled inputs, URL state, React Router | `/players?pos=SS&sort=HR` lists shortstops by projected HR, and typing "sot" finds matching players |
| 4 | Team list, PIN sign-in and sessions | Cookies, `HttpOnly`, password hashing, rate limiting, `401` / `403` / `429`, auth state in React | Two browsers are signed in as two different teams, and a signed-out browser lands on `/login` |
| 5 | ULTRA picks with the event log, plus the Teams and History pages | POST handlers with JSON bodies, transactions, event sourcing, showing `409` reasons | Picks survive stopping and restarting the server, and show on Teams and History |
| 6 | Live updates over SSE, the shared top bar and the player popup | Long-lived responses, broadcasting, effects and cleanup, Context, reducers | A pick in one browser appears in the other instantly, on every page |
| 7 | The auction and its server timer | Background tasks, concurrency, locks, a countdown that can't drift | Two browsers bid against each other, and the timer sells the lot |
| 8 | Rules module with tests | Keeping logic separate from HTTP, `pytest` | League rules, including position eligibility ("Position unavailable"), are enforced, with a test for each |
| 9 | OnRoto login and importer | Being an HTTP *client*: form POSTs, cookies, redirects, parsing HTML | One click logs in to OnRoto, and real keepers, contracts and projections load and review cleanly |
| 10 | Production build, systemd plus Cloudflare Tunnel | Static files and caching, the SPA fallback, reverse proxies, TLS, services, logs | A friend on another network signs in and bids |
| 11 | Commissioner tools, polish, mock draft | Admin authorization, UX testing | A full mock draft with real managers works |

**How we'll work:** I write most of the code and explain it as we go. You write pieces yourself when you want the practice, and no code goes in until you understand it. At each milestone, look at the raw HTTP traffic in the browser's DevTools Network tab and with `curl -v`, and look at components and state with the React DevTools extension, so neither framework feels like magic.

## Testing

Most tests are fast unit tests, a fair number exercise real HTTP requests or render real components, and a few checks happen by hand in a real browser. Server tests use `pytest`, and frontend tests use Vitest with React Testing Library. A test is the "predict" step of each lesson written down.

- **Unit tests (most):** plain functions with no server or network. They cover the rules (`can_bid` returns OK or a reason such as "Max bid $4" or "Position unavailable"), event replay (events in, draft state out, which also covers crash recovery and undo), the OnRoto parser on saved sample pages, and the CSV export (BOM, escaped `=` cells, an accented name like Acuña). On the frontend, the reducer that applies SSE events is a plain function too and gets the same kind of tests.
- **Integration tests (some):** FastAPI's `TestClient` sends real requests straight into the app with a fresh temporary database, and checks status codes, headers, cookies and JSON bodies. Examples: a wrong PIN gives `401`; a right PIN gives `200` with `Set-Cookie`; a rule-breaking bid gives `409` with its reason; a pick survives rebuilding the app on the same database file.
- **Component tests (some):** React Testing Library renders a component with a fake `fetch` and checks what the user would see and do. Examples: the bid button shows "Max bid $4" and is disabled; clicking Bid shows "Bidding…" and a `409` reason appears; typing "sot" filters the player list.
- **End-to-end (few, by hand):** a real server with `curl -v`, two browser windows bidding against each other, then the mock draft. Browser automation (Playwright) is a heavy dependency, so we decide on it in Milestone 11.

**Three design choices keep the code testable:**

1. **The rules module never touches HTTP or the database.** It takes the current state and an action and returns a decision, so every league rule can have its own test.
2. **The timer takes its time from a clock we pass in.** Tests use a fake clock and move it forward by hand ("advance 30 seconds, the lot is sold"), so timer tests run instantly and give the same result every time. The React `Countdown` uses Vitest's fake timers the same way.
3. **Components get data through `api.js` and the live-state Context, never by calling `fetch` directly.** Tests swap in fakes for both.

| # | What we test | Kind |
| --- | --- | --- |
| 0 | The handler returns the right status line, headers and body | Unit, plus `curl -v` |
| 1 | Routes return `200` / `404`, and `/api/teams` returns 12 teams as JSON | Integration |
| 2 | The teams list renders from a fake `fetch`, and shows an error if it fails | Component |
| 3 | Filter and sort SQL, a name in a search can't break the SQL, and the filter controls update the URL | Unit, integration and component |
| 4 | PIN hashing, lockout after 5 tries, cookie flags, `401` / `403` / `429`, and a `401` sends React to `/login` | Unit, integration and component |
| 5 | Replaying events rebuilds the state, undo works, picks survive a restart, a `409` reason is shown | Unit, integration and component |
| 6 | `/api/events` sends the right `event:` / `data:` lines, reconnecting with `Last-Event-ID` replays missed events, and the browser reducer applies each event | Integration and unit, plus `curl -N` |
| 7 | The fake-clock timer sells the lot, late bids are rejected, two bids at the same instant are handled one at a time, the countdown redraws from the deadline | Unit, integration and component |
| 8 | Every league rule, including "Position unavailable" | Unit, the biggest set |
| 9 | Parsing saved sample pages (session IDs removed), and spotting a failed login | Unit |
| 10 | Deep links and refresh work through the SPA fallback; the site works from a phone on cellular data | Integration and manual |
| 11 | A full mock draft | Manual |

In the step-by-step loop, the user gets a failing test before writing a piece themselves, and reads the tests first for pieces Claude writes.

## Open questions

- [ ] Salary cap amount, minimum salary, and what "well under the cap" means before the draft
- [ ] Position slots in the 23-man starting roster, and how multi-position players fill them (coming with the Rules module)
- [ ] Nomination order, minimum opening bid, bid increments and timer length. Does each bid reset the timer to full, or add a few seconds?
- [ ] Contract terms: years and salary rules for auction players, and the full round-by-round contract table for ULTRA picks
- [ ] ULTRA order: straight or snake, and how traded picks from OnRoto apply
- [ ] Which players are eligible in each phase (MLB only, minor leaguers, the 40-man pool)?
- [ ] Is there a rules or constitution page on OnRoto to import from?
- [ ] Draft date, which sets how many milestones must be done by then
- [ ] Domain name for the site
- [ ] OnRoto login details: the login URL and form fields, whether it sets cookies as well as `session_id`, and whether there's a CAPTCHA or two-factor step
- [ ] Do OnRoto's terms allow a scripted login to your own league?
- [ ] Plain JavaScript or TypeScript for the React app? TypeScript is common in React jobs, but adds a type layer to learn at the same time.
- [ ] Which Node version to install on the dev machine (the current LTS is the default choice)
