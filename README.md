# Uber Ride Analytics Platform

A streaming data platform on Azure Databricks that models Uber ride operations end to end. A live
booking application emits ride events to Event Hubs, static reference data and historical rides are
attached from ADLS Gen2, and a declarative pipeline transforms both into a star schema with slowly
changing dimensions. The wide join that feeds the model is generated from metadata rather than
hand-written SQL.

---

## Source system: a booking app, not a CSV

The upstream system is a real application. `api.py` serves a FastAPI booking page; requesting a
ride calls `data.py`, which builds a full ride confirmation with Faker — passenger, driver, vehicle,
pickup and dropoff coordinates, distance, duration and a fare derived from base rate, per-mile and
per-minute components with a surge multiplier applied — and `connection.py` publishes it to Azure
Event Hubs.

Standing up a producing source is normally the tedious part of a streaming project, so most skip it
and replay a static file. That quietly removes the thing that makes streaming a design problem: a
file arrives all at once, in order, exactly once. An application does not.

Building the producer mattered here in three ways:

- **Events arrive when they arrive**, which is what makes watermarking and late-data handling a real
  decision rather than a label on a batch read.
- **Every ride carries foreign keys** — `vehicle_type_id`, `payment_method_id`, `pickup_city_id`,
  `cancellation_reason_id` — so the dimensional model is derived from real keys instead of parsed
  out of denormalised text.
- **Reference data lives outside the event**, so the pipeline has to join a fast stream against slow
  lookup tables, which is where the interesting failure modes are.

---

## Two ingestion paths, and why

Not all source data changes at the same rate, and treating it as though it does burns compute for
nothing.

**Ride events, via Event Hubs.** Rides are continuous and unbounded, so they follow the event
stream. `ingest.py` reads the hub through its Kafka-compatible endpoint into `rides_raw`, keeping
the payload as a raw string. `maxOffsetsPerTrigger` caps how much a single trigger pulls,
`startingOffsets: earliest` makes a rebuild replay the full retention window, and
`failOnDataLoss: true` turns a silently skipped offset into a failure rather than a quiet gap.

**Reference and history, via ADLS Gen2.** The six mapping tables — cities, vehicle makes, vehicle
types, payment methods, ride statuses, cancellation reasons — change on the order of months, and the
historical ride backfill is written once and never again. Running either through the stream would
mean repeatedly re-processing data that had not meaningfully changed. Instead `bronze_adls.ipynb`
reads them from blob storage with a scoped SAS token and lands them as Delta tables. The bulk load
is guarded by a `tableExists` check, so re-running the notebook is a no-op rather than a duplicate.

This is the decision the architecture is built around: match the ingestion mechanism to how often
the data actually changes.

---

## Pipeline layers

**Bronze** — `rides_raw` holds the Event Hubs payload untransformed, one row per event, value cast
to string and nothing else. The `map_*` tables and `bulk_rides` land alongside it. Nothing here is
parsed, so a schema mistake costs a re-run of the pipeline, not a re-extraction from the source.

**Silver, staging** — `stg_rides` is a streaming table with two append flows writing into it.
`rides_bulk` reads the historical load and casts `booking_timestamp` to a real timestamp;
`rides_stream` parses the raw JSON against an explicit `StructType` and flattens it. Both land in
the same table with the same shape, so everything downstream sees one continuous ride history rather
than a batch table and a stream table that have to be reconciled at query time.

**Silver, one big table** — `silver_obt.sql` joins `stg_rides` to all six mapping tables and
resolves every foreign key to its label: `vehicle_make`, `vehicle_type` and its rate card,
`ride_status`, `payment_method`, `pickup_city` with state and region, `cancellation_reason`. The
stream side declares `WATERMARK booking_timestamp DELAY OF INTERVAL 5 MINUTES`, which bounds how
long state is held for late arrivals. The joins are `LEFT`, so an event with an unrecognised key
survives with nulls instead of disappearing. This is the conformed view: one definition per business
concept, and one place to validate.

**Gold** — `model.py` carves the wide table back into a star schema. Each dimension is a
deduplicated view over `silver_obt` fed into `create_auto_cdc_flow`: `dim_passenger`, `dim_driver`,
`dim_vehicle`, `dim_payment` and `dim_booking` as Type 1, `dim_location` as Type 2. `fact` sits on
the ride grain, carrying the measures — distance, duration, the fare components, surge multiplier,
tip, rating — and the foreign keys back to each dimension.

**History where it matters.** `dim_location` is the one dimension that changes meaningfully: a city
can be renamed or reassigned to a different region, and the mapping table carries `updated_at` to
say when. Sequencing by that column and storing Type 2 keeps the old row with validity timestamps
instead of overwriting it, so a ride joins to the city as it was when the ride happened. Current-state
queries filter on `__END_AT IS NULL`. The remaining dimensions describe entities whose prior values
carry no analytical meaning, so they overwrite.

---

## Metadata-driven SQL

