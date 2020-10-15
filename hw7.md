Для выполнения задания я воспользовалась демо-базой "Авиаперевозки" версии demo-big.

- [x] Создать индекс к какой-либо из таблиц вашей БД
- [x] Прислать текстом результат команды explain, в которой используется данный индекс  
    
**До создания индекса:**
      
      demo=# select count(*) from flights where aircraft_code = '321';
       count
      -------
       12672
      (1 row)

      demo=# explain select count(*) from flights where aircraft_code = '321';
                                             QUERY PLAN
      ----------------------------------------------------------------------------------------
      Finalize Aggregate  (cost=5222.68..5222.69 rows=1 width=8)
        ->  Gather  (cost=5222.57..5222.68 rows=1 width=8)
               Workers Planned: 1
               ->  Partial Aggregate  (cost=4222.57..4222.58 rows=1 width=8)
                     ->  Parallel Seq Scan on flights  (cost=0.00..4203.90 rows=7465 width=0)
                           Filter: (aircraft_code = '321'::bpchar)
      (6 rows)

**После создания индекса:**

      demo=# create index idx_aircraft_code on flights(aircraft_code);
      CREATE INDEX
      demo=# explain select count(*) from flights where aircraft_code = '321';
                                               QUERY PLAN
      --------------------------------------------------------------------------------------------
      Aggregate  (cost=3057.14..3057.15 rows=1 width=8)
        ->  Bitmap Heap Scan on flights  (cost=242.78..3025.41 rows=12691 width=0)
              Recheck Cond: (aircraft_code = '321'::bpchar)
              ->  Bitmap Index Scan on idx_aircraft_code  (cost=0.00..239.60 rows=12691 width=0)
                     Index Cond: (aircraft_code = '321'::bpchar)
      (5 rows)

- [x] Реализовать индекс для полнотекстового поиска

