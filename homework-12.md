### **Домашнее задание 12**
#### *Работа с join'ами, статистикой*

> Необходимо:
> 1. Реализовать прямое соединение двух или более таблиц
  ```sql
  postgres=# select o.order_id, c.city_name as city, b.cname as buyer, o.order_sum 
    from tbl_orders o, tbl_city c, tbl_names b 
    where c.city_id=o.city_id
          and b.pkey=o.buyer_id
    limit 10; 
   order_id |  city  | buyer  |     order_sum      
  ----------+--------+--------+--------------------
          1 | Moscow | Dima   | 34.331113458899445
          2 | Minsk  | Masha  |  80.76671442559271
          3 | Omsk   | Sasha  |   79.0007158733092
          4 | Minsk  | Mickle |  47.75236914290505
          5 | Omsk   | Vanya  |  70.69224304207677
          6 | Minsk  | Dima   |  65.51118196232768
          7 | Moscow | Dima   |  88.92297902272955
          8 | Omsk   | Sasha  | 42.585477660987124
          9 | Omsk   | Masha  | 124.68334499402735
         10 | Omsk   | Dima   | 136.18656019586598
  (10 rows)
  
  postgres=# 
  postgres=# explain analyze                                                       
  select o.order_id, c.city_name as city, b.cname as buyer, o.order_sum 
    from tbl_orders o, tbl_city c, tbl_names b 
    where c.city_id=o.city_id
          and b.pkey=o.buyer_id;
                                                                    QUERY PLAN                                                                  
  ----------------------------------------------------------------------------------------------------------------------------------------------
   Merge Join  (cost=121925.86..206642.82 rows=4733925 width=80) (actual time=891.246..1534.096 rows=745500 loops=1)
     Merge Cond: (((c.city_id)::double precision) = o.city_id)
     ->  Sort  (cost=88.17..91.35 rows=1270 width=36) (actual time=33.664..33.667 rows=4 loops=1)
           Sort Key: ((c.city_id)::double precision)
           Sort Method: quicksort  Memory: 25kB
           ->  Seq Scan on tbl_city c  (cost=0.00..22.70 rows=1270 width=36) (actual time=33.635..33.641 rows=4 loops=1)
     ->  Materialize  (cost=121837.69..125565.19 rows=745500 width=56) (actual time=857.534..1282.511 rows=745500 loops=1)
           ->  Sort  (cost=121837.69..123701.44 rows=745500 width=56) (actual time=857.527..1001.326 rows=745500 loops=1)
                 Sort Key: o.city_id
                 Sort Method: external merge  Disk: 25248kB
                 ->  Hash Join  (cost=1.18..23638.68 rows=745500 width=56) (actual time=0.054..405.735 rows=745500 loops=1)
                       Hash Cond: (o.buyer_id = (b.pkey)::double precision)
                       ->  Seq Scan on tbl_orders o  (cost=0.00..12455.00 rows=745500 width=32) (actual time=0.013..93.037 rows=745500 loops=1)
                       ->  Hash  (cost=1.08..1.08 rows=8 width=36) (actual time=0.023..0.024 rows=8 loops=1)
                             Buckets: 1024  Batches: 1  Memory Usage: 9kB
                             ->  Seq Scan on tbl_names b  (cost=0.00..1.08 rows=8 width=36) (actual time=0.011..0.013 rows=8 loops=1)
   Planning Time: 0.436 ms
   JIT:
     Functions: 20
     Options: Inlining false, Optimization false, Expressions true, Deforming true
     Timing: Generation 5.013 ms, Inlining 0.000 ms, Optimization 2.340 ms, Emission 31.309 ms, Total 38.663 ms
   Execution Time: 1595.112 ms
  (22 rows)
  
  postgres=# 
  ```

