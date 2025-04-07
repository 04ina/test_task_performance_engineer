# test_task_performance_engineer

## Установка PostgreSQL 16.7
Устанавливать мы постгрес будем путем компиляции из исходников. Это позволит нам воспользоваться отладочными инструментами, если мы встретим "необъяснимое" поведение СУБД при построении планов. 

При компиляции с ключом -O0 большинство оптимизаций производительности отключены, что может замедлить исполнение запроса, но не на порядок; поэтому мы все равно должны уложиться во временные диапазоны, указанные в задании. 

Примечания:
1. Пользователя postgres в linux мы создавать не будем, поскольку постгрес нам необходим не для enterprise задач - безопасность нам ни к чему. Вследствие этого суперпользователь у нас будет иметь доступ к PGDATA.
2. Все зависимости (flex, bison и тд.) уже установлены.
```bash
git clone https://github.com/postgres/postgres.git
cd postgres/
git checkout REL_16_7
./configure --prefix=$(pwd)/../pg CFLAGS='-O0 -g'
make -j4
sudo make install -j4
cd ../pg/bin
mkdir ./../../PGDATA
./initdb -D ./../../PGDATA
./pg_ctl -D ./../../PGDATA start
```
## Запуск скриптов
Подключаемся к постгресу посредством терминального клиента psql
```bash
./psql postgres
```
Создаем базу данных и выполняем скрипты генерации
```sql
CREATE DATABASE test_db;
\c test_db
```
```sql
-- turn off parallel queries
alter system set max_parallel_workers_per_gather=0;
-- apply changes
select pg_reload_conf();

-- t1 table has 10e7 rows
create table if not exists t1 as
 select a.id as id
      , cast(a.id - floor( random() *(a.id) ) as integer ) as parent_id
      , substr( md5( random()::text ), 0, 30 )  as name
 from generate_series ( 1, 10000000 ) a(id);

-- t2 table has 5*10e6 rows
create table if not exists t2  as
select row_number() over() as id
     , id as t_id
     , to_char(date_trunc('day', now()- random()*'1 year'::interval),'yyyymmdd') as day
from t1
order by random() 
limit 5000000;
```
## Задачи
### Задача №1
ускорить простой запроc, добиться времени выполнения < 10ms
```sql
select name from t1 where id = 50000;
```
###### Почему запрос медленный
Чтобы найти и вывести нужное нам значение name, постгресу необходимо с помощью полного перебора посредством seq scan найти все строки, id которых равно значению 50000. Поскольку существует всего одна строка с id=50000 (из-за generate_series), использование полного перебора всех строк таблицы t1 неоправдано.
```sql
explain select name from t1 where id = 50000;
                         QUERY PLAN                         
------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..208480.00 rows=50035 width=32)
   Filter: (id = 50000)
(2 rows)
```
###### Решение
Для увеличения скорости выполнения данного запроса необходимо "навестить" индекс на столбец id. Это позволит производить выборку с высокой селективностью по t1.id значительно быстрее. Для наших целей подойдет как btree, так и hash индексы. 
```sql
CREATE INDEX ON t1(id);
CREATE INDEX

explain (analyze, timing off) select name from t1 where id = 50000;
                                         QUERY PLAN                                          
---------------------------------------------------------------------------------------------
 Index Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=30) (actual rows=1 loops=1)
   Index Cond: (id = 50000)
 Planning Time: 0.124 ms
 Execution Time: 0.085 ms
(4 rows)
```
### Задача №2
ускорить запрос "max + left join", добиться времени выполнения < 10ms
```sql
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
###### Почему запрос медленный
Данный запрос выполняется медленно вследствие использования соединения, которое, несмотря на предварительную фильтрацию по столбцу t1.name и использование хеш функций, сопутствуется высокими накладными расходами. Хеш соединение само по себе является отличным механизмом соединения, однако порой использование соединения в принципе не оправдано для получения той или иной выборки.
```sql
explain select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Aggregate  (cost=385682.77..385682.78 rows=1 width=32)
   ->  Hash Left Join  (cost=218335.76..373182.96 rows=4999923 width=9)
         Hash Cond: (t2.t_id = t1.id)
         ->  Seq Scan on t2  (cost=0.00..81871.23 rows=4999923 width=13)
         ->  Hash  (cost=208392.00..208392.00 rows=606061 width=4)
               ->  Seq Scan on t1  (cost=0.00..208392.00 rows=606061 width=4)
                     Filter: (name ~~ 'a%'::text)
