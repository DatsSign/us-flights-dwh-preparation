# U.S. Flights DWH Preparation

**Stack:** SQL (PostgreSQL)

* **Data Volume & Quality (5.8M rows):** Data cleaning, standardization, accuracy, and consistency checks.
* **Query Optimization:** Accelerated filtering and `SELECT` queries using B-tree and partial indexes.
* **Star Schema:** Designed a BI-ready set of fact and dimension tables.
* **Data Mart:** Built a dedicated analytical mart to test a specific business hypothesis.

<img width="947" height="723" alt="ER_diagram" src="https://github.com/user-attachments/assets/55358ddb-7861-4f89-930d-82751fdf803f" />

## Query Optimization Benchmark

```sql
begin;

-- 1. Benchmark before indexing
explain (analyze, buffers)
select 
    fl.flight_date,
    fl.tail_number,
    count(fl.flight_number) as flight_count,
    cc.cancellation_description
from fact_flights fl
join dim_airports ai 
    on fl.destination_ap = ai.iata_code_ap
join dim_calendar cr
    on fl.flight_date = cr.date
left join dim_cancellation_codes cc
    on fl.cancellation_reason = cc.cancellation_reason
where ai.state = 'TX' 
    and cr.month_name = 'Mar'
    and fl.cancelled = 0
group by 1, 2, 4
order by 3 desc;

-- 2. Apply B-tree and partial indexes
create index idx_date_dest on fact_flights (flight_date, destination_ap);
create index idx_airline on fact_flights (airline_code);
create index idx_origin_ap on fact_flights (origin_ap);
create index idx_cncl_reason on fact_flights (cancellation_reason) where cancellation_reason is not null;

-- 3. Benchmark after indexing
explain (analyze, buffers)
select 
    fl.flight_date,
    fl.tail_number,
    count(fl.flight_number) as flight_count,
    cc.cancellation_description
from fact_flights fl
join dim_airports ai 
    on fl.destination_ap = ai.iata_code_ap
join dim_calendar cr
    on fl.flight_date = cr.date
left join dim_cancellation_codes cc
    on fl.cancellation_reason = cc.cancellation_reason
where ai.state = 'TX' 
    and cr.month_name = 'Mar'
    and fl.cancelled = 0
group by 1, 2, 4
order by 3 desc;

rollback;
```
### Benchmark Results

| Metric | Before Indexing | After Indexing | Improvement |
| :--- | :---: | :---: | :---: |
| **Scan Strategy** | `Parallel Seq Scan` | `Index Scan` (`idx_date_dest`) | Targeted lookup |
| **Execution Time** | 3,870.6 ms | 1,256.5 ms | **3.1x faster (-67.5%)** |
| **Disk Reads (`shared read`)** | 196,573 blocks (~1.54 GB) | 6,124 blocks (~48 MB) | **-96.9% (32x less I/O)** |
| **Total Buffers Accessed** | 211,833 | 35,586 | **-83.2%** |

> The composite index `idx_date_dest (flight_date, destination_ap)` eliminated the full table scan across 5.8M rows, cutting disk I/O by nearly 97% and speeding up query execution by more than 3x.

## Analytical Data Mart (`dm_route_performance`)

Monthly route performance mart designed for BI reporting (Grain: `Month × Airline × Route`):
* **Delay Split:** Separates average arrival delays and early arrivals (completed flights only).
* **MoM Trends:** Tracks previous month metrics via `LAG()` partitioned by IATA airport codes to avoid multi-airport city collisions.
* **Rankings & Edge Cases:** Ranks routes per airline using `DENSE_RANK()` with `NULLS LAST` and zero-division protection (`NULLIF`).

<details>
<summary><b>View SQL Query</b></summary>

```sql
with month_grain as (
    select
        date_trunc('month', ff.flight_date)::date as month_start,
        dc.month_year,
        ff.airline_code,
        ff.origin_ap,
        ff.destination_ap,
        count(*) as total_flights,
        count(*) filter (where ff.cancelled = 1) as cancelled_flight,
        count(*) filter (where ff.cancelled = 0 and ff.diverted = 0) as completed_flights,
        sum(ff.arrival_delay) filter (where ff.cancelled = 0 and ff.diverted = 0 and ff.arrival_delay > 0) as sum_arr_delay,
        sum(ff.arrival_delay) filter (where ff.cancelled = 0 and ff.diverted = 0 and ff.arrival_delay < 0) as sum_arr_hurry
    from fact_flights ff
    join dim_calendar dc
        on ff.flight_date = dc."date"
    group by 1, 2, 3, 4, 5
),
  pull_info as (
    select
        mg.month_start,
        mg.month_year as "Month-Year",
        mg.origin_ap,
        mg.destination_ap,
        airL.airline as "Airline",
        airP_o.city as "Origin city",
        airP_d.city as "Destination city",
        mg.total_flights as "Total flights",
        round(mg.sum_arr_delay::numeric / nullif(mg.completed_flights, 0), 2) as "Avg arrival delay",
        round(mg.sum_arr_hurry::numeric / nullif(mg.completed_flights, 0), 2) as "Avg early arrival",
        round((mg.cancelled_flight * 100.0) / nullif(mg.total_flights, 0), 2) as "Cancelled %"
    from month_grain mg
    join dim_airlines airL
        on mg.airline_code = airL.airline_code
    join dim_airports airP_o
        on mg.origin_ap = airP_o.iata_code_ap
    join dim_airports airP_d
        on mg.destination_ap = airP_d.iata_code_ap
)
select
    "Month-Year",
    "Airline",
    "Origin city",
    "Destination city",
    "Total flights",
    "Avg arrival delay",
    "Avg early arrival",
    lag("Avg arrival delay") over (
        partition by "Airline", origin_ap, destination_ap
        order by month_start
    ) as "Previous Avg arrival delay",
    lag("Avg early arrival") over (
        partition by "Airline", origin_ap, destination_ap
        order by month_start
    ) as "Previous Avg early arrival",
    "Cancelled %",
    dense_rank() over (
        partition by month_start, "Airline"
        order by "Avg arrival delay" desc nulls last
    ) as "Delay rank",
    dense_rank() over (
        partition by month_start, "Airline"
        order by "Avg early arrival" asc nulls last
    ) as "Early arrival rank"
from pull_info;
```
</details>

## Data Source
[Maven Analytics — Airline Flight Delays (U.S. DOT, 2015)](https://app.mavenanalytics.io/datasets?order=-fields.numberOfRecords)