> 2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
  ```sql
  postgres=# select *
      from tbl_city c left join tbl_orders o on c.city_id=o.city_id limit 10;             
   city_id | city_name | order_id | buyer_id | city_id |     order_sum      
  ---------+-----------+----------+----------+---------+--------------------
         1 | Moscow    |        1 |        4 |       1 | 34.331113458899445
         3 | Minsk     |        2 |        1 |       3 |  80.76671442559271
         4 | Omsk      |        3 |        3 |       4 |   79.0007158733092
         3 | Minsk     |        4 |        8 |       3 |  47.75236914290505
         4 | Omsk      |        5 |        6 |       4 |  70.69224304207677
         3 | Minsk     |        6 |        4 |       3 |  65.51118196232768
         1 | Moscow    |        7 |        4 |       1 |  88.92297902272955
         4 | Omsk      |        8 |        3 |       4 | 42.585477660987124
         4 | Omsk      |        9 |        1 |       4 | 124.68334499402735
         4 | Omsk      |       10 |        4 |       4 | 136.18656019586598
  (10 rows)
  
  postgres=# 
  postgres=# explain analyze                                                               
  select *
      from tbl_city c left join tbl_orders o on c.city_id=o.city_id;      
                                                                 QUERY PLAN                                                               
  ----------------------------------------------------------------------------------------------------------------------------------------
   Merge Left Join  (cost=103098.18..187815.14 rows=4733925 width=68) (actual time=526.519..1160.331 rows=745500 loops=1)
     Merge Cond: (((c.city_id)::double precision) = o.city_id)
     ->  Sort  (cost=88.17..91.35 rows=1270 width=36) (actual time=11.217..11.221 rows=4 loops=1)
           Sort Key: ((c.city_id)::double precision)
           Sort Method: quicksort  Memory: 25kB
           ->  Seq Scan on tbl_city c  (cost=0.00..22.70 rows=1270 width=36) (actual time=11.196..11.201 rows=4 loops=1)
     ->  Materialize  (cost=103010.01..106737.51 rows=745500 width=32) (actual time=515.258..942.647 rows=745500 loops=1)
           ->  Sort  (cost=103010.01..104873.76 rows=745500 width=32) (actual time=515.252..658.590 rows=745500 loops=1)
                 Sort Key: o.city_id
                 Sort Method: external merge  Disk: 26840kB
                 ->  Seq Scan on tbl_orders o  (cost=0.00..12455.00 rows=745500 width=32) (actual time=0.012..96.132 rows=745500 loops=1)
   Planning Time: 0.275 ms
   JIT:
     Functions: 9
     Options: Inlining false, Optimization false, Expressions true, Deforming true
     Timing: Generation 2.178 ms, Inlining 0.000 ms, Optimization 0.762 ms, Emission 10.439 ms, Total 13.379 ms
   Execution Time: 1219.459 ms
  (17 rows)
  
  postgres=# 
  ```

> 3. Реализовать кросс соединение двух или более таблиц
  ```sql
  postgres=# 
  postgres=# select *
      from tbl_city c cross join tbl_orders o
          limit 10;       
   city_id | city_name | order_id | buyer_id | city_id |     order_sum      
  ---------+-----------+----------+----------+---------+--------------------
         1 | Moscow    |        1 |        4 |       1 | 34.331113458899445
         2 | Kemerovo  |        1 |        4 |       1 | 34.331113458899445
         3 | Minsk     |        1 |        4 |       1 | 34.331113458899445
         4 | Omsk      |        1 |        4 |       1 | 34.331113458899445
         1 | Moscow    |        2 |        1 |       3 |  80.76671442559271
         2 | Kemerovo  |        2 |        1 |       3 |  80.76671442559271
         3 | Minsk     |        2 |        1 |       3 |  80.76671442559271
         4 | Omsk      |        2 |        1 |       3 |  80.76671442559271
         1 | Moscow    |        3 |        3 |       4 |   79.0007158733092
         2 | Kemerovo  |        3 |        3 |       4 |   79.0007158733092
  (10 rows)
  
  postgres=# 
  postgres=# explain analyze                                                               
  select *
      from tbl_city c cross join tbl_orders o;
                                                           QUERY PLAN                                                         
  ----------------------------------------------------------------------------------------------------------------------------
   Nested Loop  (cost=0.00..11847293.38 rows=946785000 width=68) (actual time=174.246..1139.354 rows=2982000 loops=1)
     ->  Seq Scan on tbl_orders o  (cost=0.00..12455.00 rows=745500 width=32) (actual time=0.029..86.036 rows=745500 loops=1)
     ->  Materialize  (cost=0.00..29.05 rows=1270 width=36) (actual time=0.000..0.000 rows=4 loops=745500)
           ->  Seq Scan on tbl_city c  (cost=0.00..22.70 rows=1270 width=36) (actual time=0.012..0.015 rows=4 loops=1)
   Planning Time: 0.212 ms
   JIT:
     Functions: 3
     Options: Inlining true, Optimization true, Expressions true, Deforming true
     Timing: Generation 1.243 ms, Inlining 101.821 ms, Optimization 47.773 ms, Emission 24.596 ms, Total 175.432 ms
   Execution Time: 1335.072 ms
  (10 rows)
  
  postgres=# 
  ```

