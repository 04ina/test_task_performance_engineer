# test_task_performance_engineer

### Задача №1
ускорить простой запроc, добиться времени выполнения < 10ms
```sql
select name from t1 where id = 50000;
```
###### Почему запрос медленный
Чтобы найти и вывести нужное нам значение name, постгресу необходимо с помощью полного перебора посредством seq scan найти все строки, id которых равно значению 50000. Поскольку существует всего одна строка с id=50000, использование полного перебора неоправдано.
```sql
explain analyze select name from t1 where id = 50000;
                                             QUERY PLAN                                             
----------------------------------------------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..294492.00 rows=1 width=30) (actual time=5.268..726.719 rows=1 loops=1)
   Filter: (id = 50000)
   Rows Removed by Filter: 9999999
 Planning Time: 0.041 ms
 Execution Time: 726.743 ms
```
###### Решение
Для увеличения скорости выполнения данного запроса необходимо "навестить" индекс на столбец id. Это позволит производить выборку с высокой селективностью по id значительно быстрее. Для наших целей подойдет как btree, так и hash индексы. 
```sql
CREATE INDEX ON t1(id);
CREATE INDEX

explain analyze select name from t1 where id = 50000;
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Index Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=30) (actual time=0.023..0.027 rows=1 loops=1)
   Index Cond: (id = 50000)
 Planning Time: 0.067 ms
 Execution Time: 0.049 ms
```
### Задача №2
ускорить запрос "max + left join", добиться времени выполнения < 10ms
```sql
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
###### Почему запрос медленный
Данный запрос выполняется медленно вследствие использования соединения, которое, несмотря на фильтрацию по столбцу name, сопутствуется высокими накладными расходами.
```sql
explain analyze select t2.day from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
                                                        QUERY PLAN                                                        
--------------------------------------------------------------------------------------------------------------------------
 Hash Left Join  (cost=304435.76..459286.06 rows=5000000 width=9) (actual time=2750.594..23434.829 rows=5000000 loops=1)
   Hash Cond: (t2.t_id = t1.id)
   ->  Seq Scan on t2  (cost=0.00..81872.00 rows=5000000 width=13) (actual time=0.007..6402.769 rows=5000000 loops=1)
   ->  Hash  (cost=294492.00..294492.00 rows=606061 width=4) (actual time=2750.354..2750.359 rows=625787 loops=1)
         Buckets: 262144  Batches: 4  Memory Usage: 7553kB
         ->  Seq Scan on t1  (cost=0.00..294492.00 rows=606061 width=4) (actual time=0.038..1871.625 rows=625787 loops=1)
               Filter: (name ~~ 'a%'::text)
               Rows Removed by Filter: 9374213
 Planning Time: 0.199 ms
 Execution Time: 29671.847 ms
```
###### Решение
Нетрудно догадаться, что данный запрос на самом деле не нуждается в левом внешнем соединении -- при любых предикатах соединения левая результирующая часть будет состоять из 5000000 значений левого отношения t2 (выражения t2.t_id = t1.id и t1.name like 'a%' лишены всякого смысла вместе с самим соединением). Тем временем, значение max(t2.day) может варьироваться только в соответсвтии с левой частью таблицы соединения -- правая часть для max(t2.day) будет фиктивной. В связи с этим мы можем убрать любое упоминание таблицы t1 в запросе:
```sql
explain analyze select max(t2.day) from t2;
                                                     QUERY PLAN                                                      
---------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=94372.00..94372.01 rows=1 width=32) (actual time=13229.088..13229.094 rows=1 loops=1)
   ->  Seq Scan on t2  (cost=0.00..81872.00 rows=5000000 width=9) (actual time=0.012..6250.995 rows=5000000 loops=1)
 Planning Time: 0.088 ms
 Execution Time: 13229.126 ms
``` 

Чтобы избежать полного перебора и последующей агрегации, навесим индекс на столбец day:

```sql
CREATE INDEX ON t2(day);
CREATE INDEX
explain analyze select max(t2.day) from t2;
                                                                      QUERY PLAN                                                                      
------------------------------------------------------------------------------------------------------------------------------------------------------
 Result  (cost=0.45..0.46 rows=1 width=32) (actual time=0.030..0.039 rows=1 loops=1)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.43..0.45 rows=1 width=9) (actual time=0.021..0.027 rows=1 loops=1)
           ->  Index Only Scan Backward using t2_day_idx on t2  (cost=0.43..104784.44 rows=5000000 width=9) (actual time=0.017..0.018 rows=1 loops=1)
                 Index Cond: (day IS NOT NULL)
                 Heap Fetches: 0
 Planning Time: 0.078 ms
 Execution Time: 0.060 ms
```
### Задача №3
ускорить запрос "anti-join", добиться времени выполнения < 10sec
```sql
select day from t2 where t_id not in ( select t1.id from t1 );
```
###### Почему запрос медленный
Данный запрос для каждой записи из t2 проверяет, не содержится ли t2.t_id в столбце t1.id таблицы t1. Стоит отметить, что планировщик для подзапроса использует материализацию, чтобы по нескольку раз не считывать таплы из таблицы t1; однако сильно запросу это не помогает. 
```sql
explain select day from t2 where t_id not in ( select t1.id from t1 );
                                 QUERY PLAN                                 
----------------------------------------------------------------------------
 Seq Scan on t2  (cost=0.00..958887594372.00 rows=2500000 width=9)
   Filter: (NOT (SubPlan 1))
   SubPlan 1
     ->  Materialize  (cost=0.00..358555.00 rows=10000000 width=4)
           ->  Seq Scan on t1  (cost=0.00..269492.00 rows=10000000 width=4)
