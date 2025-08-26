# Stress Mora — Chat with Excel via AI + Neon (MVP)

> **Status:** Working MVP. **Deliberately slow** right now (explained below) to keep behavior stable and predictable with messy Excel files. Includes fixes for common ingest and SQL issues.

## What this app does

- You drop an **Excel file** (`data.xlsx`) in the **repo root**.
- On server start, we **read every sheet** and **create a Postgres table per sheet** in **Neon** (no changes to the Excel file).
- The user asks natural-language questions (e.g., “How many patients?”).
- The server uses an **LLM to generate a SQL `SELECT`**, sanitizes it, runs it against Neon, and returns results.

This lets you **chat with your Excel** without manual uploads or schema editing.

---

## Why it’s slow right now (by design)

1. **Large “catalog prompt” to the LLM**  
   We send the model a summary of **all tables and their columns**. This makes the model safer (fewer hallucinations) but the request is **heavier** → slower.

2. **Single model call per question with generous timeout**  
   To maximize reliability across varying network/model conditions, we allow a **long timeout** for the AI call. That increases worst-case time.

3. **No aggressive caching** in the current mode.  
   Every question is computed fresh.

> Result: You may see **10–30+ seconds** on some queries (network/model dependent). The database query itself is usually fast; 99% of the time is the AI call.

---

## How to make it fast (options you can enable later)

Pick any (they stack):

1. **Slim prompt per table (recommended)**  
   Instead of sending the full catalog, **pick one table locally** (heuristic routing from the question) and send only that table’s columns to the model. This can **cut latency dramatically**.

2. **Multiple model fallbacks + short timeouts**  
   Try a list of known-fast model slugs with **8–12s timeouts** each and **auto-fallback** on 404/timeout. Avoids getting stuck on a dead/slow model.

3. **5-minute answer cache**  
   Cache `q → (sql, rows)` for quick repeats and variations.

4. **Local patterns for common intents**  
   For simple “COUNT/AVG by X” questions, build the SQL **locally** and only run the model for more complex queries.

5. **Use a faster model**  
   In `.env`, set a fast model available on your OpenRouter key (e.g. small instruction-tuned models).

> All of these optimizations were provided as an optional fast server variant earlier. You can swap to that when you’re ready.

---

## Project structure (key files)

```
src/
  loadExcel.ts   # Reads Excel, infers schema, creates & populates Neon tables
  server.ts      # Express server: receives questions, calls AI, sanitizes SQL, runs query
public/
  index.html     # Minimal UI for asking questions and viewing results
.env             # Environment variables
data.xlsx        # Your Excel file (root-level, unchanged by the app)
```

---

## Code flow

### 1) Startup

- `src/loadExcel.ts`:
  - Reads **all sheets** via `xlsx` (ESM build `xlsx/xlsx.mjs`).
  - For each sheet:
    - **Detect headers** (or lack thereof).
    - **Normalize column names** (snake_case; semantic hints like `visit_date`, `age`, `sex`, `phone`, `taluka`, `village`, etc when obvious).
    - **Infer types conservatively**: only `double precision` if **all** samples are numeric; only timestamp if **all** samples are date-like; otherwise **text**.
    - **Coerce values safely** at insert time:
      - numeric columns: non-numeric → `NULL` (prevents errors like `"Total"`).
      - date columns: unparsable → `NULL`.
    - **Skip obvious “Total” rows** (mostly empty + `Total` token).
    - **Create one table per sheet** in Postgres and **bulk insert** rows.
      - **Safe table names** (`t_` prefix if the sheet starts with a digit, max 63 chars, unique suffixes).
  - Returns a **catalog**: list of `{tableName, columns, rowCount}`, and selects the **largest table as the base**.

### 2) Asking a question (`/ask`)

- `src/server.ts`:
  - Builds a **catalog prompt** (slow but safer): first N columns per table.
  - Calls the LLM to produce a `SELECT` (no DDL/UPDATE/DELETE/JOIN).
  - **Sanitizes the SQL** (critical fix for parser issues):
    - Strips comments.
    - **Rewrites the entire first `FROM …` segment** to a clean `FROM "<chosen_table>"` (currently the **base table** for stability).
    - Adds `LIMIT 100` automatically for non-aggregates.
  - Runs the query on Neon and returns JSON.

### 3) Diagnostics

- `/health` → `{ ok: true }`
- `/diag` → base table name + row count + list of tables
- `/refresh` → re-reads Excel and reloads all tables

---

## Endpoints

- `GET /health`  
  Simple liveness probe.

