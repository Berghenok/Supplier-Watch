# Supplier Watch 

This includes CRUD APIs, manual website checks, snapshots, deterministic change events with evidence, structured AI analysis, a transparent supplier risk status, a server-rendered dashboard, provider-neutral purchase history, an internal alert inbox, optional email delivery, and guided customer onboarding. CSV is the first purchase-data adapter. Website checks remain manual.

## Project structure

```text
app/
  config.py       environment settings
  database.py     SQLAlchemy engine and session factory
  main.py         FastAPI app and health endpoints
  api.py          company, supplier, and source routes
  models.py       SQLAlchemy tables and relationships
  schemas.py      request and response validation
  monitoring.py   safe fetching, text normalization, and snapshot creation
  acquisition.py  source-type adapter boundary for canonical acquired text
  change_detection.py  deterministic word-level differences and deduplication
  ai_analysis.py  structured OpenAI interpretation of new events
  risk.py         deterministic supplier risk rules
  alerts.py       alert rules and duplicate prevention
  alert_api.py    alert list and read/unread API endpoints
  email_notifications.py  capture/SMTP delivery from saved alerts
  notification_jobs.py    periodic time-based alert evaluation
  onboarding.py  guided company, import, review, importance, and email setup
  i18n.py        central Norwegian Bokmål UI translation catalog
  localized_templates.py  localized dashboard rendering
  dashboard.py    dashboard pages and form actions
  purchases.py    validated CSV import and purchase profile calculation
  purchase_format.py  provider-neutral StandardPurchase schema
  purchase_api.py purchase import and profile API endpoints
  templates/      dashboard HTML templates
  static/         dashboard CSS
alembic/          database migration setup and revisions
examples/         sample purchase CSV
tests/            API and database setup tests
compose.yaml      local PostgreSQL and API
Dockerfile        Python runtime for the API
```


## Use the dashboard

1. Open <http://localhost:8000/dashboard> in your browser. Create a company if the page asks for one. If you already have several companies, use the selector at the top to switch between them.
2. Add a supplier with its importance level. Open the supplier and add a public monitored page URL.
3. Select **Check now** to save the first snapshot. Run it again after the page changes. The supplier page then shows detected events and the current GREEN, YELLOW, or RED status.
4. Open an event to compare previous and current text. **Fact · Source evidence**, **AI interpretation**, and **Recommendation** have separate sections. If AI analysis is unavailable, the page says that it is pending.
5. Remove a monitored URL from the supplier page when you no longer want to track it. This also removes that URL's saved snapshots and events.

The dashboard uses the same database and business rules as the API. It is intended for local use at this stage; authentication is not implemented. Checks are manual in this phase. To allow live AI analysis, add your key as described below.

## Import purchase history

In the dashboard, select a company and open **Purchase profile**. Upload [examples/sample_purchases.csv](examples/sample_purchases.csv) from this project. The sample has two suppliers and five illustrative purchases. Northline Electrical's switch module rises from GBP 100 to GBP 120, so the page shows a 20% price change with both purchase rows as evidence. It also shows latest and previous prices, total quantity, average days between purchase dates, and spend by currency. The sample values are examples, not live supplier data.

Your CSV must be UTF-8 with this header (the order may vary):

```csv
supplier,product_name,product_number,quantity,unit_price,currency,purchase_date
Northline Electrical,Switch module,SW-100,4,100.00,GBP,2026-01-10
Northline Electrical,Switch module,SW-100,6,120.00,GBP,2026-03-10
```

Use a decimal point for quantity and unit price, a three-letter currency code, and dates in `YYYY-MM-DD` format. All seven fields are required. The limit is 5,000 rows or 2 MB. Every row is validated before the import is saved; a failed import saves nothing. Uploading the exact same file again for the same company adds no purchases. Suppliers are matched by name ignoring case and extra whitespace; products are matched by supplier and product number. A new supplier or product is created automatically when no match exists.

