# GMRL Draft Site — Spec

> Snapshot of the living spec doc, exported 2026-10-10. The doc is the source of truth:
> https://claude.ai/code/artifact/afea4ea0-d688-4edf-8148-728b9e08e1aa
> Re-export this file when the doc changes. The doc's two diagrams don't export; they're described in words below.

## Overview and goals

We'll build a web app, hosted on this machine, where all 12 GMRL managers sign in remotely and run the whole draft: first the live auction with a timer, then the 17-round ULTRA draft. It replaces the paper draft.

**Primary goal: learning.** The project is a way to learn how web servers work, so the spec favors approaches that show the moving parts (HTTP requests, sessions, a database, real-time updates, hosting) over frameworks that hide them. When a library would do something for us, the milestones say what it's doing underneath.

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

1. When a user opens the site, they will be presented with a list of every team in the league.
   1. To log in as this team ("Manager"), the user will have to enter their team's 4-digit PIN. The commissioner gives each manager their PIN before the draft. This prevents anyone from logging in as the wrong team, by accident or on purpose.
   2. There will be a "Spectator" option that does not have permission to draft anyone
   3. When signing in with any team, there will be an option to sign in as a "Commissioner", which requires a password.

**Security basics** (all part of the learning)

- **PINs:** random 4-digit numbers, generated in `/admin` and shown to the commissioner once. The database stores only a hash (Python's built-in `hashlib.scrypt`), never the PIN itself. The commissioner can reset a forgotten PIN.
- **Limited guesses:** there are only 10,000 possible PINs, so after 5 wrong tries a team is locked for 10 minutes. Every failed try is logged.
- **Commissioner password:** long, hashed the same way, and kept in a config file outside the code, never committed to git.
- **Session cookie:** after a correct PIN, the server creates a long random token (`secrets.token_urlsafe`), stores its hash in `sessions`, and sends it as a cookie marked `HttpOnly`, `Secure` and `SameSite=Lax`. Sessions expire after the draft.
- **More than one device is fine:** a manager can be signed in on a laptop and a phone at once.
- **The server decides:** every state-changing request re-checks who the user is and what they're allowed to do. The page never decides on its own.

## Architecture and tech stack

One Python process on this machine serves every page, API call and live update, and stores everything in one SQLite file.

*Diagram (see the doc):* Manager browsers ⇄ (HTTPS) ⇄ Cloudflare ⇄ cloudflared tunnel client ⇄ **FastAPI app** (routes, sessions, live updates), all on this machine. The app calls three internal parts: the **Importer** (reads OnRoto), **Rules** (checks actions) and **SQLite** (`draft.db`).

Managers reach the app through Cloudflare, which forwards traffic down a tunnel the machine opened. So no router ports are opened. The app calls the rules module before accepting any action and writes to SQLite before telling anyone.

**What a web server actually does.** A browser sends an HTTP request: a method (`GET`, `POST`), a path (`/board`), headers (including cookies) and sometimes a body. The server matches the path to a function, runs it, and returns a response: a status code (`200`, `303`, `403`), headers, and a body of HTML or JSON. Everything in this project is a variation on that loop.

| Layer | Choice | Why, for learning |
| --- | --- | --- |
| Language | Python 3.12+ | Readable, great for scraping, one language everywhere |
| Web framework | FastAPI, run by the Uvicorn server | Small, explicit routes. You can see each request's path, method, cookies and body. |
| Pages | Server-rendered HTML with Jinja2 templates | The server builds each page, so no frontend build tools to learn yet |
| Interactivity | A little plain JavaScript | Search-as-you-type and bid buttons without a framework |
| Live updates | Server-Sent Events (SSE) | One long-lived HTTP response the server keeps writing to, and the simplest real-time option |
| Database | SQLite through Python's built-in `sqlite3` | One file, real SQL, no separate database server. Writing raw SQL teaches more than an ORM. |
| Scraping | `httpx` + BeautifulSoup | Fetch OnRoto pages and parse the HTML tables |
| Hosting | systemd + Cloudflare Tunnel | Runs on this machine and is reachable from anywhere |

**Stretch option:** after v1 works, rewrite a small piece with the standard library `http.server`, no framework, to see what FastAPI does for you.

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

## HTTP routes and live updates

About 20 routes cover the whole app. `GET` routes return pages or data, and `POST` routes change something and always go through auth, the rules check and a database write.

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| GET | `/login` | Anyone | The sign-in page: every team as a big button, plus Spectator |
| POST | `/login` | Anyone | Checks the team and PIN (and the commissioner password, if chosen), sets the session cookie, then redirects (`303`) to `/`. Wrong PIN means `401` and a friendly message. |
| POST | `/logout` | Signed in | Ends the session |
| GET | `/` | Signed in | The draft room: current phase, auction or pick panel, board. Signed out means a redirect to `/login`. |
| GET | `/teams/{id}` | Signed in | The Teams page: a dropdown to pick the team, its roster by position with projected stats, money left and players still needed |
| GET | `/players?pos=SS&sort=HR` | Signed in | The Players page: available players with projected stats. ?pos= filters by position, ?sort= sorts by a stat, ?q= searches by name |
| GET | `/players/{id}` | Signed in | One player's popup (HTML fragment): projected stats, plus fantasy team, salary and contract years if drafted |
| GET | `/history` | Signed in | The History page: every auction sale and ULTRA pick in order |
| GET | `/events` | Signed in | SSE stream of live updates |
| POST | `/auction/nominate` | Team whose turn it is | Nominates a player with an opening bid |
| POST | `/auction/bid` | Any team that can afford it | Places a higher bid |
| POST | `/ultra/pick` | Team on the clock | Makes an ULTRA pick |
| GET | `/admin` | Commissioner | Control panel |
| POST | `/admin/import` | Commissioner | Runs the OnRoto import |
| POST | `/admin/start`, `/pause`, `/resume` | Commissioner | Controls the draft |
| POST | `/admin/undo` | Commissioner | Undoes the last action |
| POST | `/admin/act-as/{team}` | Commissioner | Nominates, bids or picks for a team |
| POST | `/admin/pins/{team}` | Commissioner | Generates or resets a team's PIN and unlocks a locked-out team |
| GET | `/export.csv` | Anyone signed in | Draft results download |

**Every POST handler follows the same five steps:**

1. **Who?** Read the session cookie and look up the team. Nobody signed in means `401`.
2. **Allowed?** Ask the rules module whether this team can do this now. If not, return `409` with a plain-English reason shown to the user.
3. **Save.** Append the event in a database transaction.
4. **Apply.** Update the in-memory state.
5. **Broadcast.** Push the event to every open `/events` stream.

A single lock around steps 2–5 means two bids arriving at once are handled one after the other, never interleaved.

**How SSE works.** The browser opens `/events` with `new EventSource('/events')`. The server keeps that response open and writes lines like `event: bid` and `data: {...}` whenever something happens. If the connection drops, the browser reconnects automatically and sends `Last-Event-ID`, so the server can replay what it missed. It's plain HTTP, which makes it easy to watch with `curl -N`.

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

**The timer (a key learning piece)**

- When a bid is accepted, the server stores `deadline = now + N seconds` and broadcasts it.
- Browsers draw the countdown from that deadline. They never decide when the lot is over.
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
- **Review screen:** after import, show each team's roster, total salary and cap room, and flag anything odd (a team over the cap, unparseable rows).
- **Be polite:** about 15 requests per import, one at a time. Never scrape during the draft.
- **Security:** treat the OnRoto password and every session link as passwords. Never commit them or write them to logs.

**Automatic OnRoto login**

The importer gets its own session ID by logging in the way a browser does, so nobody copies links by hand. At the start of Milestone 8, we watch a real login in the DevTools Network tab (with "Preserve log" on) to learn OnRoto's exact login URL and form fields.

1. **Credentials:** the commissioner's OnRoto username and password live in the gitignored `.env` file. They are never stored in SQLite, shown on a page or written to logs.
2. **Load the login form:** `GET` the OnRoto login page and parse its `<form>` with BeautifulSoup: the `action` URL, the field names and any hidden fields. Reading these from the page instead of hard-coding them keeps the login working if OnRoto tweaks the form.
3. **Submit it:** `POST` the username, password and hidden fields from one `httpx.Client`, which keeps any cookies OnRoto sets.
4. **Find the session ID:** take `session_id` from the redirect's `Location` header, or from a link on the page that comes back.
5. **Check it:** fetch one league page and confirm it is real data, not the login form again. A login page returned with `200 OK` counts as a failure.
6. **Use it, then drop it:** the session ID stays in memory for that one import run and is never saved.

If the login fails (wrong password, a changed form, a CAPTCHA), `/admin/import` says so plainly and offers two fallbacks. The commissioner can paste a session link, or open each OnRoto page in a browser, save it, and upload the saved HTML files. Uploaded files go to `imports/` and run through the same parser, so this still works if a bot check (like Cloudflare's "Verify you are human") ever blocks scripted requests. A test on 2026-10-10 got `200 OK`, so for now the scripted fetch works.

## Persistence, recovery and undo

Every action is written to SQLite and committed before any browser is told about it. A crash or reboot loses at most an action that was never confirmed.

- **On startup:** load the settings, replay `events` in `seq` order to rebuild state, and resume. If a lot was mid-auction, it comes back **paused**, so the commissioner resumes it and nobody loses a bid to the downtime.
- **Browsers reconnect by themselves** (SSE retry) and fetch the current state, so managers see a short "Reconnecting…" notice, then carry on.
- **Undo:** the commissioner's undo appends an `undone` event that cancels the latest action. Replay skips cancelled events. Nothing is deleted.
- **Durability settings:** SQLite in WAL mode, with `synchronous=FULL` for commits that survive power loss.
- **Backups:** copy `draft.db` every 5 minutes during the draft with SQLite's backup API, keep the last 50 copies, and put a copy somewhere off the machine after the draft.
- **Export:** `/export.csv` gives results anyone can open in a spreadsheet, and doubles as a paper fallback. Three details make it open cleanly in Excel:
  - Encoded as UTF-8 with a BOM (Python's `utf-8-sig`), so accented names like Acuña aren't garbled.
  - Sent with `Content-Disposition: attachment; filename="gmrl-draft-YEAR.csv"`, so the browser downloads it instead of showing it.
  - Any cell starting with `=`, `+`, `-` or `@` is escaped, so Excel never runs it as a formula.

## Hosting from this machine

The app runs on this Linux machine as a systemd service, and Cloudflare Tunnel makes it reachable at a public HTTPS address. No router changes are needed, and your home IP stays hidden.

1. **The app** listens only on `127.0.0.1:8000`, so nothing on your network can reach it directly.
2. **systemd** (`draft.service`) starts it on boot and restarts it if it crashes. Logs go to `journalctl -u draft`.
3. **cloudflared** (`cloudflared.service`) keeps an outbound tunnel open to Cloudflare, and Cloudflare forwards `draft.<yourdomain>` to `localhost:8000`.
4. **Domain:** about $10 a year, managed in Cloudflare. Tailscale Funnel is a free fallback, but its address is harder to share.

**Draft-day checklist:** wired Ethernet, sleep and suspend disabled, laptop plugged in, a fresh backup, and a test sign-in from a phone on cellular data an hour before.

**Known risk:** a home internet or power outage pauses the draft for everyone. The event log means nothing is lost. If this proves flaky, the same code moves to a $5 cloud server.

## UI and accessibility

The site should be usable by a manager who has never seen it, on a laptop, tablet or phone, without instructions.

**Pages.** The site has four pages that share one layout: a live top bar, then links to Draft · Players · Teams · History. Every page opens the SSE stream, so the top bar stays current wherever you are.

- **Top bar, auction:** the player up for bid, the current high bid and bidder, and the timer.
- **Top bar, ULTRA:** who's on the clock, the round and pick, and the pick timer if it's on.

| Page | Path | Shows |
| --- | --- | --- |
| Draft | `/` | The auction panel (bid buttons, your cap room, max bid, open roster spots) and recent sales, or in ULTRA the pick panel and board. |
| Players | `/players` | Available players with projected stats. Filter by position and sort by any stat, alone or within a position, all in the URL (`/players?pos=SS&sort=HR`). Hitters and pitchers show their own stat columns. On your turn, each row has a Nominate button (auction) or Pick button (ULTRA). Nominate asks for the opening bid, then confirms. |
| Teams | `/teams/{id}` | A dropdown picks the team. Its roster by position with projected stats, money left and players still needed. |
| History | `/history` | Every auction sale with its price and every ULTRA pick with its round and pick, in order. |

**Player popup.** Every player name, on any page, is a link. Clicking it fetches `/players/{id}` and shows it in a popup: projected stats and, if drafted, the fantasy team, salary and contract years. Without JavaScript, the same link opens as a normal page.

- **Readable:** base text 18 px or larger, strong contrast, buttons at least 48 px tall, and no text in faint grey.
- **One job per screen:** the draft room shows what's happening now and what *you* can do. Everything else is one click away.
- **Plain words:** "Bid $23", "You can bid up to $41", "Waiting for Peeking Ducks to nominate". No jargon or icon-only buttons.
- **Forgiving:** a confirm step before nominating and before an ULTRA pick. Clear error messages that say what to do next.
- **Always oriented:** the live top bar on every page shows the phase, who's up, and the time left.
- **Sound (optional):** a chime when it's your turn to nominate or pick.
- **Works everywhere:** a responsive layout and current Chrome, Safari and Edge. No app to install.
- **Test it:** before draft day, run a mock draft with two or three of the least technical managers and watch where they get stuck.

## Learning roadmap

Each milestone adds one web-server idea and ends with something you can run and see. Milestones 1–6 get you a working auction, and the rest make it real.

| # | Build | Web-server idea you learn | Done when |
| --- | --- | --- | --- |
| 0 | Tiny server with Python's `http.server`, no framework | What a request and response really are | `curl -v localhost:8000` shows your headers and body |
| 1 | FastAPI hello world plus a Jinja2 page | Routing, templates, status codes | A browser shows a page listing 12 fake teams |
| 2 | SQLite tables (with fake projections) and the Players page | SQL, query parameters, HTML forms | `/players?pos=SS&sort=HR` lists shortstops by projected HR, and typing "sot" finds matching players |
| 3 | Team list, PIN sign-in and sessions | Cookies, redirects, password hashing, rate limiting, auth checks, `401` / `403` | Two browsers are signed in as two different teams |
| 4 | ULTRA picks with the event log, plus the Teams and History pages | POST handlers, transactions, event sourcing | Picks survive stopping and restarting the server, and show on Teams and History |
| 5 | Live updates over SSE, the shared top bar and the player popup | Long-lived responses, broadcasting to many clients, fetching HTML fragments with JavaScript | A pick in one browser appears in the other instantly, on every page |
| 6 | The auction and its server timer | Background tasks, concurrency, locks | Two browsers bid against each other, and the timer sells the lot |
| 7 | Rules module with tests | Keeping logic separate from HTTP, `pytest` | League rules, including position eligibility ("Position unavailable"), are enforced, with a test for each |
| 8 | OnRoto login and importer | Being an HTTP *client*: form POSTs, cookies, redirects, parsing HTML | One click logs in to OnRoto, and real keepers, contracts and projections load and review cleanly |
| 9 | systemd plus Cloudflare Tunnel | Reverse proxies, TLS, services, logs | A friend on another network signs in and bids |
| 10 | Commissioner tools, polish, mock draft | Admin authorization, UX testing | A full mock draft with real managers works |

**How we'll work:** I write most of the code and explain it as we go. You write pieces yourself when you want the practice, and no code goes in until you understand it. At each milestone, look at the raw HTTP traffic in the browser's DevTools Network tab and with `curl -v`, so the framework never feels like magic.

## Testing

Most tests are fast unit tests of plain Python, a fair number exercise real HTTP requests, and a few checks happen by hand in a real browser. Tests use `pytest`, and a test is the "predict" step of each lesson written down.

- **Unit tests (most):** plain functions with no server or network. They cover the rules (`can_bid` returns OK or a reason such as "Max bid $4" or "Position unavailable"), event replay (events in, draft state out, which also covers crash recovery and undo), the OnRoto parser on saved sample pages, and the CSV export (BOM, escaped `=` cells, an accented name like Acuña).
- **Integration tests (some):** FastAPI's `TestClient` sends real requests straight into the app with a fresh temporary database, and checks status codes, headers and cookies. Examples: a wrong PIN gives `401`; a right PIN gives `303` with `Set-Cookie`; a rule-breaking bid gives `409` with its reason; a pick survives rebuilding the app on the same database file.
- **End-to-end (few, by hand):** a real server with `curl -v`, two browser windows bidding against each other, then the mock draft. Browser automation (Playwright) is a heavy dependency, so we decide on it in Milestone 10.

**Two design choices keep the code testable:**

1. **The rules module never touches HTTP or the database.** It takes the current state and an action and returns a decision, so every league rule can have its own test.
2. **The timer takes its time from a clock we pass in.** Tests use a fake clock and move it forward by hand ("advance 30 seconds, the lot is sold"), so timer tests run instantly and give the same result every time.

| # | What we test | Kind |
| --- | --- | --- |
| 0 | The handler returns the right status line, headers and body | Unit, plus `curl -v` |
| 1 | Routes return `200` / `404`, and the page lists 12 teams | Integration |
| 2 | Filter and sort SQL, and a name in a search can't break the SQL | Unit and integration |
| 3 | PIN hashing, lockout after 5 tries, cookie flags, `401` / `403` | Unit and integration |
| 4 | Replaying events rebuilds the state, undo works, picks survive a restart | Unit and integration |
| 5 | `/events` sends the right `event:` / `data:` lines, and reconnecting with `Last-Event-ID` replays missed events | Integration, plus `curl -N` |
| 6 | The fake-clock timer sells the lot, late bids are rejected, two bids at the same instant are handled one at a time | Unit and integration |
| 7 | Every league rule, including "Position unavailable" | Unit, the biggest set |
| 8 | Parsing saved sample pages (session IDs removed), and spotting a failed login | Unit |
| 9 | The site works from a phone on cellular data | Manual |
| 10 | A full mock draft | Manual |

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