> 4. Реализовать полное соединение двух или более таблиц
  ```sql
  postgres=# 
  postgres=# select *
      from tbl_city c full outer join tbl_orders o on c.city_id=o.city_id
          limit 10;               
   city_id | city_name | order_id | buyer_id | city_id |     order_sum      
  ---------+-----------+----------+----------+---------+--------------------
         1 | Moscow    |        1 |        4 |       1 | 34.331113458899445
         3 | Minsk     |        2 |        1 |       3 |  80.76671442559271
         4 | Omsk      |        3 |        3 |       4 |   79.0007158733092
         3 | Minsk     |        4 |        8 |       3 |  47.75236914290505
         4 | Omsk      |        5 |        6 |       4 |  70.69224304207677
         3 | Minsk     |        6 |        4 |       3 |  65.51118196232768
         1 | Moscow    |        7 |        4 |       1 |  88.92297902272955
         4 | Omsk      |        8 |        3 |       4 | 42.585477660987124
         4 | Omsk      |        9 |        1 |       4 | 124.68334499402735
         4 | Omsk      |       10 |        4 |       4 | 136.18656019586598
  (10 rows)
  
  postgres=# 
  postgres=# explain analyze                                                               
  select *
      from tbl_city c full outer join tbl_orders o on c.city_id=o.city_id;
                                                                 QUERY PLAN                                                               
  ----------------------------------------------------------------------------------------------------------------------------------------
   Merge Full Join  (cost=103098.18..187815.14 rows=4733925 width=68) (actual time=520.312..1151.898 rows=745500 loops=1)
     Merge Cond: (((c.city_id)::double precision) = o.city_id)
     ->  Sort  (cost=88.17..91.35 rows=1270 width=36) (actual time=10.743..10.747 rows=4 loops=1)
           Sort Key: ((c.city_id)::double precision)
           Sort Method: quicksort  Memory: 25kB
           ->  Seq Scan on tbl_city c  (cost=0.00..22.70 rows=1270 width=36) (actual time=10.722..10.726 rows=4 loops=1)
     ->  Materialize  (cost=103010.01..106737.51 rows=745500 width=32) (actual time=509.525..934.728 rows=745500 loops=1)
           ->  Sort  (cost=103010.01..104873.76 rows=745500 width=32) (actual time=509.519..651.423 rows=745500 loops=1)
                 Sort Key: o.city_id
                 Sort Method: external merge  Disk: 26840kB
                 ->  Seq Scan on tbl_orders o  (cost=0.00..12455.00 rows=745500 width=32) (actual time=0.013..94.654 rows=745500 loops=1)
   Planning Time: 0.247 ms
   JIT:
     Functions: 9
     Options: Inlining false, Optimization false, Expressions true, Deforming true
     Timing: Generation 3.076 ms, Inlining 0.000 ms, Optimization 1.046 ms, Emission 9.673 ms, Total 13.796 ms
   Execution Time: 1211.928 ms
  (17 rows)
  
  postgres=# 
  ```