```
###### Решение
Данный запрос можно значительно улучшить, если вместо not in воспользоваться NOT EXISTIS, что спровоцирует планировщик использовать вместо подзапроса антисоединение, которое позволит работать лишь со строками, для которых не нашлось соответствия.
```sql
explain (analyze,timing off) SELECT day FROM t2 WHERE NOT EXISTS (SELECT * FROM t1 WHERE t1.id = t2.t_id);
                                              QUERY PLAN                                               
-------------------------------------------------------------------------------------------------------
 Hash Right Anti Join  (cost=168787.00..678320.00 rows=1 width=9) (actual rows=0 loops=1)
   Hash Cond: (t1.id = t2.t_id)
   ->  Seq Scan on t1  (cost=0.00..269492.00 rows=10000000 width=4) (actual rows=10000000 loops=1)
   ->  Hash  (cost=81872.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
         Buckets: 262144  Batches: 64  Memory Usage: 5724kB
         ->  Seq Scan on t2  (cost=0.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
 Planning Time: 0.095 ms
 Execution Time: 5155.374 ms
```
Также мы можем воспользоваться иной конструкцией запроса, которая будет эквивалентна вышеописанному варианту благодаря преобразованиям на уровне оптимизатора.
```sql
explain (analyze, timing off) SELECT t2.day FROM t2 LEFT JOIN t1 ON t2.t_id = t1.id WHERE t1.id IS NULL;
                                              QUERY PLAN                                               
-------------------------------------------------------------------------------------------------------
 Hash Right Anti Join  (cost=168787.00..678320.00 rows=1 width=9) (actual rows=0 loops=1)
   Hash Cond: (t1.id = t2.t_id)
   ->  Seq Scan on t1  (cost=0.00..269492.00 rows=10000000 width=4) (actual rows=10000000 loops=1)
   ->  Hash  (cost=81872.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
         Buckets: 262144  Batches: 64  Memory Usage: 5724kB
         ->  Seq Scan on t2  (cost=0.00..81872.00 rows=5000000 width=13) (actual rows=5000000 loops=1)
 Planning Time: 0.084 ms
 Execution Time: 5167.781 ms
```
### Задача №4
ускорить запрос "semi-join", добиться времени выполнения < 10sec
```sql
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
###### Почему запрос медленный
Ситуация с данным запросом похожа на ситуацию с предыдущим -- у нас имеется оператор in, который требует полного вычисления внутреннего запроса для проверки фильтра выборки; однако для этого необходимо провести полный перебор. Также в отличае от предыдущего запроса у нас не используется материализация, поскольку мы должны во внутреннем запросе найти всего лишь одну строку, которая может меняться при разных итерациях внешнего цикла перебора.
```sql
explain select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
                                                     QUERY PLAN                                                      
---------------------------------------------------------------------------------------------------------------------
 Seq Scan on t2  (cost=0.00..736230163122.00 rows=384294 width=9)
   Filter: ((day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text)) AND (SubPlan 1))
   SubPlan 1
     ->  Seq Scan on t1  (cost=0.00..294492.00 rows=1 width=4)
           Filter: (t2.t_id = id)
```
###### Решение
Чтобы улучшить данный запрос, необходимо воспользоваться вместо in оператором EXISTS, который позволит нам закончить перебор внутреннего запроса при нахождении первого соответствия. Реализация этой логики производится посредством полусоединения. 
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
Также мы можем решить данную задачу несколько иначе. Для этого можно даже не переписывать запрос -- нужно лишь навесить индекс на столбец t1.id. Это позволит нам быстро вычислять внутренний запрос.  
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
###### Почему запрос медленный
Поскольку в таблице t1 изначально очень много строк, для изменения конкретной необходимо перебрать абсолютно все строки, что крайне отрицательно сказывается на скорости обновления. 
###### Решение
Чтобы ускорить процесс изменения строк, необходимо ускорить процесс их нахождения. Для этого мы опять же можем использовать индекс для столбца t1.id, поскольку поиск изменяемой строки происходит именно по нему. 
```sql
CREATE INDEX ON t1(id);
CREATE INDEX
```
Таким образом мы улучшим скорость обновления строк
```sql
./pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql 2>&1 > generate_100_subtrans_pgbench.log test_db2
progress: 1.0 s, 552.0 tps, lat 17.595 ms stddev 11.975, 0 failed
progress: 2.0 s, 631.0 tps, lat 15.811 ms stddev 2.074, 0 failed
progress: 3.0 s, 634.0 tps, lat 15.724 ms stddev 2.003, 0 failed
progress: 4.0 s, 626.0 tps, lat 15.943 ms stddev 3.125, 0 failed
progress: 5.0 s, 637.0 tps, lat 15.709 ms stddev 2.076, 0 failed
progress: 6.0 s, 642.0 tps, lat 15.594 ms stddev 1.965, 0 failed
progress: 7.0 s, 635.0 tps, lat 15.632 ms stddev 2.059, 0 failed
progress: 8.0 s, 642.0 tps, lat 15.616 ms stddev 1.917, 0 failed
progress: 9.0 s, 630.0 tps, lat 15.867 ms stddev 2.905, 0 failed
progress: 10.0 s, 635.0 tps, lat 15.707 ms stddev 1.814, 0 failed
progress: 11.0 s, 641.0 tps, lat 15.636 ms stddev 2.029, 0 failed
```
Стоит отметить, что, навесив индекс, мы не сможем выполнять hot очистку, вследствие чего риск разбухания таблицы t1 может быть увеличен.