(7 rows)
```
###### Решение
Нетрудно догадаться, что данный запрос на самом деле не нуждается в левом внешнем соединении -- при любых условиях соединения левая часть результирующего отношения соединения будет состоять из 5000000 значений левого исходного отношения t2, поэтому предикат t1.name like 'a%' может повлиять лишь на правую часть результирующего отношения соединения. Тем временем, значение max(t2.day) может варьироваться только в соответсвтии с левой частью результирующей таблицы соединения -- правая часть для max(t2.day) будет фиктивной. В связи с этим мы можем убрать любое упоминание таблицы t1 и операцию соединения в запросе:
```sql
explain (analyze, timing off) select max(t2.day) from t2;
                                           QUERY PLAN                                           
------------------------------------------------------------------------------------------------
 Aggregate  (cost=94371.04..94371.05 rows=1 width=32) (actual rows=1 loops=1)
   ->  Seq Scan on t2  (cost=0.00..81871.23 rows=4999923 width=9) (actual rows=5000000 loops=1)
 Planning Time: 0.165 ms
 Execution Time: 1698.118 ms
(4 rows)
``` 
Чтобы избежать полного перебора и последующей агрегации, навесим индекс на столбец day. Таким образом, seq scan и агрегация будет заменена на сканирование непосредственно из индекса самого первого наибольшего значения:
```sql
CREATE INDEX ON t2(day);
CREATE INDEX

explain (analyze, timing off) select max(t2.day) from t2;
                                                             QUERY PLAN                                                             
------------------------------------------------------------------------------------------------------------------------------------
 Result  (cost=0.45..0.46 rows=1 width=32) (actual rows=1 loops=1)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.43..0.45 rows=1 width=9) (actual rows=1 loops=1)
           ->  Index Only Scan Backward using t2_day_idx on t2  (cost=0.43..104780.44 rows=5000000 width=9) (actual rows=1 loops=1)
                 Index Cond: (day IS NOT NULL)
                 Heap Fetches: 0
 Planning Time: 0.285 ms
 Execution Time: 0.132 ms
(8 rows)
```
### Задача №3
ускорить запрос "anti-join", добиться времени выполнения < 10sec
```sql
select day from t2 where t_id not in ( select t1.id from t1 );
```
###### Почему запрос медленный
Данный запрос для каждой записи из таблицы t2 проверяет, не содержится ли t2.t_id в столбце t1.id таблицы t1. Стоит отметить, что планировщик для подзапроса использует материализацию, чтобы по нескольку раз не считывать таплы из страниц таблицы t1, однако сильно запросу это не помогает. Подзапрос выполняется отдельно, материализуется, а затем уже происходит фильтрация всего материализованного набора строк таблицы t1 для каждой строки таблицы t2. Поэтому запрос и выполняется медленно.
```sql
explain select day from t2 where t_id not in ( select t1.id from t1 );
                                 QUERY PLAN                                 
----------------------------------------------------------------------------
 Seq Scan on t2  (cost=0.00..743637594372.00 rows=2500000 width=9)
   Filter: (NOT (SubPlan 1))
   SubPlan 1
     ->  Materialize  (cost=0.00..272455.00 rows=10000000 width=4)
           ->  Seq Scan on t1  (cost=0.00..183392.00 rows=10000000 width=4)