Price changes are deterministic: a change of at least 5% between consecutive purchases of the same product and currency is shown. The purchase records, CSV filename, row numbers, and file fingerprints remain attached as evidence. Imported prices describe what your company paid; they do not claim to be current supplier quotes. Purchase changes are separate from website events and do not alter supplier risk status. Website monitoring remains available.

The profile also lists frequently purchased products by purchase-row count, suppliers ordered by configured importance then purchasing frequency, and each product's dated price history. For a significant price change, it estimates financial impact as **unit-price difference × quantity purchased in the preceding 365 days**. The result is signed (a decrease is negative), shown in the purchase currency, and omitted if there is no prior-year purchase volume. It is an illustrative estimate, not a forecast. Spend is never summed across currencies.

### Standard purchase-data flow

Every input adapter maps its data to `StandardPurchase` in `app/purchase_format.py`: supplier, product name, product number/SKU, decimal quantity and unit price, currency, purchase date, provider, source reference, and stable source record ID. A connector passes a batch of those validated records to `ingest_purchase_batch()` in `app/purchases.py` with a stable batch ID and source label. CSV already uses this path. The ingestion service matches suppliers and products, saves original purchase lines with source identity, skips repeated records, and recalculates deterministic price changes. Existing source IDs with changed values are rejected so historical evidence is not silently overwritten. The API and dashboard read the same profile.

A future ERP or accounting adapter needs authorized access and must supply one stable, account-aware ID per purchase line (for example account + invoice + line number), a stable ID for each immutable batch or page, a human-readable invoice/export reference, and the seven business fields above. It must map provider units, VAT basis, currencies, dates, and corrections explicitly before ingestion. No provider credentials or provider-specific API code are included in this phase. CSV files have no transaction ID, so an identical file is deduplicated by its content hash; a modified CSV may still contain repeated purchases and should be reviewed before upload.

The same data can be inspected through `POST /companies/{company_id}/purchases/import` and `GET /companies/{company_id}/purchases/profile` in `/docs`.

## Alerts

Select a company in the dashboard and open **Alerts** in the sidebar. Filter by unread/read status and alert type, and mark alerts read or unread. Each alert shows its factual evidence, any AI interpretation, a recommended action, and an estimated financial impact when prior purchase volume is available. Alerts are stored in PostgreSQL and deduplicated by their underlying event, purchase record, or source-health episode. Source-health alerts are operational warnings; they do not change supplier risk.

To try each alert type:

1. **RED risk:** Configure `OPENAI_API_KEY` in `.env`, add a HIGH or CRITICAL supplier and a permitted webpage URL, then check it twice with a meaningful change. If the AI marks the event relevant with HIGH severity and at least 0.70 confidence, the deterministic risk rule turns RED and creates an alert. The alert keeps the original page evidence separate from the AI summary and reason. A repeated unchanged check adds no alert.
2. **Price increase:** Open **Purchase profile** and import [examples/sample_purchases.csv](examples/sample_purchases.csv). Its switch-module purchase rises from GBP 100 to GBP 120 and creates a price alert with both purchase rows and an annualized impact estimate based on prior 365-day quantity. Reimporting the same file adds no alert. Price decreases do not trigger alerts.
3. **Source health:** Check an active URL that returns HTTP 401 or 403. The source becomes `ACCESS_BLOCKED`; if it remains blocked for 24 hours, a source-health alert is created. A previously successful source becomes `STALE` after seven days without a successful check. Failed checks preserve the last snapshot and never create supplier risk events. To verify both time thresholds immediately with controlled data, run `docker compose exec -T api pytest -q tests/test_alerts.py`.

The alert list and read/unread actions are also available under `/companies/{company_id}/alerts` in <http://localhost:8000/docs>. Source-health thresholds are evaluated when the Alerts page or API is opened, or on a subsequent blocked check; a small periodic job also evaluates them every five minutes for companies with email enabled.

