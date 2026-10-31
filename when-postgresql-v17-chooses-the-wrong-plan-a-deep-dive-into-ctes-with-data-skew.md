# When PostgreSQL v17 Chooses the Wrong Plan: A Deep Dive into CTEs with Data Skew

<https://techcommunity.microsoft.com/blog/adforpostgresql/when-postgresql-v17-chooses-the-wrong-plan-a-deep-dive-into-ctes-with-data-skew/4515705>

<span style="color: rgb(30, 30, 30);">Common Table Expressions or CTEs provide temporary named result set that can be referenced within single complex SQL statements. CTE offer great flexibility in breaking down complex SQL queries into multiple readable parts and enables recursion. CTEs allows materialized result set which allows users to refer the result set multiple times.</span>

<span style="color: rgb(30, 30, 30);">PostgreSQL 17 propagates column statistics from MATERIALIZED CTEs into the parent query, allowing the planner to estimate hash aggregation and join cardinalities more accurately.</span>

<span style="color: rgb(30, 30, 30);">However, when queries combine </span>**<span style="color: rgb(30, 30, 30);">highly skewed data distributions</span>**<span style="color: rgb(30, 30, 30);"> with </span>**<span style="color: rgb(30, 30, 30);">runtime randomness</span>**<span style="color: rgb(30, 30, 30);"> (for example, ORDER BY random() LIMIT 1), this increased planner confidence can sometimes lead to execution plans that are correct in theory but catastrophic for specific runtime values.</span>

# <span style="color: rgb(30, 30, 30);">What is Data Skew and how it affects plans?</span>

<span style="color: rgb(30, 30, 30);">Data skew is an uneven distribution of values in a column, where one or a few values occur far more often than the rest. To choose an efficient execution plan, PostgreSQL’s planner estimates row counts using statistics such as n_distinct, histograms, and most-common-values (MCV) frequencies stored in </span>*<span style="color: rgb(30, 30, 30);">pg_statistic</span>*<span style="color: rgb(30, 30, 30);">. If these estimates are off—especially if selectivity is overestimated—query performance can degrade significantly.</span>

<span style="color: rgb(30, 30, 30);">Baseline Query and Observed Planner Behavior</span>

<span style="color: rgb(30, 30, 30);">To gauge the performance impact of data skew with PGv17, we generated data into 3 tables relating to Accounts, Purchases and Audit.  </span>

<span style="color: rgb(30, 30, 30);">This baseline query selects a single random HIGH_RISK account and retrieves matching rows from a large audit_logs table via two materialized CTEs.</span>

### <span style="color: rgb(30, 30, 30);">Original Query</span>

```sql
with selected_account as materialized (  
select account_id
    from accounts
    where account_type = 'HIGH_RISK'
    order by random()
    limit 1
),
audit_purchases as materialized (
    select distinct p.account_id
    from purchases p
    join selected_account sa
      on p.account_id = sa.account_id
)
select al.*
from audit_logs al
where exists (
 select 1 
from audit_purchases ap 
where al.account_id = ap.account_id);
```

### <span style="color: rgb(30, 30, 30);">PostgreSQL 14 Execution Plan</span>

<span style="color: rgb(30, 30, 30);">PostgreSQL 14 treats materialized CTEs as optimization fences and does not propagate detailed statistics into the parent query.</span>

<span style="color: rgb(30, 30, 30);">As a result, the planner assumes relatively small CTE result sets and consistently favors </span>**<span style="color: rgb(30, 30, 30);">nested loop joins with parameterized index scans</span>**<span style="color: rgb(30, 30, 30);">. While these estimates are not strictly accurate, this conservative planning approach prevents full-table scans and avoids catastrophic performance under extreme data skew.</span>

<span style="color: rgb(30, 30, 30);">In this workload, PostgreSQL 14 completes the query in approximately </span>**<span style="color: rgb(30, 30, 30);">7 ms</span>**<span style="color: rgb(30, 30, 30);">.</span>