Для тестирования использован [архив рассылки pgsql-hackers](https://oc.postgrespro.ru/index.php/s/fRxTZ0sVfPZzbmd).
Для ускорения полнотекстового поиска можно использовать индексы GiST и GIN. Выбор зависит от вида нагрузки на бд. GIN лучше по скорости и точности поиска. Если данные редко обновляются и важна скорость поиска, следует выбрать GIN. GiST же ищет медленней, но быстро обновляется.  

**До создания индекса:**

      fts=# alter table mail_messages add column tsv tsvector;
      ALTER TABLE
      fts=# update mail_messages
      fts-# set tsv = to_tsvector(subject||' '||author||' '||body_plain);
      UPDATE 356125
      fts=# explain select * from mail_messages where tsv @@ to_tsquery('magic & value');
                                           QUERY PLAN
      ------------------------------------------------------------------------------------
       Gather  (cost=1000.00..193679.52 rows=20 width=1019)
         Workers Planned: 2
         ->  Parallel Seq Scan on mail_messages  (cost=0.00..192677.52 rows=8 width=1019)
               Filter: (tsv @@ to_tsquery('magic & value'::text))
       JIT:
         Functions: 2
         Options: Inlining false, Optimization false, Expressions true, Deforming true
      (7 rows)
      
      fts=# explain (analyze, costs off)
      fts-# select * from mail_messages where tsv @@ to_tsquery('magic & value');
                                              QUERY PLAN
      ------------------------------------------------------------------------------------------
       Gather (actual time=10.520..4716.040 rows=898 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Parallel Seq Scan on mail_messages (actual time=35.253..4669.938 rows=299 loops=3)
               Filter: (tsv @@ to_tsquery('magic & value'::text))
               Rows Removed by Filter: 118409
       Planning Time: 0.191 ms
       Execution Time: 4717.011 ms
      (8 rows)

**Индекс GiST**

      fts=# create index idx_tsv_gist on mail_messages using gist(tsv);
      CREATE INDEX
      fts=# explain (analyze, costs off)
      fts-# select * from mail_messages where tsv @@ to_tsquery('magic & value');
                                                QUERY PLAN
      ----------------------------------------------------------------------------------------------
       Index Scan using idx_tsv_gist on mail_messages (actual time=0.687..246.803 rows=898 loops=1)
         Index Cond: (tsv @@ to_tsquery('magic & value'::text))
         Rows Removed by Index Recheck: 7859
       Planning Time: 0.881 ms
       Execution Time: 246.943 ms
      (5 rows)

**Индекс GIN**

      fts=# drop index idx_tsv_gist;
      DROP INDEX
      fts=# create index idx_tsv_gin on mail_messages using gin(tsv);
      CREATE INDEX
      fts=# explain (analyze, costs off)
      fts-# select * from mail_messages where tsv @@ to_tsquery('magic & value');
                                           QUERY PLAN
      ------------------------------------------------------------------------------------
       Bitmap Heap Scan on mail_messages (actual time=1.164..3.842 rows=898 loops=1)
         Recheck Cond: (tsv @@ to_tsquery('magic & value'::text))
         Heap Blocks: exact=842
         ->  Bitmap Index Scan on idx_tsv_gin (actual time=1.045..1.045 rows=898 loops=1)
               Index Cond: (tsv @@ to_tsquery('magic & value'::text))
       Planning Time: 1.501 ms
       Execution Time: 3.926 ms
      (7 rows)


- [x] Реализовать индекс на часть таблицы или индекс на поле с функцией  

### Индекс на поле с функцией:

Индекс на поле с функцией создается не по самому полю, а по функции с полем таблицы. Чтобы такой индекс использовался, необходимо в запросе использовать функцию с полем таблицы, по которой создан индекс.  

**До создания индекса**

      demo=# explain select * from tickets where passenger_name = 'ELENA BELOVA';
                                        QUERY PLAN
      ------------------------------------------------------------------------------
       Gather  (cost=1000.00..65805.04 rows=262 width=104)
         Workers Planned: 2
         ->  Parallel Seq Scan on tickets  (cost=0.00..64778.84 rows=109 width=104)
               Filter: (passenger_name = 'ELENA BELOVA'::text)
      (4 rows)
      
**После создания индекса**
      
      demo=#  create index idx_passenger_name_lower on tickets(lower(passenger_name));
      CREATE INDEX
      demo=# explain select * from tickets where lower(passenger_name) = 'elena belova';
                                               QUERY PLAN
      ---------------------------------------------------------------------------------------------
       Bitmap Heap Scan on tickets  (cost=370.73..32306.35 rows=14749 width=104)
         Recheck Cond: (lower(passenger_name) = 'elena belova'::text)
         ->  Bitmap Index Scan on idx_passenger_name_lower  (cost=0.00..367.05 rows=14749 width=0)
               Index Cond: (lower(passenger_name) = 'elena belova'::text)
      (4 rows)
      
      demo=# explain select lower(passenger_name) from tickets where lower(passenger_name) = 'elena belova';
                                               QUERY PLAN
      ---------------------------------------------------------------------------------------------
       Bitmap Heap Scan on tickets  (cost=370.73..32343.22 rows=14749 width=32)
         Recheck Cond: (lower(passenger_name) = 'elena belova'::text)
         ->  Bitmap Index Scan on idx_passenger_name_lower  (cost=0.00..367.05 rows=14749 width=0)
               Index Cond: (lower(passenger_name) = 'elena belova'::text)
      (4 rows)

### Индекс на часть таблицы

Индекс на часть таблицы имеет смысл строить при сильной неравномерности распределения данных в таблице. Редкие значения имеет смысл искать по индексу.

**До создания индекса**

      demo=# select fare_conditions, count(*) from ticket_flights group by fare_conditions;
       fare_conditions |  count
      -----------------+---------
       Business        |  859656
       Comfort         |  139965
       Economy         | 7392231
      (3 rows)
      
      demo=# explain select * from ticket_flights where fare_conditions = 'Comfort';
                                            QUERY PLAN
      ---------------------------------------------------------------------------------------
       Gather  (cost=1000.00..128431.16 rows=137906 width=32)
         Workers Planned: 2
         ->  Parallel Seq Scan on ticket_flights  (cost=0.00..113640.56 rows=57461 width=32)
               Filter: ((fare_conditions)::text = 'Comfort'::text)
       JIT:
         Functions: 2
         Options: Inlining false, Optimization false, Expressions true, Deforming true
      (7 rows)

**После создания индекса**

      demo=# create index idx_conditions_comfort on ticket_flights(fare_conditions) where fare_conditions = 'Comfort';
      CREATE INDEX
      demo=# explain select * from ticket_flights where fare_conditions = 'Comfort';
                                                    QUERY PLAN
      -------------------------------------------------------------------------------------------------------
       Index Scan using idx_conditions_comfort on ticket_flights  (cost=0.42..69746.71 rows=137906 width=32)
      (1 row)
      
      demo=# explain select * from ticket_flights where fare_conditions = 'Business';
                                         QUERY PLAN
      ---------------------------------------------------------------------------------
       Seq Scan on ticket_flights  (cost=0.00..174831.15 rows=871074 width=32)
         Filter: ((fare_conditions)::text = 'Business'::text)
       JIT:
         Functions: 2
         Options: Inlining false, Optimization false, Expressions true, Deforming true
      (5 rows)

- [x] Создать индекс на несколько полей

Порядок полей, добавленных в индекс, имеет значение. Индекс будет использоваться если в условии указана часть полей, начиная с первого, либо все поля, участвующие в индексе. Если на первое поле не будет наложено условие, индекс использоваться не будет.

**До создания индекса**

      demo=# select status, count(*) from flights where departure_airport = 'DME' group by status;
        status   | count
      -----------+-------
       Arrived   | 19275
       Cancelled |    32
       Delayed   |     6
       Departed  |     7
       On Time   |    45
       Scheduled |  1510
      (6 rows)
      
      demo=# explain select * from flights where departure_airport = 'DME' and status = 'Scheduled';
                                                 QUERY PLAN
      ------------------------------------------------------------------------------------------------
       Gather  (cost=1000.00..5673.19 rows=1533 width=63)
         Workers Planned: 1
         ->  Parallel Seq Scan on flights  (cost=0.00..4519.89 rows=902 width=63)
               Filter: ((departure_airport = 'DME'::bpchar) AND ((status)::text = 'Scheduled'::text))
      (4 rows)
      
      demo=# explain select * from flights where departure_airport = 'DME';
                                QUERY PLAN
      ---------------------------------------------------------------
       Seq Scan on flights  (cost=0.00..5309.84 rows=21258 width=63)
         Filter: (departure_airport = 'DME'::bpchar)
      (2 rows)


**После создания индекса**

      demo=# create index idx_departure_airport_status on flights(departure_airport, status);
      CREATE INDEX
      demo=# explain select * from flights where departure_airport = 'DME' and status = 'Scheduled';
                                                   QUERY PLAN
      ----------------------------------------------------------------------------------------------------
       Bitmap Heap Scan on flights  (cost=40.13..2416.07 rows=1533 width=63)
         Recheck Cond: ((departure_airport = 'DME'::bpchar) AND ((status)::text = 'Scheduled'::text))
         ->  Bitmap Index Scan on idx_departure_airport_status  (cost=0.00..39.75 rows=1533 width=0)
               Index Cond: ((departure_airport = 'DME'::bpchar) AND ((status)::text = 'Scheduled'::text))
      (4 rows)
      
      demo=# explain select * from flights where departure_airport = 'DME';
                                                 QUERY PLAN
      -------------------------------------------------------------------------------------------------
       Bitmap Heap Scan on flights  (cost=497.17..3386.89 rows=21258 width=63)
         Recheck Cond: (departure_airport = 'DME'::bpchar)
         ->  Bitmap Index Scan on idx_departure_airport_status  (cost=0.00..491.86 rows=21258 width=0)
               Index Cond: (departure_airport = 'DME'::bpchar)
      (4 rows)
      
      demo=# explain select * from flights where status = 'Scheduled';
                                QUERY PLAN
      ---------------------------------------------------------------
       Seq Scan on flights  (cost=0.00..5309.84 rows=15499 width=63)
         Filter: ((status)::text = 'Scheduled'::text)
      (2 rows)
     
