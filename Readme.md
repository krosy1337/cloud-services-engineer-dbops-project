# dbops-project

Исходный репозиторий для выполнения проекта дисциплины "DBOps"

Создание пользователя и выдача ему прав

```sql
CREATE ROLE migration_service_user WITH LOGIN PASSWORD 'super_secret';
GRANT ALL PRIVILEGES ON DATABASE store TO migration_service_user;
GRANT ALL ON SCHEMA public TO migration_service_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO migration_service_user;
ALTER DEFAULT PRIVILEGES FOR USER migration_service_user IN SCHEMA public
   GRANT ALL PRIVILEGES ON TABLES TO migration_service_user;
```

Запрос на количество проданных сосисок:

```sql
SELECT
   o.date_created, SUM(op.quantity)
FROM
   orders AS o
JOIN order_product AS op ON o.id = op.order_id
WHERE
   o.status = 'shipped' AND o.date_created > NOW() - INTERVAL '7 DAY'
GROUP BY
   o.date_created;
```

Без индексов:

```
 date_created |  sum
--------------+--------
 2026-04-21   | 937027
 2026-04-22   | 940427
 2026-04-23   | 940481
 2026-04-24   | 941524
 2026-04-25   | 940392
 2026-04-26   | 944741
 2026-04-27   | 823028
(7 rows)

Time: 36027.160 ms (00:36.027)
```

```
 Finalize GroupAggregate  (cost=266220.32..266243.38 rows=91 width=12) (actual time=35616.838..35632.325 rows=7 loops=1)
   Group Key: o.date_created
   ->  Gather Merge  (cost=266220.32..266241.56 rows=182 width=12) (actual time=35616.800..35632.285 rows=21 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=265220.30..265220.53 rows=91 width=12) (actual time=35549.193..35549.199 rows=7 loops=3)
               Sort Key: o.date_created
               Sort Method: quicksort  Memory: 25kB
               Worker 0:  Sort Method: quicksort  Memory: 25kB
               Worker 1:  Sort Method: quicksort  Memory: 25kB
               ->  Partial HashAggregate  (cost=265216.43..265217.34 rows=91 width=12) (actual time=35549.162..35549.169 rows=7 loops=3)
                     Group Key: o.date_created
                     Batches: 1  Memory Usage: 24kB
                     Worker 0:  Batches: 1  Memory Usage: 24kB
                     Worker 1:  Batches: 1  Memory Usage: 24kB
                     ->  Parallel Hash Join  (cost=148376.82..264676.59 rows=107967 width=8) (actual time=17684.090..35527.086 rows=84596 loops=3)
                           Hash Cond: (op.order_id = o.id)
                           ->  Parallel Seq Scan on order_product op  (cost=0.00..105362.15 rows=4166715 width=12) (actual time=6.734..16388.698 rows=3333333 loops=3)
                           ->  Parallel Hash  (cost=147027.26..147027.26 rows=107965 width=12) (actual time=17675.577..17675.578 rows=84596 loops=3)
                                 Buckets: 262144  Batches: 1  Memory Usage: 13984kB
                                 ->  Parallel Seq Scan on orders o  (cost=0.00..147027.26 rows=107965 width=12) (actual time=25.879..17620.547 rows=84596 loops=3)
                                       Filter: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
                                       Rows Removed by Filter: 3248738
 Planning Time: 0.219 ms
 JIT:
   Functions: 54
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 7.038 ms, Inlining 0.000 ms, Optimization 1.274 ms, Emission 60.840 ms, Total 69.152 ms
 Execution Time: 35633.519 ms
(29 rows)
```

С индексами

```
date_created | sum
--------------+-------
2026-04-22 | 9334
2026-04-21 | 8657
2026-04-27 | 6082
2026-04-26 | 9403
2026-04-25 | 9461
2026-04-23 | 9517
2026-04-24 | 10118
(7 rows)

Time: 31.216 ms
                                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=2669.23..2670.14 rows=91 width=12) (actual time=35.454..35.460 rows=7 loops=1)
   Group Key: o.date_created
   Batches: 1  Memory Usage: 24kB
   ->  Hash Join  (cost=757.11..2656.62 rows=2522 width=8) (actual time=4.495..35.095 rows=2457 loops=1)
         Hash Cond: (op.order_id = o.id)
         ->  Seq Scan on order_product op  (cost=0.00..1637.00 rows=100000 width=12) (actual time=0.005..11.058 rows=100000 loops=1)
         ->  Hash  (cost=725.59..725.59 rows=2522 width=12) (actual time=4.468..4.471 rows=2457 loops=1)
               Buckets: 4096  Batches: 1  Memory Usage: 148kB
               ->  Bitmap Heap Scan on orders o  (cost=38.15..725.59 rows=2522 width=12) (actual time=0.256..4.046 rows=2457 loops=1)
                     Recheck Cond: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
                     Heap Blocks: exact=627
                     ->  Bitmap Index Scan on orders_status_date_idx  (cost=0.00..37.52 rows=2522 width=0) (actual time=0.163..0.163 rows=2457 loops=1)
                           Index Cond: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
 Planning Time: 0.274 ms
 Execution Time: 35.504 ms
(15 rows)

Time: 36.772 ms
```