```c
                                                                            QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------
Nested Loop  (cost=1412.94..212757485.16 rows=1500600064 width=27) (actual time=5.226..7.192 rows=30 loops=1)
   Buffers: shared hit=277 read=13
   CTE selected_account
     ->  Limit  (cost=971.00..971.00 rows=1 width=12) (actual time=4.460..4.460 rows=1 loops=1)
           Buffers: shared hit=271
           ->  Sort  (cost=971.00..996.00 rows=10000 width=12) (actual time=4.459..4.459 rows=1 loops=1)
                 Sort Key: (random())
                 Sort Method: top-N heapsort  Memory: 25kB
                 Buffers: shared hit=271
                 ->  Seq Scan on accounts  (cost=0.00..921.00 rows=10000 width=12) (actual time=0.011..3.533 rows=10000 loops=1)
                       Filter: (account_type = 'HIGH_RISK'::text)
                       Rows Removed by Filter: 40000
                       Buffers: shared hit=271
   CTE audit_purchases
     ->  HashAggregate  (cost=317.44..355.57 rows=3813 width=4) (actual time=4.735..4.741 rows=1 loops=1)
           Group Key: p.account_id
           Batches: 1  Memory Usage: 217kB
           Buffers: shared hit=274 read=1
           ->  Nested Loop  (cost=0.44..292.52 rows=9966 width=4) (actual time=4.722..4.726 rows=10 loops=1)
                 Buffers: shared hit=274 read=1
                 ->  CTE Scan on selected_account sa  (cost=0.00..0.02 rows=1 width=4) (actual time=4.461..4.461 rows=1 loops=1)
                       Buffers: shared hit=271
                 ->  Index Only Scan using idx_purchases_account on purchases p  (cost=0.44..192.84 rows=9966 width=4) (actual time=0.260..0.262 rows=10 loops=1)
                       Index Cond: (account_id = sa.account_id)
                       Heap Fetches: 0
                       Buffers: shared hit=3 read=1
   ->  HashAggregate  (cost=85.79..87.79 rows=200 width=4) (actual time=4.743..4.744 rows=1 loops=1)
         Group Key: ap.account_id
         Batches: 1  Memory Usage: 40kB
         Buffers: shared hit=274 read=1
         ->  CTE Scan on audit_purchases ap  (cost=0.00..76.26 rows=3813 width=4) (actual time=4.737..4.742 rows=1 loops=1)
               Buffers: shared hit=274 read=1
   ->  Index Scan using idx_account_audit on audit_logs al  (cost=0.58..817780.34 rows=24600001 width=27) (actual time=0.480..2.438 rows=30 loops=1)
         Index Cond: (account_id = ap.account_id)
         Buffers: shared hit=3 read=12
 Planning Time: 1.121 ms
 Execution Time: 7.253 ms
(37 rows)
```

### <span style="color: rgb(30, 30, 30);">PostgreSQL 17 Execution Plan</span>

<span style="color: rgb(30, 30, 30);">PostgreSQL 17 correctly estimates that the </span>*<span style="color: rgb(30, 30, 30);">audit_purchases</span>*<span style="color: rgb(30, 30, 30);"> CTE may contain thousands of rows and that joining it with audit_logs could produce tens of millions of matches.</span>

<span style="color: rgb(30, 30, 30);">Based on these estimates, the planner selects a </span>**<span style="color: rgb(30, 30, 30);">Hash Semi Join with a Sequential Scan</span>**<span style="color: rgb(30, 30, 30);"> on </span>*<span style="color: rgb(30, 30, 30);">audit_logs</span>*<span style="color: rgb(30, 30, 30);">, which is optimal when many rows are expected to match.</span>

<span style="color: rgb(30, 30, 30);">However, at runtime the selected account produces only \~30 matching audit rows. Despite this, PostgreSQL must execute the chosen plan fully, scanning millions of rows from disk. This results in an execution time of </span>**<span style="color: rgb(30, 30, 30);">521,625 ms (\~8.7 minutes)</span>**<span style="color: rgb(30, 30, 30);">.</span>

<span style="color: rgb(30, 30, 30);">Importantly, this behavior is not a cost estimation bug—the estimates are statistically correct.</span>

<span style="color: rgb(30, 30, 30);">The failure occurs because the PostgreSQL 17 planner estimates a </span>**<span style="color: rgb(30, 30, 30);">large matching result set</span>**<span style="color: rgb(30, 30, 30);"> based on the cardinality of values produced by the </span>*<span style="color: rgb(30, 30, 30);">audit_purchases</span>*<span style="color: rgb(30, 30, 30);"> CTE, and consequently chooses a plan optimized for that estimated volume rather than for the single runtime-selected account.</span>

