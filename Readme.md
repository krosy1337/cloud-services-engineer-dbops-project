# dbops-project

Исходный репозиторий для выполнения проекта дисциплины "DBOps"

1. CREATE DATABASE "store";
2. CREATE ROLE migration_service_user WITH LOGIN PASSWORD 'super_secret';
3. GRANT ALL PRIVILEGES ON DATABASE store TO migration_service_user;
4. GRANT ALL ON SCHEMA public TO migration_service_user;
5. GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO migration_service_user;
6. ALTER DEFAULT PRIVILEGES FOR USER migration_service_user IN SCHEMA public
   GRANT ALL PRIVILEGES ON TABLES TO migration_service_user;

Запрос на количество проданных сосисок:
SELECT
o.date_created, SUM(op.quantity)
FROM
orders AS o
JOIN order_product AS op ON o.id = op.order_id
WHERE
o.status = 'shipped' AND o.date_created > NOW() - INTERVAL '7 DAY'
GROUP BY
o.date_created;

Без индексов:
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

Time: 49.189 ms

                                                             QUERY PLAN

---

HashAggregate (cost=4580.65..4581.56 rows=91 width=12) (actual time=58.458..58.464 rows=7 loops=1)
Group Key: o.date_created
Batches: 1 Memory Usage: 24kB
-> Hash Join (cost=2668.53..4568.04 rows=2522 width=8) (actual time=23.165..58.032 rows=2457 loops=1)
Hash Cond: (op.order_id = o.id)
-> Seq Scan on order_product op (cost=0.00..1637.00 rows=100000 width=12) (actual time=0.006..16.714 rows=100000 loops=1)
-> Hash (cost=2637.00..2637.00 rows=2522 width=12) (actual time=23.137..23.139 rows=2457 loops=1)
Buckets: 4096 Batches: 1 Memory Usage: 148kB
-> Seq Scan on orders o (cost=0.00..2637.00 rows=2522 width=12) (actual time=0.010..22.760 rows=2457 loops=1)
Filter: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
Rows Removed by Filter: 97543
Planning Time: 0.214 ms
Execution Time: 58.499 ms
(13 rows)

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
