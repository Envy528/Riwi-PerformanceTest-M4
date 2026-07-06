# [PROJECT_NAME] — Data Cleaning & PostgreSQL Performance Exam

This project takes a **dirty Excel dataset** (`[DATASET_NAME]_SUCIO.xlsx`), cleans and standardizes it, models it as a **star schema** (dimension + fact tables), migrates it into **PostgreSQL**, and exercises DDL, DML, DQL, DCL and TCL through a series of queries, subqueries, views, permissions and transactions.

---

## Technologies used

- PostgreSQL
- pgAdmin 4 (Query Tool)
- Google Sheets (data cleaning stage)
- SQL: DDL, DML, DQL, DCL, TCL

---

## Installation

1. Install PostgreSQL (and pgAdmin 4, if not bundled with your installer).
2. Open pgAdmin and connect to your local server.
3. Create the database from the Query Tool (connected to `postgres`):

```sql
CREATE DATABASE [database_name];
```

4. Refresh the server tree, select `[database_name]`, and open a **new** Query Tool on that specific database (not on `postgres`) before running anything else.

---

## Running the Project

The full process is documented inside `script.sql`, in this order:

1. **Schema creation** — `CREATE TABLE` statements for all dimension and fact tables, with `PRIMARY KEY`, `FOREIGN KEY ... ON DELETE ...`, `NOT NULL`, `UNIQUE` and `CHECK` constraints.
2. **Data import** — `COPY` statements loading each cleaned `.csv` (exported from the Google Sheets workbook) into its corresponding table.

```sql
COPY [dimension_table] ([col1], [col2], [col3])
FROM '/path/to/[dimension_table].csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',', ENCODING 'UTF8');
```

3. **Queries** — `SELECT` statements covering joins, aggregations, `HAVING`, subqueries (`IN`, `EXISTS`, correlated and scalar), and indicators (`ROUND`, `SUM`, `MAX`, `MIN`, `COUNT`).
4. **View** — a curated `CREATE VIEW` exposing only the columns needed for reporting.
5. **Access control** — a read-only role created with `GRANT`/`REVOKE` over the view and base tables.
6. **Transaction demo** — `BEGIN` / `SAVEPOINT` / `ROLLBACK TO` / `COMMIT` simulating a correction to a stored value.

To run it end to end, open `script.sql` in pgAdmin's Query Tool (connected to `[database_name]`) and execute section by section, or all at once if no manual `.csv` import step is required.

---

## Project Structure

```txt
├── README.md
├── script.sql
├── data/
│   ├── [dimension_1].csv
│   ├── [dimension_2].csv
│   ├── [dimension_3].csv
│   ├── [fact_1].csv
│   └── [fact_2].csv
└── [DATASET_NAME]_SUCIO.xlsx   (original + cleaning work, for reference)
```

---

## Data Model

### Dimension tables
Stable, descriptive attributes of each real-world entity — deduplicated, one row per entity, natural keys preserved from the source spreadsheet (not auto-generated), so fact tables can reference them directly.

- **[dimension_1]** — e.g. product catalog (name, category, unit of measure)
- **[dimension_2]** — e.g. supplier/customer directory (contact info, location)
- **[dimension_3]** — e.g. location/warehouse directory (location, person in charge)

### Fact tables
Transactional records — one row per event, holding the measures that change every time (price, quantity, date, resulting balance/stock), referencing dimensions via foreign key.

- **[fact_1]** — one row per transaction (e.g. purchase, sale, enrollment)
- **[fact_2]** — one row per related event, if the raw data mixes two distinct business processes (e.g. inventory movement triggered by a transaction; 1-to-1 or 1-to-many with `[fact_1]`, depending on the case)

---

## Technical decisions

- **Star schema over a single flat table**: the raw spreadsheet mixed [entity]/[entity]/[entity] and transactional data in one sheet, violating normalization (transitive dependencies, repeated groups). Splitting it into dimensions + facts removes redundancy and matches how the data is actually used (lookup vs. transaction).
- **Natural keys, not `GENERATED ALWAYS AS IDENTITY`, on dimension tables**: the source data already carries meaningful IDs (e.g. `[id_column] = 101...`) that the fact tables reference. Auto-generated identities would renumber them and break every foreign key relationship on import.
- **`NUMERIC(p,s)` instead of `FLOAT`/`double precision` for monetary columns**: floating-point binary types introduce rounding error unacceptable for currency; `NUMERIC` is exact.
- **`ON DELETE RESTRICT` on `[fact_1]` foreign keys**: a [dimension entity] with existing transactional history should not be silently deletable — that would erase real evidence. Deactivation (a status flag) is preferred over deletion in a real system. (Adjust to `CASCADE`/`SET NULL` per table if the business logic calls for it — justify whichever is chosen.)
- **`ON DELETE CASCADE` between `[fact_2]` and `[fact_1]`**, if applicable: a dependent event only exists because of the transaction that triggered it, so it has no independent meaning if that record is removed.
- **Date standardization to `YYYY-MM-DD` before import**: the raw column mixed multiple date formats (ISO, `MM-DD-YYYY`, `DD-MM-YYYY`, and month-name text). Each inconsistency was resolved with a documented rule (locale mismatch fixed by substitution, numeric ambiguity resolved when one segment could only be a day) rather than guessed row by row. Where truly ambiguous, state the assumed convention explicitly.
- **Placeholder date (e.g. `1600-01-01`) for missing dates**: makes missing values explicit and filterable via `WHERE`, instead of silently guessing a date, pending confirmation from the original source document.