```c
                                                                            QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Hash Semi Join  (cost=1691.60..93353890.16 rows=3001396992 width=26) (actual time=86577.858..521625.286 rows=30 loops=1)
   Hash Cond: (al.account_id = ap.account_id)
   Buffers: shared hit=28879 read=22040419
   CTE selected_account
     ->  Limit  (cost=971.00..971.00 rows=1 width=12) (actual time=4.486..4.487 rows=1 loops=1)
           Buffers: shared hit=274
           ->  Sort  (cost=971.00..996.00 rows=10000 width=12) (actual time=4.485..4.486 rows=1 loops=1)
                 Sort Key: (random())
                 Sort Method: top-N heapsort  Memory: 25kB
                 Buffers: shared hit=274
                 ->  Seq Scan on accounts  (cost=0.00..921.00 rows=10000 width=12) (actual time=0.021..3.551 rows=10000 loops=1)
                       Filter: (account_type = 'HIGH_RISK'::text)
                       Rows Removed by Filter: 40000
                       Buffers: shared hit=271
   CTE audit_purchases
     ->  HashAggregate  (cost=658.72..673.28 rows=1456 width=4) (actual time=4.907..4.912 rows=1 loops=1)
           Group Key: p.account_id
           Batches: 1  Memory Usage: 73kB
           Buffers: shared hit=277 read=1
           ->  Nested Loop  (cost=0.44..606.86 rows=20742 width=4) (actual time=4.898..4.902 rows=10 loops=1)
                 Buffers: shared hit=277 read=1
                 ->  CTE Scan on selected_account sa  (cost=0.00..0.02 rows=1 width=4) (actual time=4.487..4.487 rows=1 loops=1)
                       Buffers: shared hit=274
                 ->  Index Only Scan using idx_purchases_account on purchases p  (cost=0.44..399.42 rows=20742 width=4) (actual time=0.410..0.411 rows=10 loops=1)
                       Index Cond: (account_id = sa.account_id)
                       Heap Fetches: 0
                       Buffers: shared hit=3 read=1
   ->  Seq Scan on audit_logs al  (cost=0.00..52082989.92 rows=3001396992 width=26) (actual time=0.012..286755.571 rows=3001400050 loops=1)
         Buffers: shared hit=28602 read=22040418
   ->  Hash  (cost=29.12..29.12 rows=1456 width=4) (actual time=4.919..4.920 rows=1 loops=1)
         Buckets: 2048  Batches: 1  Memory Usage: 17kB
         Buffers: shared hit=277 read=1
         ->  CTE Scan on audit_purchases ap  (cost=0.00..29.12 rows=1456 width=4) (actual time=4.908..4.911 rows=1 loops=1)
               Buffers: shared hit=277 read=1
 Planning:
   Buffers: shared hit=197
 Planning Time: 1.831 ms
 Execution Time: 521625.433 ms
(38 rows)
```

### <span style="color: rgb(30, 30, 30);">Forced nested loop</span>

<span style="color: rgb(30, 30, 30);">Disabling hash joins at the session level forces PostgreSQL 17 to revert to a nested loop plan using index scans. This diagnostic step completes in </span>**<span style="color: rgb(30, 30, 30);">\~8 ms</span>**<span style="color: rgb(30, 30, 30);">, confirming that index-based execution is sufficient for the actual runtime workload. However, disabling planner features is not suitable for production use and serves only to validate the root cause.</span>