The OBT join is not hand-written. `silver_obt.ipynb` holds `jinja_config`, a list where each entry
declares one table, the columns it contributes, its join condition and an optional filter. A Jinja2
template walks that list to render the `SELECT`, the `FROM`, the chain of `LEFT JOIN`s and the
`WHERE`, and `silver_obt.sql` is the rendered output.

The point is what a change costs. Adding a seventh lookup table is one dictionary appended to a
list, not a forty-column `SELECT` edited by hand with a new join spliced into the middle of it — the
edit most likely to silently drop a column or duplicate rows. The join logic is written once and
applies to every entry, so a fix to the join pattern is a fix everywhere at once.

---

## Technology stack

| Layer | Technology |
|-------|-----------|
| Event producer | FastAPI, Jinja2 templates, Faker |
| Event ingestion | Azure Event Hubs, Kafka protocol over SASL_SSL |
| Object storage | Azure Data Lake Storage Gen2, SAS-scoped reads |
| Lakehouse | Azure Databricks, Delta Lake, Unity Catalog |
| Stream processing | Spark Structured Streaming, PySpark |
| Pipeline framework | Lakeflow Declarative Pipelines (`pyspark.pipelines`) |
| Change capture | `create_auto_cdc_flow`, SCD Type 1 and Type 2 |
| SQL generation | Jinja2 |
| Tooling | Python 3.12, `uv` |

---

## Running the producer

The Databricks side of the project (ingestion notebooks, pipeline, model) runs in a workspace against
your own Event Hubs namespace and ADLS Gen2 account. What you can run locally is the booking
application that produces the events.

Dependencies are managed with `uv` against the pins in `pyproject.toml`, on Python 3.12:

```bash
git clone https://github.com/nandeshkanagaraju/Uber_Data_Engineering.git
cd Uber_Data_Engineering

brew install uv          # or: curl -LsSf https://astral.sh/uv/install.sh | sh
uv sync
source .venv/bin/activate
```

`connection.py` reads its Event Hubs credentials from the environment, so create a `.env` in the
project root before starting the app:

```
CONNECTION_STRING=<event hubs namespace connection string>
EVENT_HUBNAME=<event hub name>
```

```bash
python api.py
```

The booking page is then served at `http://localhost:8000`; requesting a ride generates a
confirmation and publishes it to the hub.

---

## Data quality

Controls sit at the boundary where bad data would otherwise enter, upstream of the star schema, so a
problem stops before it reaches anything a consumer queries:

- An explicit `StructType` on `from_json`, so a payload that drifts from the contract surfaces as
  nulls in known columns rather than a silently widened schema
- `failOnDataLoss: true` on the stream, so a skipped offset fails the pipeline instead of leaving a
  gap
- A watermark on `booking_timestamp`, bounding join state and making late-arrival behaviour explicit
- `dropDuplicates` on each dimension key before the CDC flow, so replays and at-least-once delivery
  do not create duplicate dimension rows
- An idempotency guard on the historical load, so re-running the ingestion notebook cannot double
  the backfill

---

## Design decisions

**Ingestion matched to change rate.** A continuous stream for rides; direct storage reads for
mapping tables and a one-time backfill that do not change hourly.

**Raw data untouched.** Bronze keeps the Event Hubs payload as-is, so fixing parsing or business
logic means a re-run, not a re-extraction from a hub whose retention window has since moved.

**One staging table for batch and stream.** Two append flows converge on `stg_rides`, so history and
live events are one dataset downstream rather than two that have to be unioned at read time.

**One Big Table before dimensional modelling.** One definition per concept and one validation point,
so the star schema is built on data that has already been conformed.

**Left joins on reference data.** An unmapped key should degrade a row, not delete it. A ride with
an unknown vehicle type is still a ride, and still revenue.

**SCD Type 2 only where history has meaning.** Locations get validity windows because a city's
region can change and old rides belong to the old region. Passengers and drivers overwrite, because
a stale phone number is not an analytical fact.

**Generated joins over hand-written SQL.** The OBT is the widest, most-edited query in the project,
which makes it the one most worth generating from a declaration.

---

## Limitations and next steps

Development-scale data from a synthetic producer, no CI, and no declared pipeline expectations —
quality is enforced by schema and watermark rather than by assertions that fail a run. All three
medallion layers currently share the `uber.bronze` schema, so the logical layering is not reflected
in the catalog. Most Type 1 dimensions sequence by their own key rather than by an event timestamp,
which makes ordering arbitrary when the same key arrives twice in one batch.

Next: pipeline expectations on keys, ranges and accepted values, with a quarantine path for rows
that fail; separate `bronze`, `silver` and `gold` schemas; an event-time sequencing column on the
remaining dimensions; CI running the full pipeline against a fixture stream on pull requests; and a
BI layer over the fact and dimension tables.

---

Nandesh Kanagaraju — [github.com/nandeshkanagaraju](https://github.com/nandeshkanagaraju)