> 5. Реализовать запрос, в котором будут использованы разные типы соединений
  ```sql
  postgres=# 
  postgres=# select *
    from tbl_orders o left join tbl_city c on c.city_id=o.city_id right join tbl_names b on b.pkey=o.buyer_id  
    limit 10;     
   order_id | buyer_id | city_id |     order_sum      | city_id | city_name | pkey | cname 
  ----------+----------+---------+--------------------+---------+-----------+------+-------
          2 |        1 |       3 |  80.76671442559271 |       3 | Minsk     |    1 | Masha
          9 |        1 |       4 | 124.68334499402735 |       4 | Omsk      |    1 | Masha
         11 |        1 |       3 |  58.17465110258638 |       3 | Minsk     |    1 | Masha
         21 |        1 |       2 | 257.66966162152306 |       2 | Kemerovo  |    1 | Masha
         23 |        1 |       3 | 151.17088910868824 |       3 | Minsk     |    1 | Masha
         26 |        1 |       1 | 199.68429957928865 |       1 | Moscow    |    1 | Masha
         50 |        1 |       2 |  451.4456596267322 |       2 | Kemerovo  |    1 | Masha
         51 |        1 |       3 |  524.7015096701587 |       3 | Minsk     |    1 | Masha
         58 |        1 |       3 |  438.2363685287484 |       3 | Minsk     |    1 | Masha
         60 |        1 |       1 |  160.6898466086346 |       1 | Moscow    |    1 | Masha
  (10 rows)
  
  postgres=# 
  postgres=# explain analyze                                                               
  select *
    from tbl_orders o left join tbl_city c on c.city_id=o.city_id right join tbl_names b on b.pkey=o.buyer_id  
  postgres-# ;
                                                                    QUERY PLAN                                                                  
  ----------------------------------------------------------------------------------------------------------------------------------------------
   Merge Right Join  (cost=157601.36..242318.32 rows=4733925 width=104) (actual time=953.603..1635.072 rows=745500 loops=1)
     Merge Cond: (((c.city_id)::double precision) = o.city_id)
     ->  Sort  (cost=88.17..91.35 rows=1270 width=36) (actual time=38.459..38.468 rows=4 loops=1)
           Sort Key: ((c.city_id)::double precision)
           Sort Method: quicksort  Memory: 25kB
           ->  Seq Scan on tbl_city c  (cost=0.00..22.70 rows=1270 width=36) (actual time=38.439..38.444 rows=4 loops=1)
     ->  Materialize  (cost=157513.19..161240.69 rows=745500 width=68) (actual time=915.100..1364.439 rows=745500 loops=1)
           ->  Sort  (cost=157513.19..159376.94 rows=745500 width=68) (actual time=915.095..1071.868 rows=745500 loops=1)
                 Sort Key: o.city_id
                 Sort Method: external merge  Disk: 38472kB
                 ->  Hash Right Join  (cost=1.18..23638.68 rows=745500 width=68) (actual time=0.046..416.550 rows=745500 loops=1)
                       Hash Cond: (o.buyer_id = (b.pkey)::double precision)
                       ->  Seq Scan on tbl_orders o  (cost=0.00..12455.00 rows=745500 width=32) (actual time=0.005..89.905 rows=745500 loops=1)
                       ->  Hash  (cost=1.08..1.08 rows=8 width=36) (actual time=0.023..0.024 rows=8 loops=1)
                             Buckets: 1024  Batches: 1  Memory Usage: 9kB
                             ->  Seq Scan on tbl_names b  (cost=0.00..1.08 rows=8 width=36) (actual time=0.012..0.014 rows=8 loops=1)
   Planning Time: 0.684 ms
   JIT:
     Functions: 19
     Options: Inlining false, Optimization false, Expressions true, Deforming true
     Timing: Generation 4.869 ms, Inlining 0.000 ms, Optimization 2.512 ms, Emission 35.940 ms, Total 43.321 ms
   Execution Time: 1700.036 ms
  (22 rows)
  
  postgres=#
  ```

> 6. Сделать комментарии на каждый запрос
  * п.1 Здесь показываются все заказы у который есть покупатель и город покупки+расшифровка названия города и имя покупателя.
  * п.2 Здесь для списка городов выводятся заказы. Если в заказах не указан город, то такой заказ не будет показан.
  * п.3 Здесь выводится Декартово произведение, другими словами идет перемножения количества записей из одной таблицы на количество записей из другой таблицы
  * п.4 Будут выведены записи из обоих таблиц в связке по ключу. Но если данные не будут найдены в одной из таблиц, то будет пустое значение
  * п.5 Здесь показываются все заказы у который есть покупатель и город покупки+расшифровка названия города и имя покупателя.
    

> 7. К работе приложить структуру таблиц, для которых выполнялись соединения
  ```sql
  postgres=# create table tbl_names (pkey int, cname text);
  CREATE TABLE
  postgres=# 
  postgres=# insert into tbl_names values (1,'Masha');
  insert into tbl_names values (2,'Dasha');
  insert into tbl_names values (3,'Sasha');
  insert into tbl_names values (4,'Dima');
  insert into tbl_names values (5,'Tolya');
  insert into tbl_names values (6,'Vanya');
  insert into tbl_names values (7,'Igor');
  insert into tbl_names values (8,'Mickle');
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  postgres=# create table tbl_city (city_id int, city_name text);
  CREATE TABLE
  insert into tbl_city values (1,'Moscow');
  insert into tbl_city values (2,'Kemerovo');
  insert into tbl_city values (3,'Minsk');
  insert into tbl_city values (4,'Omsk');
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  INSERT 0 1
  postgres=# 
  postgres=# create table tbl_orders as
  select trunc(generate_series(1,500)) as order_id,
         trunc((random() * 8+1)) as buyer_id,
         trunc((random() * 4+1)) as city_id,
         generate_series(10,1500)*(random()*8+1) as order_sum                                                                                                                                                                      
  from generate_series(1, 500);
  SELECT 745500
  postgres=# 
  postgres=# analyze tbl_orders;
  ANALYZE
  postgres=# 
  ```