```c
        	                                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=1677.62..3361930001.36 rows=3001396992 width=26) (actual time=5.513..8.194 rows=30 loops=1)
   Buffers: shared hit=277 read=12
   CTE selected_account
     ->  Limit  (cost=971.00..971.00 rows=1 width=12) (actual time=4.524..4.524 rows=1 loops=1)
           Buffers: shared hit=271
           ->  Sort  (cost=971.00..996.00 rows=10000 width=12) (actual time=4.523..4.523 rows=1 loops=1)
                 Sort Key: (random())
                 Sort Method: top-N heapsort  Memory: 25kB
                 Buffers: shared hit=271
                 ->  Seq Scan on accounts  (cost=0.00..921.00 rows=10000 width=12) (actual time=0.014..3.608 rows=10000 loops=1)
                       Filter: (account_type = 'HIGH_RISK'::text)
                       Rows Removed by Filter: 40000
                       Buffers: shared hit=271
   CTE audit_purchases
     ->  HashAggregate  (cost=658.72..673.28 rows=1456 width=4) (actual time=4.890..4.894 rows=1 loops=1)
           Group Key: p.account_id
           Batches: 1  Memory Usage: 73kB
           Buffers: shared hit=274 read=1
           ->  Nested Loop  (cost=0.44..606.86 rows=20742 width=4) (actual time=4.882..4.885 rows=10 loops=1)
                 Buffers: shared hit=274 read=1
                 ->  CTE Scan on selected_account sa  (cost=0.00..0.02 rows=1 width=4) (actual time=4.525..4.525 rows=1 loops=1)
                       Buffers: shared hit=271
                 ->  Index Only Scan using idx_purchases_account on purchases p  (cost=0.44..399.42 rows=20742 width=4) (actual time=0.354..0.356 rows=10 loops=1)
                       Index Cond: (account_id = sa.account_id)
                       Heap Fetches: 0
                       Buffers: shared hit=3 read=1
   ->  HashAggregate  (cost=32.76..47.32 rows=1456 width=4) (actual time=4.896..4.899 rows=1 loops=1)
         Group Key: ap.account_id
         Batches: 1  Memory Usage: 73kB
         Buffers: shared hit=274 read=1
         ->  CTE Scan on audit_purchases ap  (cost=0.00..29.12 rows=1456 width=4) (actual time=4.892..4.895 rows=1 loops=1)
               Buffers: shared hit=274 read=1
   ->  Index Scan using idx_account_audit on audit_logs al  (cost=0.58..1782455.82 rows=52656088 width=26) (actual time=0.615..3.283 rows=30 loops=1)
         Index Cond: (account_id = ap.account_id)
         Buffers: shared hit=3 read=11
 Planning Time: 0.184 ms
 Execution Time: 8.252 ms
(37 rows)
```

### <span style="color: rgb(30, 30, 30);">Fix Strategies</span>

<span style="color: rgb(30, 30, 30);">While the fix isn’t straightforward, we attempt to modify the query plan through query rewrite using following methods:</span>

1. <span style="color: rgb(30, 30, 30);">Lateral Join with Offset 0</span>
2. <span style="color: rgb(30, 30, 30);">Limit on Purchases</span>
3. <span style="color: rgb(30, 30, 30);">Using subqueries</span>

<span style="color: rgb(30, 30, 30);">Lateral Join with Offset 0</span>

<span style="color: rgb(30, 30, 30);">Adding OFFSET 0 inside a LATERAL subquery introduces an </span>**<span style="color: rgb(30, 30, 30);">optimizer barrier</span>**<span style="color: rgb(30, 30, 30);"> that prevents join reordering.</span>

<span style="color: rgb(30, 30, 30);">This forces PostgreSQL to execute a </span>**<span style="color: rgb(30, 30, 30);">parameterized nested loop</span>**<span style="color: rgb(30, 30, 30);">, passing the runtime account ID into an index scan on </span>*<span style="color: rgb(30, 30, 30);">audit_logs</span>*<span style="color: rgb(30, 30, 30);">.</span>

<span style="color: rgb(30, 30, 30);">With this rewrite, PostgreSQL 17 executes the query in </span>**<span style="color: rgb(30, 30, 30);">6.519 ms</span>**<span style="color: rgb(30, 30, 30);">, a </span>**<span style="color: rgb(30, 30, 30);">99.9988% improvement</span>**<span style="color: rgb(30, 30, 30);"> over the original plan.</span>

```sql
with selected_account as materialized (
  select account_id
  from accounts
  where account_type = 'HIGH_RISK'
  order by random()
  limit 1
),
audit_purchases as materialized (
  select p.account_id
  from purchases p
  join selected_account sa
    on p.account_id = sa.account_id
)
select al.*
from audit_purchases ap
join lateral (
  select *
  from audit_logs al where al.account_id = ap.account_id offset 0) al on true;
```

<span style="color: rgb(30, 30, 30);">Execution plan</span>