- `GET /diag`  
  Returns base table stats and the list of loaded tables (with row counts). Useful to ensure ingestion worked.

- `POST /ask`  
  Body: `{ "q": "How many patients?" }`  
  Response: `{ ok, ms, table, sql, rows, columns }`

- `POST /refresh`  
  Forces a full reload of the Excel into Neon (use after replacing `data.xlsx`).

---

## Installation & setup

1) **Install dependencies**
```bash
npm i
npm i pg xlsx openai
```

2) **Environment variables** (`.env` in repo root)
```
# Neon Postgres
DATABASE_URL=postgres://USER:PASSWORD@HOST/DBNAME?sslmode=require

# OpenRouter (OpenAI-compatible)
OPENROUTER_API_KEY=sk-or-...
OPENAI_BASE_URL=https://openrouter.ai/api/v1
OPENROUTER_MODEL=gpt-oss-20b:free

# Excel path (keep file in repo root; file is never modified)
EXCEL_FILE=./data.xlsx
```

> **Tip (TypeScript + xlsx ESM):** add one file `src/declarations.d.ts` with  
> `declare module "xlsx/xlsx.mjs";`

3) **Place your Excel**  
Copy `data.xlsx` into the **repo root** (same level as `package.json`). No edits needed.

4) **Run**
```bash
npm run dev
# or
npm run build && npm start
```

5) **Open UI**  
Visit `http://localhost:3001` and ask questions.

---

## Example usage

- **“How many patients?”**  
  -> `SELECT COUNT(*) AS total FROM "<base_table>"`  
  (You’ll see a 1-row result with the count.)

- **“Average age by taluka (top 5)”**  
  -> `SELECT taluka, AVG(age) AS avg_age FROM "<base_table>" GROUP BY taluka ORDER BY avg_age DESC LIMIT 5;`

- **“Patients in July 2024”**  
  -> Simple date filter on the base table (if there is a `visit_date`-like column).

---

## Troubleshooting (common errors we handled)

- **`TypeError: XLSX.readFile is not a function`**  
  Use ESM import: `import * as XLSX from "xlsx/xlsx.mjs";` and call `XLSX.set_fs(fs)`. Add `declare module "xlsx/xlsx.mjs";` if TS complains.

- **`22P02: invalid input syntax for type double precision: "Total"`**  
  Fixed by **strict type inference** (mixed → `text`) and **safe coercion** (non-numeric → `NULL`).

- **`42P01: relation ... does not exist` / `42P07: already exists`**  
  Caused by DDL visibility/order or concurrent loads. Use a **single connection + transaction** for `DROP → CREATE → INSERT`, and an **init guard** to avoid double loads.

- **`42601: syntax error near ".."`**  
  LLM produced a malformed `FROM` clause. We **strip/replace the entire `FROM …` segment** with a clean `FROM "<table>"` and add `LIMIT` when needed.

---

## Security & limits (MVP)

- The server **rejects** any non-`SELECT` statements; **no JOINs** in MVP.
- Only the listed tables/columns are intended to be used by the LLM.
- The Excel file is **read-only**; never modified.
- This is an MVP; do not expose publicly without authentication/rate-limits.

---

## Moving to “fast mode” (when you’re ready)

Switch `server.ts` to the **fast variant** (previously provided) or implement these changes:

1. **Heuristic routing** to pick one table from the question.
2. **Single-table prompt** (`buildSystemPromptForTable`) instead of full catalog.
3. **Short AI timeout (8–12s)**, **try multiple model slugs** with fallback.
4. **Add a 5-minute in-memory cache** for answers.
5. Keep the **same SQL sanitizer** (`pinTableAndLimit`) to guarantee clean `FROM` and `LIMIT`.

This typically reduces perceived latency from **~10–30s → ~1–6s**, depending on model and network.

---

## FAQ

**Q: Do I need to edit or reformat my Excel?**  
No. The loader handles missing headers, mixed types, date serials, “Total” rows, and odd sheet names. The Excel remains unchanged.

**Q: Can I ask about specific program sheets (e.g., HTN suspected)?**  
In the slow/stable mode we pin to the **base table** for reliability. In fast mode (or a hybrid), we route to the most relevant table for each question.

**Q: Can I join across sheets?**  
Not in this MVP. We can add a lightweight entity key and enable joins later.

**Q: How do I refresh after updating the Excel file?**  
Replace `data.xlsx` in the root and call `POST /refresh` or restart the server.

---

## License

MVP demo for internal evaluation; adapt as needed for your organization’s policies.
