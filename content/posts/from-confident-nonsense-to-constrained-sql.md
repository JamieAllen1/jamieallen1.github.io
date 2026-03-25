+++
date = '2026-03-25T07:00:00+10:00'
draft = false
title = 'From Confident Nonsense to Constrained SQL'
+++

# TL/DR
I built a SQL AI Agent (using OpenAI/ChatGPT 5) just to see how they worked, and the results were kind of surprising. All code in this example is in my Github repo (https://github.com/JamieAllen1/duckdb-sql-agent-poc)

# Introduction
I was flicking through some reading thanks to [TLDR AI](https://tldr.tech/ai/2026-01-30) and linked through to this blog on [OpenAI's in-house data agent](https://openai.com/index/inside-our-in-house-data-agent/?utm_source=tldrai). Interesting read. Seeing as I'm a lot of a data nerd and a bit of an AI nerd, I figured it would be interesting to see what it takes to build my own.

OpenAI has a very nice document on what makes an AI Agent an Agent and what doesn't with some neat examples at (https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) but I want one that queries a database. Simples right?

So where do I start - the beginning.

# Step 1 - Find a database and some data
I just happen to have DuckDB laying around on my laptop. Don't judge me - I like in-memory, column oriented DBs. Data? DuckDB comes with a nice little data generator that's a core extension called tpch and is used for benchmarking the database. Do I care about DuckDB benchmarking? No I don't. Does it create tables with data in them? Yes it does. It comes with a nice little parameter called Scale Factor (sf), the larger the number - the more data it creates. I don't need a lot to define stuff - so lets go with 0.1 which will give us a database of about 25MB.

```python
import duckdb

con = duckdb.connect("data/tpch.duckdb")

# Generate TPC-H at scale factor 0.1 (tiny but useful)
con.execute("CALL dbgen(sf=0.1);")

# Sanity check
tables = con.execute("SHOW TABLES;").fetchall()
print("Tables created:")
for t in tables:
    print("-", t[0])

con.close()
```

This script created the following tables:
```
Tables created:
- customer
- lineitem
- nation
- orders
- part
- partsupp
- region
- supplier
```

Remember these table names - we'll need them soon.

# v0 - Can an LLM create a SQL query?

One could assume it can - let's prove it. 

```python
from dotenv import load_dotenv
import os

from openai import OpenAI

load_dotenv()

client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY")
)

SYSTEM_PROMPT = """
You translate analytical questions into DuckDB SQL.
Assume a reasonable schema if not provided.
Return only SQL.
"""

def generate_sql(question: str) -> str:
    resp = client.responses.create(
        model="gpt-5",
        input=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": question},
        ],
    )
    return resp.output_text.strip()

if __name__ == "__main__":
    q = "Top 10 customers by total revenue last quarter"
    print(generate_sql(q))
```

Nothing world breaking there. Let's see what it comes up with.

```
WITH quarter_bounds AS (
  SELECT
    date_trunc('quarter', current_date) - INTERVAL '3 months' AS start_last_quarter,
    date_trunc('quarter', current_date) AS start_this_quarter
)
SELECT
  c.customer_id,
  c.customer_name,
  SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN customers c ON c.customer_id = o.customer_id
CROSS JOIN quarter_bounds qb
WHERE o.order_date >= qb.start_last_quarter
  AND o.order_date < qb.start_this_quarter
GROUP BY c.customer_id, c.customer_name
ORDER BY total_revenue DESC
LIMIT 10;
```

Anyone see any issues yet? Just for the hell of it, and because we'll need it later, let's get it to run the query. Let's create this as v1

# v1 - Create a query and run it
I've changed a few things in this version. 

1. Added DuckDB connectivity
2. Upgraded the prompt as it was creating some very weird sql and changed the constraint
3. Added then run_sql function to allow it to run the query.
4. Added main() function to call everything and print the results (if any) and show errors.

The new code is right in the repo [here](https://github.com/JamieAllen1/duckdb-sql-agent-poc/tree/main/v1_tool_execution)

So what did I get from it. 
```
--- Generated SQL ---

WITH bounds AS (
  SELECT
    date_trunc('quarter', current_date) AS start_current_q,
    date_trunc('quarter', current_date) - INTERVAL 3 MONTH AS start_last_q
)
SELECT
  c.customer_id,
  c.customer_name,
  SUM(COALESCE(o.total_amount, 0)) AS total_revenue
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
CROSS JOIN bounds b
WHERE o.order_date >= b.start_last_q
  AND o.order_date < b.start_current_q
GROUP BY c.customer_id, c.customer_name
ORDER BY total_revenue DESC
LIMIT 10

--- DuckDB Result ---

ERROR:
Catalog Error: Table with name customers does not exist!
Did you mean "customer"?

LINE 11: JOIN customers c ON c.customer_id = o.customer_id
              ^

SQL that failed:

WITH bounds AS (
  SELECT
    date_trunc('quarter', current_date) AS start_current_q,
    date_trunc('quarter', current_date) - INTERVAL 3 MONTH AS start_last_q
)
SELECT
  c.customer_id,
  c.customer_name,
  SUM(COALESCE(o.total_amount, 0)) AS total_revenue
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
CROSS JOIN bounds b
WHERE o.order_date >= b.start_last_q
  AND o.order_date < b.start_current_q
GROUP BY c.customer_id, c.customer_name
ORDER BY total_revenue DESC
LIMIT 10
```

What can we take from this example? ChatGPT knows nothing about the schema, and just made stuff up. That doesn't help a lot. Next step - let's let ChatGPT see the schema first, and see what it comes up with then.

# v2 - Adding schema awareness
This version adds a lot.

First off - I add a `get_schema_context` function to get the schema from DuckDB. Then, I update the `generate_sql` function to pass the schema information to the LLM in the user content (essentially context information) to allow it to use the schema to define the query. Rather than past the code here, I'll add a link to the [final version](https://github.com/JamieAllen1/duckdb-sql-agent-poc/tree/main/v2_schema_awareness).

Here's what happens when I run it.

```
--- Schema Context (excerpt) ---

SCHEMA (DuckDB catalog):
TABLES:
- customer: c_custkey:BIGINT, c_name:VARCHAR, c_address:VARCHAR, c_nationkey:INTEGER, c_phone:VARCHAR, c_acctbal:DECIMAL(15,2), c_mktsegment:VARCHAR, c_comment:VARCHAR
- lineitem: l_orderkey:BIGINT, l_partkey:BIGINT, l_suppkey:BIGINT, l_linenumber:BIGINT, l_quantity:DECIMAL(15,2), l_extendedprice:DECIMAL(15,2), l_discount:DECIMAL(15,2), l_tax:DECIMAL(15,2), l_returnflag:VARCHAR, l_linestatus:VARCHAR, l_shipdate:DATE, l_commitdate:DATE, l_receiptdate:DATE, l_shipinstruct:VARCHAR, l_shipmode:VARCHAR, l_comment:VARCHAR
- nation: n_nationkey:INTEGER, n_name:VARCHAR, n_regionkey:INTEGER, n_comment:VARCHAR
- orders: o_orderkey:BIGINT, o_custkey:BIGINT, o_orderstatus:VARCHAR, o_totalprice:DECIMAL(15,2), o_orderdate:DATE, o_orderpriority:VARCHAR, o_clerk:VARCHAR, o_shippriority:INTEGER, o_comment:VARCHAR
- part: p_partkey:BIGINT, p_name:VARCHAR, p_mfgr:VARCHAR, p_brand:VARCHAR, p_type:VARCHAR, p_size:INTEGER, p_container:VARCHAR, p_retailprice:DECIMAL(15,2), p_comment:VARCHAR
- partsupp: ps_partkey:BIGINT, ps_suppkey:BIGINT, ps_availqty:BIGINT, ps_supplycost:DECIMAL(15,2), ps_comment:VARCHAR
- region: r_regionkey:INTEGER, r_name:VARCHAR, r_comment:VARCHAR
- supplier: s_suppkey:BIGINT, s_name:VARCHAR, s_address:VARCHAR, s_nationkey:INTEGER, s_phone:VARCHAR, s_acctbal:DECIMAL(15,2), s_comment:VARCHAR

... (truncated)


--- Generated SQL ---

WITH prev_q AS (
  SELECT
    date_trunc('quarter', CURRENT_DATE) - INTERVAL 3 MONTH AS q_start,
    date_trunc('quarter', CURRENT_DATE) - INTERVAL 1 DAY AS q_end
)
SELECT
  c.c_custkey,
  c.c_name,
  SUM(li.l_extendedprice * (1 - li.l_discount)) AS total_revenue
FROM prev_q p
JOIN orders o ON o.o_orderdate BETWEEN p.q_start AND p.q_end
JOIN lineitem li ON li.l_orderkey = o.o_orderkey
JOIN customer c ON c.c_custkey = o.o_custkey
GROUP BY c.c_custkey, c.c_name
ORDER BY total_revenue DESC
LIMIT 10;

--- DuckDB Result ---

Columns: ['c_custkey', 'c_name', 'total_revenue']
```

No rows returned?!?!?!

Here's where we've got too. ChatGPT knows about the schema, but it knows nothing about the data. If I look at the orders table in DuckDB, we find this:
```
D SELECT min(o_orderdate), max(o_orderdate) FROM orders;
┌──────────────────┬──────────────────┐
│ min(o_orderdate) │ max(o_orderdate) │
│       date       │       date       │
├──────────────────┼──────────────────┤
│ 1992-01-01       │ 1998-08-02       │
└──────────────────┴──────────────────┘
```
The tpch module creates the data back last century. Let's update the prompt to generate data for the last three quarters relative to the last order date with this:
`- If the question uses relative time (e.g. "last quarter"), anchor it to the dataset using max(date_column) when available, not current_date`

Let's try it again.

```
--- Schema Context (excerpt) ---

SCHEMA (DuckDB catalog):
TABLES:
- customer: c_custkey:BIGINT, c_name:VARCHAR, c_address:VARCHAR, c_nationkey:INTEGER, c_phone:VARCHAR, c_acctbal:DECIMAL(15,2), c_mktsegment:VARCHAR, c_comment:VARCHAR
- lineitem: l_orderkey:BIGINT, l_partkey:BIGINT, l_suppkey:BIGINT, l_linenumber:BIGINT, l_quantity:DECIMAL(15,2), l_extendedprice:DECIMAL(15,2), l_discount:DECIMAL(15,2), l_tax:DECIMAL(15,2), l_returnflag:VARCHAR, l_linestatus:VARCHAR, l_shipdate:DATE, l_commitdate:DATE, l_receiptdate:DATE, l_shipinstruct:VARCHAR, l_shipmode:VARCHAR, l_comment:VARCHAR
- nation: n_nationkey:INTEGER, n_name:VARCHAR, n_regionkey:INTEGER, n_comment:VARCHAR
- orders: o_orderkey:BIGINT, o_custkey:BIGINT, o_orderstatus:VARCHAR, o_totalprice:DECIMAL(15,2), o_orderdate:DATE, o_orderpriority:VARCHAR, o_clerk:VARCHAR, o_shippriority:INTEGER, o_comment:VARCHAR
- part: p_partkey:BIGINT, p_name:VARCHAR, p_mfgr:VARCHAR, p_brand:VARCHAR, p_type:VARCHAR, p_size:INTEGER, p_container:VARCHAR, p_retailprice:DECIMAL(15,2), p_comment:VARCHAR
- partsupp: ps_partkey:BIGINT, ps_suppkey:BIGINT, ps_availqty:BIGINT, ps_supplycost:DECIMAL(15,2), ps_comment:VARCHAR
- region: r_regionkey:INTEGER, r_name:VARCHAR, r_comment:VARCHAR
- supplier: s_suppkey:BIGINT, s_name:VARCHAR, s_address:VARCHAR, s_nationkey:INTEGER, s_phone:VARCHAR, s_acctbal:DECIMAL(15,2), s_comment:VARCHAR

... (truncated)


--- Generated SQL ---

WITH bounds AS (
  SELECT
    date_trunc('quarter', max(o_orderdate)) AS current_q_start,
    date_trunc('quarter', max(o_orderdate)) - INTERVAL '3 months' AS last_q_start
  FROM orders
)
SELECT
  c.c_custkey,
  c.c_name,
  SUM(l.l_extendedprice * (1 - l.l_discount)) AS total_revenue
FROM bounds b
JOIN orders o
  ON o.o_orderdate >= b.last_q_start
 AND o.o_orderdate < b.current_q_start
JOIN lineitem l ON l.l_orderkey = o.o_orderkey
JOIN customer c ON c.c_custkey = o.o_custkey
GROUP BY c.c_custkey, c.c_name
ORDER BY total_revenue DESC
LIMIT 10;

--- DuckDB Result ---

Columns: ['c_custkey', 'c_name', 'total_revenue']
(13207, 'Customer#000013207', Decimal('1179787.4360'))
(4657, 'Customer#000004657', Decimal('934506.3642'))
(1006, 'Customer#000001006', Decimal('906958.1703'))
(9877, 'Customer#000009877', Decimal('787345.8546'))
(8635, 'Customer#000008635', Decimal('784201.0802'))
(13771, 'Customer#000013771', Decimal('782794.3374'))
(11624, 'Customer#000011624', Decimal('752368.6609'))
(8092, 'Customer#000008092', Decimal('732949.2941'))
(1408, 'Customer#000001408', Decimal('731986.5933'))
(14836, 'Customer#000014836', Decimal('715083.0379'))
```
That looks like something a little more useful. Is it analytics ready yet? Not really. But it's a clean "Top 10" dataset. 

What have we found from this version. AI (or an LLM) can generate a useful query that works and run it against the database. What we did there was let it define the query, run it, then we refined the prompt, tried it again and we got something useful. Next version, we automate that process, with guardrails, and see if we can make it better.

# v3 – Adding a constrained retry loop
In v2 I effectively acted as the “agent loop” myself:
- run the query
- observe the outcome
- adjust the prompt
- retry

In v3, I want to automate that. Not with infinite retries, and not with “autonomous” anything. Just a bounded loop that can:
- observe basic facts about the dataset
- attempt a query
- if it fails (or returns nothing), revise and retry a small number of times

The key point: **observations should never crash the run**. They’re diagnostics, not mission-critical. I actually hit this immediately when one of my own observation CTE names collided with a DuckDB reserved word. Humans: also fallible.

## What counts as “observation”?
For this dataset I used a few cheap checks:

- order date range (min/max)
- row counts for the main tables
- how many orders exist in the “last quarter” window relative to the dataset max date

Here’s the output:
```
--- Observations (TPCH sanity checks) ---

{'orders_date_range': {'ok': True, 'value': ('1992-01-01', '1998-08-02')}, 'counts_customer': {'ok': True, 'value': 15000}, 'counts_orders': {'ok': True, 'value': 150000}, 'counts_lineitem': {'ok': True, 'value': 600572}, 'orders_in_last_quarter_asof_max_date': {'ok': True, 'value': 5746}}
```

That last number is important: the dataset *does* have data in the “last quarter” window if you anchor it correctly. So if a query returns 0 rows, we know something is off.

## Attempt 1
With schema + prompt constraints + observations in place, the first attempt succeeded:
```
--- Attempt 1/3: Generated SQL ---

WITH max_date AS (
  SELECT max(o_orderdate) AS max_date
  FROM orders
),
bounds AS (
  SELECT
    date_trunc('quarter', max_date) AS curr_q_start,
    date_trunc('quarter', max_date) - INTERVAL '3 months' AS last_q_start
  FROM max_date
)
SELECT
  c.c_custkey,
  c.c_name,
  SUM(l.l_extendedprice * (1 - l.l_discount)) AS total_revenue
FROM orders o
JOIN customer c ON c.c_custkey = o.o_custkey
JOIN lineitem l ON l.l_orderkey = o.o_orderkey
JOIN bounds b ON TRUE
WHERE o.o_orderdate >= b.last_q_start
  AND o.o_orderdate < b.curr_q_start
GROUP BY c.c_custkey, c.c_name
ORDER BY total_revenue DESC
LIMIT 10;
```

And the result matched v2:
```
--- DuckDB Result ---

Columns: ['c_custkey', 'c_name', 'total_revenue']
(13207, 'Customer#000013207', Decimal('1179787.4360'))
(4657, 'Customer#000004657', Decimal('934506.3642'))
(1006, 'Customer#000001006', Decimal('906958.1703'))
(9877, 'Customer#000009877', Decimal('787345.8546'))
(8635, 'Customer#000008635', Decimal('784201.0802'))
(13771, 'Customer#000013771', Decimal('782794.3374'))
(11624, 'Customer#000011624', Decimal('752368.6609'))
(8092, 'Customer#000008092', Decimal('732949.2941'))
(1408, 'Customer#000001408', Decimal('731986.5933'))
(14836, 'Customer#000014836', Decimal('715083.0379'))
```

## So… was a retry loop even needed?
For this question and dataset, it didn’t need to retry. 

But that’s missing the point.

v3 is about changing the architecture from:
- “human-in-the-loop prompt fiddling”
to:
- “a bounded loop that can react to failures”

This matters because the moment you ask a slightly different question, or introduce different datasets, you will hit:
- SQL syntax errors
- wrong join paths
- empty result sets
- ambiguous metrics (what counts as “revenue”?)

The retry loop isn’t magic. It’s just the minimum scaffolding needed for the system to correct itself based on what actually happened, rather than what it hoped would happen.

What's next?

The next step is not to make it smarter. It is to make it safer.

This is where v4 comes in:
- enforce read-only queries
- force LIMITs
- add timeouts
- standardise output (SQL used, assumptions, truncation warnings)

# v4 - Adding Real Guardrails

Now we're almost at the point of having the beginnings of an agent. I won't consider to call it a real SQL agent as there is so much more that would have to go into it to make it work properly, and to guard it from doing something terrible. But this version will be the last for our little journey into the dark side, and at least I'll have a better idea of how they work, and hopefully you will too.

## What I did

A lot of the code in this version is still the same as v3. 

v4 is where I stopped trusting the agent and started constraining it. Up until this point it could generate, run, and even revise SQL, but it would still happily do whatever it thought was right. In this version I added a simple guardrail layer: enforce read-only queries, inject or cap `LIMIT`s, and apply a basic timeout so it can’t run away with itself. On top of that, I standardised the output so every result includes the SQL that was executed, a set of inferred assumptions, and any warnings (like truncated results or guardrail interventions). None of this makes the agent smarter. It just makes its behaviour more predictable, easier to inspect, and a lot harder to accidentally trust without thinking.

So - how did it go?

## The output

```
--- Observations (TPCH sanity checks) ---

{'orders_date_range': {'ok': True, 'value': ('1992-01-01', '1998-08-02')}, 'counts_customer': {'ok': True, 'value': 15000}, 'counts_orders': {'ok': True, 'value': 150000}, 'counts_lineitem': {'ok': True, 'value': 600572}, 'orders_in_last_quarter_asof_max_date': {'ok': True, 'value': 5746}}

--- Attempt 1/3: Generated SQL ---

WITH bounds AS (
  SELECT
    date_trunc('quarter', max(o_orderdate)) AS current_qtr_start,
    date_trunc('quarter', max(o_orderdate)) - INTERVAL '3 months' AS last_qtr_start
  FROM orders
)
SELECT
  c.c_custkey,
  c.c_name,
  c.c_address,
  c.c_phone,
  n.n_name AS nation,
  SUM(l.l_extendedprice * (1 - l.l_discount)) AS total_revenue
FROM bounds b
JOIN orders o
  ON o.o_orderdate >= b.last_qtr_start
 AND o.o_orderdate < b.current_qtr_start
JOIN lineitem l ON l.l_orderkey = o.o_orderkey
JOIN customer c ON c.c_custkey = o.o_custkey
LEFT JOIN nation n ON n.n_nationkey = c.c_nationkey
GROUP BY c.c_custkey, c.c_name, c.c_address, c.c_phone, n.n_name
ORDER BY total_revenue DESC
LIMIT 10;

--- DuckDB Result ---

Columns: ['c_custkey', 'c_name', 'c_address', 'c_phone', 'nation', 'total_revenue']
(13207, 'Customer#000013207', 'PP7IinI0y7M5aDPQNMMto', '18-380-733-7887', 'INDIA', Decimal('1179787.4360'))
(4657, 'Customer#000004657', '80BclnwGfu4R,ECOAfY5j0', '13-193-651-4217', 'CANADA', Decimal('934506.3642'))
(1006, 'Customer#000001006', 'bWUBQGOzQWhvqeJc8', '22-364-780-5932', 'JAPAN', Decimal('906958.1703'))
(9877, 'Customer#000009877', 'FvILgiPmWCTPX kEb xOHAJ OCKH6NSke', '25-675-164-2805', 'MOROCCO', Decimal('787345.8546'))
(8635, 'Customer#000008635', '8kgtIsxoCtPXQohFyS,iTILi2QhvS', '26-644-815-3446', 'MOZAMBIQUE', Decimal('784201.0802'))
(13771, 'Customer#000013771', 'Vx9ieEzC40r', '13-697-441-8002', 'CANADA', Decimal('782794.3374'))
(11624, 'Customer#000011624', '85fBe8pd2rlW1bxaf6CnyWRmsUncORaFnRY,B1,', '25-840-812-8455', 'MOROCCO', Decimal('752368.6609'))
(8092, 'Customer#000008092', 'jq1gwKQxyfPHD PRVjX0904', '20-764-894-1875', 'IRAN', Decimal('732949.2941'))
(1408, 'Customer#000001408', 'eeiPXCZnt611W3cm68,p1', '21-901-381-6344', 'IRAQ', Decimal('731986.5933'))
(14836, 'Customer#000014836', 'AEYiXgKOWnR3Zfr7lBNZ5y', '33-909-505-1055', 'UNITED KINGDOM', Decimal('715083.0379'))
jamieallen@MacBookPro duckdb-sql-agent-poc % python v4_guardrails/main.py

--- Observations (TPCH sanity checks) ---

{'orders_date_range': {'ok': True, 'value': ('1992-01-01', '1998-08-02')}, 'counts_customer': {'ok': True, 'value': 15000}, 'counts_orders': {'ok': True, 'value': 150000}, 'counts_lineitem': {'ok': True, 'value': 600572}, 'orders_in_last_quarter_asof_max_date': {'ok': True, 'value': 5746}}

--- Attempt 1/3: Generated SQL ---

WITH maxd AS (
  SELECT MAX(o_orderdate) AS max_date FROM orders
),
bounds AS (
  SELECT
    (DATE_TRUNC('quarter', max_date) - INTERVAL '3 months') AS start_prev_qtr,
    (DATE_TRUNC('quarter', max_date) - INTERVAL '1 day') AS end_prev_qtr
  FROM maxd
)
SELECT
  c.c_custkey,
  c.c_name,
  SUM(l.l_extendedprice * (1 - l.l_discount)) AS total_revenue
FROM orders o
JOIN bounds b ON TRUE
JOIN customer c ON c.c_custkey = o.o_custkey
JOIN lineitem l ON l.l_orderkey = o.o_orderkey
WHERE o.o_orderdate >= b.start_prev_qtr
  AND o.o_orderdate <= b.end_prev_qtr
GROUP BY c.c_custkey, c.c_name
ORDER BY total_revenue DESC
LIMIT 10;

--- DuckDB Result ---

Columns: ['c_custkey', 'c_name', 'total_revenue']
(13207, 'Customer#000013207', Decimal('1179787.4360'))
(4657, 'Customer#000004657', Decimal('934506.3642'))
(1006, 'Customer#000001006', Decimal('906958.1703'))
(9877, 'Customer#000009877', Decimal('787345.8546'))
(8635, 'Customer#000008635', Decimal('784201.0802'))
(13771, 'Customer#000013771', Decimal('782794.3374'))
(11624, 'Customer#000011624', Decimal('752368.6609'))
(8092, 'Customer#000008092', Decimal('732949.2941'))
(1408, 'Customer#000001408', Decimal('731986.5933'))
(14836, 'Customer#000014836', Decimal('715083.0379'))

--- Output Contract ---

Question:
Top 10 customers by total revenue last quarter

SQL used:
WITH maxd AS (
  SELECT MAX(o_orderdate) AS max_date FROM orders
),
bounds AS (
  SELECT
    (DATE_TRUNC('quarter', max_date) - INTERVAL '3 months') AS start_prev_qtr,
    (DATE_TRUNC('quarter', max_date) - INTERVAL '1 day') AS end_prev_qtr
  FROM maxd
)
SELECT
  c.c_custkey,
  c.c_name,
  SUM(l.l_extendedprice * (1 - l.l_discount)) AS total_revenue
FROM orders o
JOIN bounds b ON TRUE
JOIN customer c ON c.c_custkey = o.o_custkey
JOIN lineitem l ON l.l_orderkey = o.o_orderkey
WHERE o.o_orderdate >= b.start_prev_qtr
  AND o.o_orderdate <= b.end_prev_qtr
GROUP BY c.c_custkey, c.c_name
ORDER BY total_revenue DESC
LIMIT 10;

Assumptions:
- Relative time is anchored to max(o_orderdate) in the dataset.
- Revenue appears to be derived from line item extended price and discount.
- Results are aggregated at customer level.
- Output is intentionally limited to a subset of rows.

Warnings:
- None.

Guardrails:
{'readonly_enforced': True, 'limit_info': {'limit_added': False, 'limit_capped': False, 'final_limit': 10, 'reason': 'existing_limit'}, 'timeout_seconds': 5, 'elapsed_seconds': 0.013}
```

## Final thoughts

What started as a simple “can it write SQL?” experiment turned into something a bit more interesting. The model was never the problem. It was perfectly capable of producing plausible answers from the start. The difference between v0 and v4 wasn’t intelligence, it was constraint.

At each step, the system became less “clever” and more grounded:
- first by forcing it to see the schema
- then by anchoring it to the data itself
- then by letting it react to failure
- and finally by putting guardrails around what it’s allowed to do

That’s really the takeaway. Agents aren’t about autonomy, they’re about **controlled behaviour under uncertainty**. Left alone, they’re very good at being confidently wrong. With the right constraints, they become predictably useful.

This little PoC isn’t a production system, and it’s not trying to be. But it does show where the real work is: not in getting an LLM to generate SQL, but in building the boundaries that make its output something you can actually trust.