```c
                                                                         QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=1578.44..42162640233.98 rows=1092192577296 width=26) (actual time=4.980..6.481 rows=300 loops=1)
   Buffers: shared hit=403 read=12
   CTE selected_account
     ->  Limit  (cost=971.00..971.00 rows=1 width=12) (actual time=4.296..4.297 rows=1 loops=1)
           Buffers: shared hit=271
           ->  Sort  (cost=971.00..996.00 rows=10000 width=12) (actual time=4.295..4.296 rows=1 loops=1)
                 Sort Key: (random())
                 Sort Method: top-N heapsort  Memory: 25kB
                 Buffers: shared hit=271
                 ->  Seq Scan on accounts  (cost=0.00..921.00 rows=10000 width=12) (actual time=0.013..3.357 rows=10000 loops=1)
                       Filter: (account_type = 'HIGH_RISK'::text)
                       Rows Removed by Filter: 40000
                       Buffers: shared hit=271
   CTE audit_purchases
     ->  Nested Loop  (cost=0.44..606.86 rows=20742 width=4) (actual time=4.585..4.588 rows=10 loops=1)
           Buffers: shared hit=274 read=1
           ->  CTE Scan on selected_account sa  (cost=0.00..0.02 rows=1 width=4) (actual time=4.298..4.298 rows=1 loops=1)
                 Buffers: shared hit=271
           ->  Index Only Scan using idx_purchases_account on purchases p  (cost=0.44..399.42 rows=20742 width=4) (actual time=0.286..0.287 rows=10 loops=1)
                 Index Cond: (account_id = sa.account_id)
                 Heap Fetches: 0
                 Buffers: shared hit=3 read=1
   ->  CTE Scan on audit_purchases ap  (cost=0.00..414.84 rows=20742 width=4) (actual time=4.587..4.591 rows=10 loops=1)
         Buffers: shared hit=274 read=1
   ->  Index Scan using idx_account_audit on audit_logs al  (cost=0.58..1506157.19 rows=52656088 width=26) (actual time=0.040..0.185 rows=30 loops=10)
         Index Cond: (account_id = ap.account_id)
         Buffers: shared hit=129 read=11
 Planning:
   Buffers: shared hit=8
 Planning Time: 0.238 ms
 Execution Time: 6.519 ms
(31 rows)
```

<span style="color: rgb(30, 30, 30);">Limit on Purchases</span>

<span style="color: rgb(30, 30, 30);">Applying LIMIT 1 to the </span>*<span style="color: rgb(30, 30, 30);">audit_purchases</span>*<span style="color: rgb(30, 30, 30);"> CTE bounds its cardinality to a single row. This makes nested loops cheaper than hash joins during planning.</span>

<span style="color: rgb(30, 30, 30);">PostgreSQL 17 selects an index‑driven execution plan and completes in </span>**<span style="color: rgb(30, 30, 30);">7.742 ms</span>**<span style="color: rgb(30, 30, 30);">, yielding a </span>**<span style="color: rgb(30, 30, 30);">99.9985% improvement</span>**<span style="color: rgb(30, 30, 30);">.</span>

```sql
with selected_account as materialized (
  select account_id
  from accounts
  where account_type = 'HIGH_RISK'
  order by random()
  limit 1
),
audit_purchases as materialized (
  select p.account_id
  from purchases p
  join selected_account sa
    on p.account_id = sa.account_id limit 1
)
select al.*
from audit_logs al where exists (select 1 from audit_purchases ap where al.account_id=ap.account_id);
```

<span style="color: rgb(30, 30, 30);">Execution plan</span>

```c
                                                                           QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=972.07..933697.99 rows=24600001 width=27) (actual time=5.693..7.686 rows=30 loops=1)
   Buffers: shared hit=276 read=13
   CTE selected_account
     ->  Limit  (cost=971.00..971.00 rows=1 width=12) (actual time=4.816..4.817 rows=1 loops=1)
           Buffers: shared hit=271
           ->  Sort  (cost=971.00..996.00 rows=10000 width=12) (actual time=4.816..4.817 rows=1 loops=1)
                 Sort Key: (random())
                 Sort Method: top-N heapsort  Memory: 25kB
                 Buffers: shared hit=271
                 ->  Seq Scan on accounts  (cost=0.00..921.00 rows=10000 width=12) (actual time=0.014..3.825 rows=10000 loops=1)
                       Filter: (account_type = 'HIGH_RISK'::text)
                       Rows Removed by Filter: 40000
                       Buffers: shared hit=271
   CTE audit_purchases
     ->  Limit  (cost=0.44..0.47 rows=1 width=4) (actual time=5.133..5.134 rows=1 loops=1)
           Buffers: shared hit=274 read=1
           ->  Nested Loop  (cost=0.44..292.52 rows=9966 width=4) (actual time=5.132..5.132 rows=1 loops=1)
                 Buffers: shared hit=274 read=1
                 ->  CTE Scan on selected_account sa  (cost=0.00..0.02 rows=1 width=4) (actual time=4.817..4.817 rows=1 loops=1)
                       Buffers: shared hit=271
                 ->  Index Only Scan using idx_purchases_account on purchases p  (cost=0.44..192.84 rows=9966 width=4) (actual time=0.313..0.313 rows=1 loops=1)
                       Index Cond: (account_id = sa.account_id)
                       Heap Fetches: 0
                       Buffers: shared hit=3 read=1
   ->  HashAggregate  (cost=0.02..0.03 rows=1 width=4) (actual time=5.138..5.139 rows=1 loops=1)
         Group Key: ap.account_id
         Batches: 1  Memory Usage: 24kB
         Buffers: shared hit=274 read=1
         ->  CTE Scan on audit_purchases ap  (cost=0.00..0.02 rows=1 width=4) (actual time=5.135..5.135 rows=1 loops=1)
               Buffers: shared hit=274 read=1
   ->  Index Scan using idx_account_audit on audit_logs al  (cost=0.58..686726.47 rows=24600001 width=27) (actual time=0.552..2.534 rows=30 loops=1)
         Index Cond: (account_id = ap.account_id)
         Buffers: shared hit=2 read=12
 Planning Time: 0.216 ms
 Execution Time: 7.742 ms
(35 rows)
```

