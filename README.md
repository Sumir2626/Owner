[README.md](https://github.com/user-attachments/files/34124767/README.md)
# Carrier Hire Fiji - website and booking system

A working web application (not a prototype) for a passenger and goods carrier-hire business in Fiji.
Customers pick a pickup and destination on Google Maps, see an approximate price calculated from the
real road distance, and submit a booking **request**. Staff review every request before anything is confirmed.

**Price rule (implemented exactly, in one place - `app/pricing.py`):**
`Final estimated price = (road distance km x 1.1) x 1.6`, steps shown separately, rounded to 2 decimals only at the end.
Multipliers are configurable by an administrator (Admin > Pricing); changes apply to new quotations only.

## 1. Architecture and why

| Part | Choice | Reason |
|---|---|---|
| Web app | Python 3.10+ / Flask, server-rendered pages + small vanilla JS | One small codebase, no build step, cheap to host, fast on mobile data |
| Database | SQLite (WAL mode) | No database server to pay for or maintain; fine for a small/medium fleet. Move to PostgreSQL if you outgrow one server |
| Maps (browser) | Maps JavaScript API + Places API (New) autocomplete + Geocoding | Search, map display, click-to-pick |
| Distance (server) | Routes API `computeRoutes`, DRIVE, ferries avoided | The key never reaches the browser and the browser can never send its own distance or price |
| Email | Any SMTP server | Works with Microsoft 365, Google Workspace, Zoho, Mailgun, etc. |
| Hosting | One small Linux VPS (or any Python host) behind HTTPS | A typical budget option |

**Microsoft 365 alternative (assessed, not used):** a public Microsoft Form cannot embed an interactive Google Map,
call the Routes API, or run a server-side pricing formula. A Forms + SharePoint/Lists + Power Automate design would
need a custom front end and an API/Function in front anyway, so you would still host a web app. The email notifications
here use plain SMTP, so you can send them through your Microsoft 365 mailbox without Power Automate.

## 2. What is built (against the brief)

* Public pages: Home, About, Services, Vehicles, Get a Quote, Book Now, Contact, FAQ, Terms, Privacy, Cancellation/Refunds, booking-status page
* Service choice first; passenger vs goods fields appear dynamically; goods photos (JPEG/PNG/WebP, checked by content, max 4)
* Quote page: From/To autocomplete restricted to Fiji, click-on-map picking, markers, driving route line, distance, calculation steps, disclaimer, **Proceed to Booking** / **Request a Manual Quote**
* One-way / return; return with a different route is priced as two separately calculated legs
* Fiji validation (including the 180th-meridian islands); no route / ferry / API error / missing key => **no price**, manual quote offered
* Booking: server validation, unique reference (`CH-YYMMDD-XXXXX`), status `Pending Review`, acknowledgement page. Nothing is auto-confirmed
* Statuses: Pending Review, Quotation Sent, Confirmed, Vehicle/Driver Assigned, In Progress, Completed, Cancelled, Rejected (enforced transitions)
* Admin: search/filter, full detail, open route in Google Maps, original estimate kept separate from final agreed quotation, reason for adjustment, expiry, accept/reject/expire, email quotation, reschedule, cancel, vehicle + driver assignment with clash/capacity/availability checks (admin-only override with a recorded reason), internal notes, change history, payment *status* only (no card data), pricing versions, business settings, users/roles, reports, CSV/Excel export, retention (anonymise old bookings)
* Customer status page (reference + email): see status, accept/decline a quotation, request cancel/reschedule (admin is notified; nothing changes until staff act)
* Emails (customer + admin) for every event in the brief; estimates and final quotations are labelled differently

Nothing is invented: business name, phone, email, logo, vehicles, prices, About text, legal text, FAQ and testimonials are
all blank/placeholder until entered in **Admin > Business**. Add real vehicles only; none are pre-loaded.

## 3. Setup (local)

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # then edit (see section 4)
flask --app run create-admin    # prompts for email, name, password (12+ characters)
flask --app run run --debug     # http://127.0.0.1:5000
python -m pytest tests          # 73 automated tests
```
Admin sign-in: `/admin/login`. First tasks: Business (name, phone, WhatsApp, email, texts) -> Vehicles/Drivers -> Pricing check.

## 4. Environment configuration (`.env`)

See `.env.example` (every variable is commented). Key points:
`APP_ENV=production` and a long random `SECRET_KEY` are **required** live; `SITE_URL` must be your https address;
set `TRUSTED_PROXY_COUNT=1` behind nginx/Cloudflare; `GOOGLE_MAPS_BROWSER_KEY` and `GOOGLE_MAPS_SERVER_KEY` are two *different* keys;
`SMTP_*` and `MAIL_FROM` enable real email (without `SMTP_HOST`, emails are only recorded in Admin > Emails and **not delivered**).

## 5. Google Maps Platform - setup, security and cost

1. Google Cloud Console -> create a project -> **link a billing account** (Google requires one even when usage stays inside free allowances).
2. Enable: **Maps JavaScript API**, **Places API (New)**, **Geocoding API**, **Routes API**.
3. Create **two** API keys:
   * Browser key -> *Website restriction* (HTTP referrers): `https://yourdomain.fj/*` and `https://www.yourdomain.fj/*`; *API restriction*: Maps JavaScript, Places (New), Geocoding only.
   * Server key -> *IP restriction* (your server's IP); *API restriction*: Routes API only. Never put it in front-end code.
4. Create a **Map ID** (Map Management) and put it in `GOOGLE_MAP_ID` (`DEMO_MAP_ID` is for testing only).
5. **Quotas and alerts:** APIs & Services -> Quotas: cap Routes API and Places requests per day; Billing -> Budgets & alerts: add alerts at e.g. 50/90/100% of a monthly budget; consider a hard cap via a Pub/Sub budget action.
6. **Costs:** Google bills per request for map loads, autocomplete/place details, geocoding and each route calculation, and rates and free allowances change. Check current figures at <https://mapsplatform.google.com/pricing/> and use its calculator with your expected quotes per month. The app limits spend by: 40 quote requests per IP per 10 minutes, requesting only the fields it needs, and TRAFFIC_UNAWARE routing. Other external costs: hosting (VPS), domain, SMTP provider (often free at low volume), optional Cloudflare (free tier).

## 6. Deployment (Linux VPS, nginx, HTTPS)

```bash
# on the server, as a non-root user, in /srv/carrier
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp .env.example .env && nano .env && chmod 600 .env
.venv/bin/flask --app run create-admin
.venv/bin/gunicorn -w 3 -b 127.0.0.1:8000 run:app      # run under systemd (Restart=always)
```
nginx: `proxy_pass http://127.0.0.1:8000;` with `proxy_set_header Host $host; X-Forwarded-For $remote_addr; X-Forwarded-Proto $scheme;`,
`client_max_body_size 30m;`, and a free Let's Encrypt certificate (`certbot --nginx`). Redirect HTTP -> HTTPS (the app sends HSTS in production).
**Backups:** copy `instance/carrier.db` (use `sqlite3 instance/carrier.db ".backup backup.db"`) and `instance/uploads/` daily, off the server.
**Retention:** schedule `flask --app run purge-old` (e.g. weekly cron); the retention period is set in Admin > Business.

## 7. Security notes

Server-side validation of every field; prices only ever come from the server (a signed, time-limited token carries the route
distance; client-sent prices/distances are ignored - tested); CSRF tokens on all POSTs; hashed passwords (12+ chars), lockout after 5 failures,
rate limits (login, quotes, bookings, enquiries, status lookups); honeypot + minimum-fill-time spam traps; uploads checked by file signature and stored
outside the web root, served only to signed-in staff; no personal data in URLs (booking lookup needs reference + email); strict Content-Security-Policy
(the quote page uses Google's recommended nonce-based policy); CSV/Excel export neutralises formula injection; no card data is stored.
Staff accounts have role `staff` (bookings, vehicles, drivers) or `admin` (also pricing, business settings, users, payment status, export, overrides).
Consider adding Cloudflare Turnstile/reCAPTCHA in front of the booking form if you receive spam.

## 8. Known limits - please read

* **Live Google calls were not exercised here** (no key/network in the build environment). Maps and Routes are tested with a stub and a fake Routes client. Run the production checklist below with your real keys before launch.
* Routes uses `avoidFerries`; where the only connection is a ferry (e.g. between islands) Google returns no road route and the customer is offered a manual quote. A remote place with no mapped road may be snapped to the nearest road, so its distance can be understated - staff review every estimate before confirming.
* Google's `PlaceAutocompleteElement` event/name details change between releases; the code listens to both current and older event names. Re-test after major Maps API releases.
* SQLite suits one server. Terms, privacy and cancellation text are blank - have a lawyer write them.
* Emails are sent synchronously with a 15 s timeout; a mail failure is logged and never blocks a booking.

## 9. Production test checklist (tick each on the live site, with real keys)

1. [ ] https works, http redirects, no browser warnings; `/admin` needs sign-in
2. [ ] Quote page: map loads centred on Fiji; searching "Suva" and "Nadi" shows suggestions only in Fiji; clicking the map sets the active point
3. [ ] Two Fiji points give a km distance close to Google Maps' driving distance; route line appears; "Open in Google Maps" opens the same route
4. [ ] Calculation shows `km x 1.1 = a`, then `a x 1.6 = FJ$ total`; compare with a hand calculation (100 km -> 110 -> FJ$176.00)
5. [ ] Changing either location recalculates; return trip doubles; "different return route" prices two legs
6. [ ] A ferry-only pair (e.g. Viti Levu to Taveuni), or a point outside Fiji -> no price, helpful message, manual quote works
7. [ ] Turn off the server key temporarily -> no price, manual quote still works (then restore)
8. [ ] Submit passenger and goods bookings (with a photo); unique references; confirmation page; customer + admin emails arrive (check spam folder, SPF/DKIM for your domain)
9. [ ] Booking appears in Admin; set final quotation with a reason; send quotation; accept it on the status page; confirm; assign vehicle (unchecked vehicle blocked, overlapping booking blocked); reschedule; cancel; complete - emails arrive each time
10. [ ] Staff account cannot open Pricing/Business/Users/export; anonymous POSTs to admin URLs are refused
11. [ ] Change multipliers in Admin > Pricing: new quote changes, an older booking keeps its original numbers
12. [ ] Phone and desktop layouts; keyboard-only booking; Google Cloud quotas and budget alerts visible; browser key referrer restriction rejects another domain
13. [ ] Backups run and a restore is tested; `purge-old` scheduled
