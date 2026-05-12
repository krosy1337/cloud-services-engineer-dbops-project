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
 2026-05-02   | 939604
 2026-05-03   | 942396
 2026-05-04   | 939074
 2026-05-05   | 941373
 2026-05-06   | 948902
 2026-05-07   | 954150
 2026-05-08   | 394698
(7 rows)

Time: 6808.070 ms (00:06.808)
```

```
                                                                              QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=188875.34..188979.59 rows=200 width=12) (actual time=5306.216..5328.104 rows=7 loops=1)
   Group Key: o.date_created
   ->  Gather Merge  (cost=188875.34..188975.59 rows=400 width=12) (actual time=5304.798..5328.070 rows=21 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial GroupAggregate  (cost=187875.32..187929.39 rows=200 width=12) (actual time=5239.010..5257.186 rows=7 loops=3)
               Group Key: o.date_created
               ->  Sort  (cost=187875.32..187892.68 rows=6943 width=8) (actual time=5235.168..5243.547 rows=79201 loops=3)
                     Sort Key: o.date_created
                     Sort Method: external merge  Disk: 1376kB
                     Worker 0:  Sort Method: external merge  Disk: 1408kB
                     Worker 1:  Sort Method: external merge  Disk: 1408kB
                     ->  Parallel Hash Join  (cost=71133.82..187432.31 rows=6943 width=8) (actual time=1566.977..5184.001 rows=79201 loops=3)
                           Hash Cond: (op.order_id = o.id)
                           ->  Parallel Seq Scan on order_product op  (cost=0.00..105361.13 rows=4166613 width=12) (actual time=0.255..2168.938 rows=3333333 loops=3)
                           ->  Parallel Hash  (cost=71126.08..71126.08 rows=619 width=12) (actual time=1564.834..1564.835 rows=79201 loops=3)
                                 Buckets: 262144 (originally 2048)  Batches: 1 (originally 1)  Memory Usage: 15280kB
                                 ->  Parallel Seq Scan on orders o  (cost=0.00..71126.08 rows=619 width=12) (actual time=28.218..1466.981 rows=79201 loops=3)
                                       Filter: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
                                       Rows Removed by Filter: 3254132
 Planning Time: 1.132 ms
 JIT:
   Functions: 57
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 4.299 ms, Inlining 0.000 ms, Optimization 4.093 ms, Emission 82.715 ms, Total 91.107 ms
 Execution Time: 5329.315 ms
(26 rows)
Time: 5331.807 ms (00:05.332)
```

С индексами:

```
 date_created |  sum
--------------+--------
 2026-05-02   | 939604
 2026-05-03   | 942396
 2026-05-04   | 939074
 2026-05-05   | 941373
 2026-05-06   | 948902
 2026-05-07   | 954150
 2026-05-08   | 394698
(7 rows)

Time: 3337.492 ms (00:03.337)
```

```
                                                                                    QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=187853.44..187876.49 rows=91 width=12) (actual time=3963.998..3981.404 rows=7 loops=1)
   Group Key: o.date_created
   ->  Gather Merge  (cost=187853.44..187874.67 rows=182 width=12) (actual time=3963.988..3981.392 rows=21 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=186853.41..186853.64 rows=91 width=12) (actual time=3923.738..3923.741 rows=7 loops=3)
               Sort Key: o.date_created
               Sort Method: quicksort  Memory: 25kB
               Worker 0:  Sort Method: quicksort  Memory: 25kB
               Worker 1:  Sort Method: quicksort  Memory: 25kB
               ->  Partial HashAggregate  (cost=186849.54..186850.45 rows=91 width=12) (actual time=3923.718..3923.721 rows=7 loops=3)
                     Group Key: o.date_created
                     Batches: 1  Memory Usage: 24kB
                     Worker 0:  Batches: 1  Memory Usage: 24kB
                     Worker 1:  Batches: 1  Memory Usage: 24kB
                     ->  Parallel Hash Join  (cost=70063.50..186362.67 rows=97374 width=8) (actual time=564.588..3906.099 rows=79201 loops=3)
                           Hash Cond: (op.order_id = o.id)
                           ->  Parallel Seq Scan on order_product op  (cost=0.00..105361.67 rows=4166667 width=12) (actual time=0.357..2061.566 rows=3333333 loops=3)
                           ->  Parallel Hash  (cost=68846.33..68846.33 rows=97374 width=12) (actual time=562.600..562.600 rows=79201 loops=3)
                                 Buckets: 262144  Batches: 1  Memory Usage: 13248kB
                                 ->  Parallel Bitmap Heap Scan on orders o  (cost=3203.84..68846.33 rows=97374 width=12) (actual time=52.775..508.034 rows=79201 loops=3)
                                       Recheck Cond: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
                                       Heap Blocks: exact=26325
                                       ->  Bitmap Index Scan on orders_status_date_idx  (cost=0.00..3145.42 rows=233698 width=0) (actual time=65.558..65.558 rows=237604 loops=1)
                                             Index Cond: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
 Planning Time: 2.290 ms
 JIT:
   Functions: 57
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 4.412 ms, Inlining 0.000 ms, Optimization 4.070 ms, Emission 58.602 ms, Total 67.084 ms
 Execution Time: 3982.181 ms
(31 rows)
Time: 3985.163 ms (00:03.985)
```

В QUERY PLAN видно что используется индекс orders_status_date_idx, а order_product_order_id_idx нет. Запрос стал выполняться примерно в 2 раза быстрее.