<span style="color: rgb(30, 30, 30);">Using subqueries</span>

<span style="color: rgb(30, 30, 30);">Rewriting the query using scalar subqueries converts the selected account ID into an InitPlan.</span>

<span style="color: rgb(30, 30, 30);">The resulting value is then used as a runtime parameter in index scans against audit_logs. This eliminates join reordering opportunities and guarantees a parameterized access path.</span>

<span style="color: rgb(30, 30, 30);">Both scalar subquery variants complete in approximately 7.3–7.6 ms, achieving over 99.9986% improvement.</span>

<span style="color: rgb(30, 30, 30);">Subquery Example 1</span>

```sql
select * from audit_logs al where al.account_id in (select p.account_id from purchases p where p.account_id=(
select account_id from accounts where account_type = 'HIGH_RISK' order by random() limit 1));
```

<span style="color: rgb(30, 30, 30);">Execution plan</span>

```c
                                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop Semi Join  (cost=972.02..1302907.98 rows=24600001 width=27) (actual time=5.257..7.260 rows=30 loops=1)
   Buffers: shared hit=277 read=12
   InitPlan 1 (returns $0)
     ->  Limit  (cost=971.00..971.00 rows=1 width=12) (actual time=4.612..4.613 rows=1 loops=1)
           Buffers: shared hit=271
           ->  Sort  (cost=971.00..996.00 rows=10000 width=12) (actual time=4.611..4.611 rows=1 loops=1)
                 Sort Key: (random())
                 Sort Method: top-N heapsort  Memory: 25kB
                 Buffers: shared hit=271
                 ->  Seq Scan on accounts  (cost=0.00..921.00 rows=10000 width=12) (actual time=0.012..3.674 rows=10000 loops=1)
                       Filter: (account_type = 'HIGH_RISK'::text)
                       Rows Removed by Filter: 40000
                       Buffers: shared hit=271
   ->  Index Scan using idx_account_audit on audit_logs al  (cost=0.58..686755.98 rows=24600001 width=27) (actual time=5.067..7.059 rows=30 loops=1)
         Index Cond: (account_id = $0)
         Buffers: shared hit=274 read=11
   ->  Materialize  (cost=0.44..242.67 rows=9966 width=4) (actual time=0.006..0.006 rows=1 loops=30)
         Buffers: shared hit=3 read=1
         ->  Index Only Scan using idx_purchases_account on purchases p  (cost=0.44..192.84 rows=9966 width=4) (actual time=0.187..0.188 rows=1 loops=1)
               Index Cond: (account_id = $0)
               Heap Fetches: 0
               Buffers: shared hit=3 read=1
 Planning Time: 0.160 ms
 Execution Time: 7.286 ms
(24 rows)
```

<span style="color: rgb(30, 30, 30);">Subquery Example 2</span>

```sql
select * from audit_logs al where al.account_id=(select distinct p.account_id from purchases p where p.account_id=(
select account_id from accounts where account_type = 'HIGH_RISK' order by random() limit 1));
```

<span style="color: rgb(30, 30, 30);">Execution plan</span>