(5 rows)
```
###### Решение
Семантика запроса подразумевает, что мы должны должны вывести значения t2.day всех строк таблицы t2, соответствующие значения t2.t_id которых не равны значениям t1.id ни одной строки из таблицы t1. Причем в результирующем отношении должны быть значения лишь из одной таблицы t2 (мы выводим лишь t2.day столбец). Таким образом, этот запрос можно значительно улучшить, если воспользоваться конструкцией anti-join с использованием NOT EXISTS. 
```sql
explain (analyze,timing off) SELECT day FROM t2 WHERE NOT EXISTS (SELECT * FROM t1 WHERE t1.id = t2.t_id);
                                              QUERY PLAN                                               
-------------------------------------------------------------------------------------------------------
 Hash Right Anti Join  (cost=168787.00..592220.00 rows=1 width=9) (actual rows=0 loops=1)
   Hash Cond: (t1.id = t2.t_id)
   ->  Seq Scan on t1  (cost=0.00..183392.00 rows=10000000 width=4) (actual rows=10000000 loops=1)
   ->  Hash  (cost=81872.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
         Buckets: 262144  Batches: 64  Memory Usage: 5720kB
         ->  Seq Scan on t2  (cost=0.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
 Planning Time: 0.177 ms
 Execution Time: 9385.418 ms
(8 rows)
```
Соединение по хешу с модификацией anti-join позволит за кратчайшее время выявить строки таблицы t2, для которых не было найдено ни одного соответствия из таблицы t1.

Также мы можем воспользоваться иной конструкцией запроса, которая будет эквивалентна вышеописанному варианту благодаря механизму безусловных улучшений SQL запроса. 
```sql
explain (analyze, timing off) SELECT t2.day FROM t2 LEFT JOIN t1 ON t2.t_id = t1.id WHERE t1.id IS NULL;
                                              QUERY PLAN                                               
-------------------------------------------------------------------------------------------------------
 Hash Right Anti Join  (cost=168787.00..592220.00 rows=1 width=9) (actual rows=0 loops=1)
   Hash Cond: (t1.id = t2.t_id)
   ->  Seq Scan on t1  (cost=0.00..183392.00 rows=10000000 width=4) (actual rows=10000000 loops=1)
   ->  Hash  (cost=81872.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
         Buckets: 262144  Batches: 64  Memory Usage: 5720kB
         ->  Seq Scan on t2  (cost=0.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
 Planning Time: 0.204 ms
 Execution Time: 9493.130 ms
(8 rows)
```
### Задача №4
ускорить запрос "semi-join", добиться времени выполнения < 10sec
```sql
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
###### Почему запрос медленный
Для каждой строки из таблицы t2 выполняется подзапрос, представляющий собой последовательное сканирование с фильтрацией по предикату t2.t_id = t1.id. Затем уже результат выполнения данного подзапроса еще раз фильтруется по вышестоящему предикату. Данная ситуация осложняется тем, что подзапрос из-за фильтрации t2.t_id = t1.id не может быть материализован, поскольку при каждой итерации "внешнего цикла" внутренний запрос будет иметь разный результат. 
```sql
explain select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
                                                     QUERY PLAN                                                      
---------------------------------------------------------------------------------------------------------------------
 Seq Scan on t2  (cost=0.00..520980163122.00 rows=419993 width=9)
   Filter: ((day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text)) AND (SubPlan 1))
   SubPlan 1
     ->  Seq Scan on t1  (cost=0.00..208392.00 rows=1 width=4)
           Filter: (t2.t_id = id)
(5 rows)
```
###### Решение
Чтобы ускорить исполнение данного запроса, мы должны воспользоваться соединением с модификацией semi-join. Для этого нам необходимо вместо выражения IN воспользоваться выражением EXISTS. В данном случае они будут эквивалентны, поскольку внутренний запрос всегда будет иметь хотя бы одну результирующую строку, если для какого-либо t1.id будет найден соответствующий t2.t_id. Также стоит отметить, что в результирующей выборке у нас не должны быть значения из таблицы t2, поэтому полусоединение отлично подходит для этого запроса.
```sql
explain (analyze, timing off) select day from t2 WHERE exists (select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
                                               QUERY PLAN                                                
---------------------------------------------------------------------------------------------------------
 Hash Semi Join  (cost=433555.00..643710.72 rows=768589 width=9) (actual rows=768275 loops=1)
   Hash Cond: (t2.t_id = t1.id)
   ->  Seq Scan on t2  (cost=0.00..144372.00 rows=768589 width=13) (actual rows=768275 loops=1)
         Filter: (day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text))
         Rows Removed by Filter: 4231725
   ->  Hash  (cost=269492.00..269492.00 rows=10000000 width=4) (actual rows=10000000 loops=1)
         Buckets: 262144  Batches: 128  Memory Usage: 4807kB
         ->  Seq Scan on t1  (cost=0.00..269492.00 rows=10000000 width=4) (actual rows=10000000 loops=1)
 Planning Time: 0.224 ms
 Execution Time: 6256.975 ms
```
Также мы можем решить данную задачу несколько иначе. Для этого можно даже не переписывать запрос -- нужно лишь навесить индекс на столбец t1.id. В таком случае мы сможем быстро вычислять подзапрос для каждой итерации внешнего цикла благодаря сканированию лишь по индексу (однако в данном случае полусоединение использоваться не будет).  
```sql
CREATE INDEX ON t1(id);
CREATE INDEX

explain (analyze, timing off) select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
                                                     QUERY PLAN                                                      
---------------------------------------------------------------------------------------------------------------------
 Seq Scan on t2  (cost=0.00..22381872.00 rows=384294 width=9) (actual rows=768275 loops=1)
   Filter: ((day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text)) AND (SubPlan 1))
   Rows Removed by Filter: 4231725
   SubPlan 1
     ->  Index Only Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=4) (actual rows=1 loops=768275)
           Index Cond: (id = t2.t_id)
           Heap Fetches: 768275
 Planning Time: 0.310 ms
 Execution Time: 7336.781 ms
```
### Задача №5
ускорить работу "savepoint + update", добиться постоянной во времени производительности (число транзакций в секунду)

Подготавливаем все для теста "savepoint + update" скрипта
```sql
./psql -d test_db -X -q <<'EOF'
create or replace function random(left bigint, right bigint) returns bigint
as $$
 select trunc(random.left + random()*(random.right - random.left))::bigint;
$$                                                
language sql;
EOF
```
```sql
./psql -d test_db -X -q > ./../../generate_100_subtrans.sql <<EOF
select '\\set id random(1,10000000)'
union all
select 'BEGIN;'
union all
select 'savepoint v' || v.id || ';'                || E'\n' 
    || 'update t1 set name = name where id = :id;' || E'\n'
from generate_series(1,100) v(id)
union all
select E'COMMIT;\n' \g (tuples_only=true format=unaligned)
EOF
```
Проверяем работоспособность скрипта:
```sql
 ./psql -d test_db < ./../../generate_100_subtrans.sql > ./../../generate_100_subtrans.lst
tail ./../../generate_100_subtrans.lst 
UPDATE 1
SAVEPOINT
UPDATE 1
SAVEPOINT
UPDATE 1
SAVEPOINT
UPDATE 0
SAVEPOINT
UPDATE 0
COMMIT
```
###### Почему запрос медленный
Чтобы найти и обновить одну или несколько строк, удовлетворяющих предикату, команде UPDATE необходимо перебрать все таплы таблицы, что крайне отрицательно сказывается на скорости обновления. 
```sql
# execute long running transaction
./psql -d test_db -c 'select txid_current(); select pg_sleep(3600);' &
# check using pgbench
./pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f ./../../generate_100_subtrans.sql 2>&1 > ./../../generate_100_subtrans_pgbench.log test_db
progress: 1.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
...
progress: 260.0 s, 1.0 tps, lat 259070.821 ms stddev 0.000, 0 failed
...
progress: 275.0 s, 1.0 tps, lat 274601.157 ms stddev 0.000, 0 failed
...
progress: 280.0 s, 1.0 tps, lat 279190.790 ms stddev 0.000, 0 failed
progress: 281.0 s, 1.0 tps, lat 280381.583 ms stddev 0.000, 0 failed
...
progress: 285.0 s, 1.0 tps, lat 284644.290 ms stddev 0.000, 0 failed
progress: 286.0 s, 1.0 tps, lat 285271.281 ms stddev 0.006, 0 failed
...
progress: 288.0 s, 1.0 tps, lat 287233.632 ms stddev 0.000, 0 failed
...
progress: 290.0 s, 1.0 tps, lat 288977.201 ms stddev 0.006, 0 failed
progress: 291.0 s, 1.0 tps, lat 289980.133 ms stddev 0.004, 0 failed
...
progress: 293.0 s, 1.0 tps, lat 292140.550 ms stddev NaN, 0 failed
...
```
###### Решение
Чтобы ускорить процесс обновления строк, необходимо ускорить процесс их нахождения. Для этого мы опять же можем использовать индекс для столбца t1.id, поскольку поиск изменяемых строк происходит именно по нему. 
```sql
CREATE INDEX ON t1(id);
CREATE INDEX
```
Таким образом, мы значительно ускорим процесс обновления строк.
```sql
# execute long running transaction
./psql -d test_db -c 'select txid_current(); select pg_sleep(3600);' &
# check using pgbench
./pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f ./../../generate_100_subtrans.sql 2>&1 > ./../../generate_100_subtrans_pgbench.log test_db
progress: 1.0 s, 418.0 tps, lat 22.920 ms stddev 17.083, 0 failed
progress: 2.0 s, 519.0 tps, lat 19.241 ms stddev 1.466, 0 failed
progress: 3.0 s, 528.0 tps, lat 18.982 ms stddev 1.578, 0 failed
progress: 4.0 s, 527.0 tps, lat 18.940 ms stddev 1.412, 0 failed
progress: 5.0 s, 516.0 tps, lat 19.359 ms stddev 2.905, 0 failed
progress: 6.0 s, 527.0 tps, lat 18.985 ms stddev 1.529, 0 failed
progress: 7.0 s, 524.0 tps, lat 19.041 ms stddev 1.540, 0 failed
progress: 8.0 s, 520.0 tps, lat 19.171 ms stddev 1.239, 0 failed
progress: 9.0 s, 518.0 tps, lat 19.320 ms stddev 1.415, 0 failed
progress: 10.0 s, 514.0 tps, lat 19.460 ms stddev 3.016, 0 failed
...
progress: 634.0 s, 311.0 tps, lat 29.666 ms stddev 49.570, 0 failed
progress: 635.0 s, 297.0 tps, lat 34.681 ms stddev 63.307, 0 failed
progress: 636.0 s, 371.0 tps, lat 28.183 ms stddev 41.036, 0 failed
progress: 637.0 s, 338.0 tps, lat 27.619 ms stddev 33.601, 0 failed
progress: 638.0 s, 197.0 tps, lat 46.498 ms stddev 98.825, 0 failed
progress: 639.0 s, 284.0 tps, lat 40.791 ms stddev 77.701, 0 failed
progress: 640.0 s, 267.0 tps, lat 37.872 ms stddev 71.533, 0 failed
```
Создание индекса значительно увеличивает tps, однако мы еще не добились постоянной во времени производительности из-за возможного эффекта разбухания таблицы t1 по мере обновления строк. Чем больше у нас будет мертвых строк, тем медленнее будет происходить процесс обновления. Как правило для решения данной задачи необходимо должным образом настроить autovacuum, чтобы успевать очищать старые версии строк по мере их поступления. Чтобы упростить работу autovacuum-у, мы можем воспользоваться механизмом внутристраничной очистки, посольку обновление идет не по индексируемому столбцу t1.id. Для этого установим fillfactor в 50.

ALTER SYSTEM SET autovacuum TO 'on';

ALTER SYSTEM SET autovacuum_work_mem = '256MB';
ALTER SYSTEM SET autovacuum_naptime TO '5s';  
ALTER SYSTEM SET autovacuum_max_workers TO 6;

ALTER TABLE t1 SET (
    autovacuum_vacuum_scale_factor = 0,
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_cost_limit = 6000,
    autovacuum_vacuum_cost_delay = 1,
    fillfactor = 50
);

SELECT pg_reload_conf();



установим fillfactor 

воспользоваться 

механизмом HOT-обновлений, а для этого нужно грамотно настроить автовакуум и fillfactor таблицы, однако с последним возникает некоторая неоднозначность. 




Как мы знаем, скрипт generate_100_subtrans.sql исполняет 100 одинаковых обновлений одной и той же строки в пределах одной транзакции. Тем временем, с помощью расширения pageinspect нетрудно выявить, что один тапл занимает 64 байта. Для этого необходимо всего лишь выполнить при fillfactor = 100 пару обновлений и вычислить разницу в изменении upper значения заголовка страницы (последнюю страницу можно выявлить с помощью pg_class):

Таким образом, чтобы все 100 новых версий одного тапла поместились в одной странице, нам необходим следующий fillfactor:
```math
\mbox{fillfactor} = \frac{64 \cdot 100}{8192 - 28 - 4 * n}
```

```math
\mbox{fillfactor} = 100 - \frac{\mbox{tuple\_size} \cdot n}{8192 - 28 - 4 \cdot n} \cdot
```
В нашем случае n = 100, а tuple_size = 64.
```math
\mbox{fillfactor} = 100 - \frac{64 \cdot 100}{8192 - 28 - 4 \cdot 100} \cdot 100 = 100 - 82,4317362 = 17,5682638 \approx 17
```




где n -- указатель на ...

Таким образом, у нас получится fillfactor = 100 - 80 = 20 (примерно). Но в таком случае наша таблица будет весить в 5 раз больше, хоть мы и сможем практически полностью избавиться от разбухания (при условии, что автовакуум будет грамотно настроен). 

С другой стороны, мы можем оставить fillfactor = 100. В таком случае (напирмер при tps = 500) табилца будет разбухать на 500×8×0,8 = 3200кб = 3.125 мб каждую секунду без вакуума. А когда вакуум вычистит все мертвые таплы, размер таблицы зафиксируется. И он будет оставаться таким же, если вакуум будет, во-первых, при активации вычищать абсолютно все мертвые таплы, во-вторых,




 





ALTER SYSTEM SET autovacuum TO 'on';


ALTER SYSTEM SET autovacuum_naptime TO '5s';   
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 2000;
ALTER SYSTEM SET autovacuum_max_workers = 6; 

ALTER TABLE t1 SET (
    autovacuum_vacuum_scale_factor = 0.001,
    autovacuum_vacuum_cost_limit = 4000,
    fillfactor = 70
);


ALTER TABLE t1 SET (
    fillfactor = 10
);

ALTER TABLE t1 SET (
    fillfactor = 18
);


ALTER TABLE t1 SET (
    fillfactor = 100
);

VACUUM FULL t1;


SELECT 
    relname,
    (n_tup_upd - n_tup_hot_upd) AS a           
FROM pg_stat_user_tables
WHERE relname = 'users';




ALTER SYSTEM SET autovacuum TO 'on';

ALTER SYSTEM SET autovacuum_work_mem = '256MB';
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 2000;
ALTER SYSTEM SET autovacuum_vacuum_cost_delay TO '5ms';
ALTER SYSTEM SET autovacuum_naptime TO '5s';   
ALTER TABLE t1 SET (
    fillfactor = 50,
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_scale_factor = 0, 
    autovacuum_vacuum_cost_limit = 4000 
);