## Email notifications

In **Alerts**, enter an alert email address, check **Enable email notifications**, and save. Email is disabled for every new company until configured. The same settings are available through `GET` and `PUT /companies/{company_id}/email-settings` in `/docs`. Enabling notifications sends any existing alerts that have not already been delivered, as well as future ones.

The default `EMAIL_BACKEND=capture` stores a preview and `CAPTURED` status in the database without contacting an email server. Expand **View captured email** on an alert to inspect the message and its link. The message includes the affected supplier/product, severity, concise change, recommended action, estimated financial impact when available, and a link to that alert. A RED risk email includes the saved AI summary separately. To exercise all three alert types and delivery failures without waiting or spending API credits, run:

```powershell
docker compose exec -T api pytest -q tests/test_email_notifications.py
```

For real delivery, edit `.env` and set `EMAIL_BACKEND=smtp`, `SMTP_HOST`, `SMTP_PORT` (usually `587`), `SMTP_FROM_EMAIL`, `SMTP_USERNAME`, `SMTP_PASSWORD`, and `APP_BASE_URL` to the address users can open. Then run `docker compose up -d --build api`. SMTP uses STARTTLS; credentials are never saved in the database. Keep `.env` private. The provider boundary is `EmailSender` in `app/email_notifications.py`, so another transport can be added later.

Each alert has at most one automatic email delivery record. Status and failures appear on the Alerts page and at `GET /companies/{company_id}/alerts/{alert_id}/email`. `SENT` and `CAPTURED` emails are never resent automatically. Failed or interrupted attempts are not retried automatically because SMTP acceptance can be ambiguous; a `FAILED` email can be retried explicitly from the dashboard or `POST /companies/{company_id}/alerts/{alert_id}/email/retry`. `PENDING` after an interrupted process requires manual review. The periodic job runs while the API is running; it does not fetch supplier websites.

In `/docs`, expand **companies** and use `POST /companies` with `{"name":"Example Ltd"}`. Copy the returned `id` into `POST /companies/{company_id}/suppliers` with `{"name":"Parts Supplier","website":"https://example.com","importance":"HIGH"}`. Copy that supplier's `id` into `POST /companies/{company_id}/suppliers/{supplier_id}/sources` with `{"url":"https://example.com/catalog"}`. The GET, PATCH, and DELETE routes manage each record. Only HTTP and HTTPS URLs are accepted.

## Check a supplier webpage

In `/docs`, create a company, supplier, and source as above, using the supplier's real public page URL. Then use `POST /companies/{company_id}/suppliers/{supplier_id}/sources/{source_id}/check` with the three returned IDs. The first success returns `BASELINE` and a snapshot. Repeating the check returns `UNCHANGED` if the normalized text is identical, or `CHANGED` with a new snapshot if it differs. Use the source's `/snapshots` GET route to inspect saved text and hashes.

When a check returns `CHANGED`, `events_created` shows how many substantive text changes were saved. Use `GET /companies/{company_id}/suppliers/{supplier_id}/events` to see the events and their evidence. Each evidence record includes the previous and current text excerpts, the exact removed and added words, the source URL, and IDs of both full snapshots. Case-only, punctuation-only, and common update-date changes are filtered. The same local text change on the same URL is recorded once, even if the page later changes back and repeats it. A changed page can still have `events_created: 0` when only filtered noise changed.


## Supplier risk status

Use `GET /companies/{company_id}/suppliers/{supplier_id}/risk` in `/docs`. It returns `GREEN`, `YELLOW`, or `RED`, plus the event IDs, summaries, points, and rule explanations causing a YELLOW or RED result. It also returns `pending_events`, the number of detected events without an AI analysis. `GREEN` means no analyzed event currently reaches the YELLOW threshold; it is not a guarantee that the supplier has no problems.

The risk rules use saved analysis as input, but the final color is calculated in Python:

1. Ignore an analysis marked `relevant: false` or with confidence below `0.70`.
2. Assign severity points: LOW `1`, MEDIUM `2`, HIGH `3`.
3. Add supplier importance points: LOW or MEDIUM `0`, HIGH `1`, CRITICAL `2`.
4. An event scores GREEN at `0–1`, YELLOW at `2–3`, and RED at `4+` points. The supplier takes the highest event color. Only events reaching that highest color appear in `causing_events`.

The status is calculated when requested, so changing supplier importance or adding an analysis updates it immediately. There is no risk score table to keep in sync. The OpenAI model is already configurable through `OPENAI_MODEL` in `.env`; change it there and run `docker compose up -d --build api` to apply the setting.

Checks only fetch active suppliers and sources. Only public HTTP(S) sites on ports 80 and 443 are allowed. Redirects are rejected, so enter the final page URL (usually HTTPS). Requests have a 10-second connection/read timeout and a 2 MB response limit. Only HTML and plain text are accepted. A failed attempt updates `last_checked_at` but creates no snapshot. JavaScript is never executed; pages that render all content in JavaScript may have little extractable text.

## Source health

The supplier dashboard shows each URL's source health, last attempt, last successful check, and latest acquisition error. The source API returns the same fields. `UNCHECKED` means no attempt has succeeded yet. A successful baseline, unchanged check, or changed check is `HEALTHY`. HTTP 401/403 is `ACCESS_BLOCKED`, HTTP 429 is `RATE_LIMITED`, and other request, response, or URL failures are `FETCH_FAILED`. A healthy source becomes `STALE` if its last successful check was more than seven days ago. A failed check retains its specific failure status, even if its last successful check is old.

Acquisition failures update source health only. They preserve the most recent successful snapshot and create no supplier event, AI analysis, or risk change. Supplier risk still reflects the saved evidence and analyses; source health is a separate operational signal. After a later successful check, the error clears and health returns to `HEALTHY`. A 403 should be resolved through permitted supplier access or an authorized integration, not by bypassing access controls.

The `SourceAcquirer` interface in `app/acquisition.py` routes by source type and returns canonical text. The existing webpage adapter uses the safe HTTP fetcher. A future official API, feed, or authorized integration can register another adapter that produces stable canonical text and typed acquisition failures; snapshot hashing, change detection, evidence, AI analysis, and risk scoring continue through the same pipeline. No new provider connector is included here.


## New customer setup

Open <http://localhost:8000/dashboard> and select **Start setup**. The guide walks you through:

1. Entering the business name.
2. Uploading a purchase-history CSV. Supplier Watch validates it and identifies suppliers and products automatically. The guide links to an example CSV and explains the required columns.
3. Reviewing the detected suppliers, products, latest prices, and meaningful price changes. The original purchase rows remain available in the purchase profile.
4. Setting LOW, MEDIUM, HIGH, or CRITICAL importance for each supplier.
5. Entering an alert email address. The completion screen links to the dashboard and Alerts page.

Select **Explore sample walkthrough** before setup to see the bundled sample purchases and an illustrative price increase. This preview does not create a company, import records, or monitor a website. Existing companies with previously imported purchases can resume the guide and use **I already imported purchases**.

A purchase-price increase can create an alert immediately after CSV import. Once email is enabled, earlier undelivered alerts are also processed. The default `capture` email backend displays message previews inside Alerts; it does not send messages to an inbox. For a real pilot, the operator must configure SMTP as described under **Email notifications** below and set `APP_BASE_URL` to the address customers can open. For website change alerts, add a permitted supplier URL on its supplier page and run a first check to save a baseline; later checks are manual. The operator must configure `OPENAI_API_KEY` for AI analysis of those changes and RED risk emails. Source-health checks run periodically for companies with email enabled, but the job does not fetch websites.

Run the guided workflow tests with `docker compose exec -T api pytest -q tests/test_onboarding.py`.