```c
                                                                         QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_account_audit on audit_logs al  (cost=972.04..1507128.65 rows=52656088 width=26) (actual time=5.510..7.637 rows=30 loops=1)
   Index Cond: (account_id = (InitPlan 2).col1)
   Buffers: shared hit=277 read=12
   InitPlan 2
     ->  Limit  (cost=971.44..971.46 rows=1 width=4) (actual time=4.901..4.902 rows=1 loops=1)
           Buffers: shared hit=274 read=1
           InitPlan 1
             ->  Limit  (cost=971.00..971.00 rows=1 width=12) (actual time=4.455..4.456 rows=1 loops=1)
                   Buffers: shared hit=271
                   ->  Sort  (cost=971.00..996.00 rows=10000 width=12) (actual time=4.455..4.455 rows=1 loops=1)
                         Sort Key: (random())
                         Sort Method: top-N heapsort  Memory: 25kB
                         Buffers: shared hit=271
                         ->  Seq Scan on accounts  (cost=0.00..921.00 rows=10000 width=12) (actual time=0.018..3.522 rows=10000 loops=1)
                               Filter: (account_type = 'HIGH_RISK'::text)
                               Rows Removed by Filter: 40000
                               Buffers: shared hit=271
           ->  Index Only Scan using idx_purchases_account on purchases p  (cost=0.44..399.42 rows=20742 width=4) (actual time=4.900..4.900 rows=1 loops=1)
                 Index Cond: (account_id = (InitPlan 1).col1)
                 Heap Fetches: 0
                 Buffers: shared hit=274 read=1
 Planning:
   Buffers: shared hit=4
 Planning Time: 0.154 ms
 Execution Time: 7.663 ms
(25 rows)
```

## <span style="color: rgb(30, 30, 30);">Query execution summary</span>

<span style="color: rgb(30, 30, 30);">The table below summarizes execution time improvements across all tested rewrites relative to the original PostgreSQL 17 and 14 query execution plans.</span>

<table style="min-width: 75px;">
<colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong><span style="color: rgb(30, 30, 30);">Setup / Rewrite</span></strong></p></td><td colspan="1" rowspan="1"><p><strong><span style="color: rgb(30, 30, 30);">Execution Time (ms)</span></strong></p></td><td colspan="1" rowspan="1"><p><strong><span style="color: rgb(30, 30, 30);">Speed‑up Factor (×)</span></strong></p></td></tr><tr><td colspan="1" rowspan="1"><p><strong><span style="color: rgb(30, 30, 30);">PG17 original (Hash Semi Join + Seq Scan)</span></strong></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">521,625.433</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">1.0×</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">PG14 original query shape</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">7.253</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">71,919×</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">Forced nested loop (hashjoin off – diagnostic)</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">8.252</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">63,212×</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><strong><span style="color: rgb(30, 30, 30);">LATERAL JOIN + OFFSET 0</span></strong></p></td><td colspan="1" rowspan="1"><p><strong><span style="color: rgb(30, 30, 30);">6.519</span></strong></p></td><td colspan="1" rowspan="1"><p><strong><span style="color: rgb(30, 30, 30);">80,016×</span></strong></p></td></tr><tr><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">LIMIT on purchases</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">7.742</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">67,376×</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">Subquery – IN with InitPlan (Example 1)</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">7.286</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">71,593×</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">Subquery – scalar subquery (Example 2)</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">7.663</span></p></td><td colspan="1" rowspan="1"><p><span style="color: rgb(30, 30, 30);">68,071×</span></p></td></tr></tbody>
</table>

## <span style="color: rgb(30, 30, 30);">Key Takeaways</span>

<span style="color: rgb(30, 30, 30);">While PostgreSQL 17’s planner improvements are correct, intentional, and beneficial for most workloads.</span>

<span style="color: rgb(30, 30, 30);">However, queries that combine </span>**<span style="color: rgb(30, 30, 30);">CTEs, runtime randomness, and heavily skewed data</span>**<span style="color: rgb(30, 30, 30);"> can cause the planner to select globally optimal plans that are locally inefficient.</span>

<span style="color: rgb(30, 30, 30);">When runtime values drastically narrow result sets, query rewrites that </span>**<span style="color: rgb(30, 30, 30);">force parameterized index scans</span>**<span style="color: rgb(30, 30, 30);">—such as LATERAL joins, scalar subqueries, or bounded CTEs—provide predictable and stable performance.</span>
