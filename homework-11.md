### **Домашнее задание 11**
#### *Работа с индексами*

> Создать индексы на БД, которые ускорят доступ к данным.
> Необходимо:
> 1. Создать индекс к какой-либо из таблиц вашей БД
  ```sql
  postgres=# create table otus_edu as 
  select generate_series as pkey, generate_series::text || (random() * 10)::text as id, (array['text1', 'text2', 'text3'])[floor(random() * 6)] as info from generate_series(1, 100000);
  postgres=# 
  postgres=# explain analyse
  select * from otus_edu where info = 'text1';
                                                  QUERY PLAN                                                 
  -----------------------------------------------------------------------------------------------------------
   Seq Scan on otus_edu  (cost=0.00..2072.00 rows=32943 width=33) (actual time=0.021..42.989 rows=33087 loops=1)
     Filter: (info = 'text1'::text)
     Rows Removed by Filter: 66913
   Planning Time: 0.824 ms
   Execution Time: 47.361 ms
  (5 rows)
  
  postgres=#
  postgres=# create index otus_edu_pkey_idx on otus_edu(pkey);                    
  CREATE INDEX
  postgres=# 
  ```
> 2. Прислать текстом результат команды explain, в которой используется данный индекс
  ```sql
  postgres=# 
  postgres=# explain analyse
  select * from otus_edu where pkey = 3;
                                                       QUERY PLAN                                                      
  ---------------------------------------------------------------------------------------------------------------------
   Index Scan using otus_edu_pkey_idx on otus_edu  (cost=0.29..8.31 rows=1 width=33) (actual time=0.083..0.086 rows=1 loops=1)
     Index Cond: (pkey = 3)
   Planning Time: 0.510 ms
   Execution Time: 0.122 ms
  (4 rows)
  
  postgres=# 
  ```

> 3. Реализовать индекс для полнотекстового поиска
  ```sql
  postgres=# alter table otus_edu add column info_tsv tsvector;
  ALTER TABLE
  postgres=# update otus_edu set info_tsv=to_tsvector(info);
  UPDATE 100000
  postgres=# 
  postgres=# explain analyse
  select *
    from otus_edu
    where info_tsv @@ to_tsquery('text1');
                                                            QUERY PLAN                                                           
  -------------------------------------------------------------------------------------------------------------------------------
   Bitmap Heap Scan on otus_edu  (cost=159.30..6242.46 rows=16393 width=50) (actual time=4.636..9.946 rows=16680 loops=1)
     Recheck Cond: (info_tsv @@ to_tsquery('text1'::text))
     Heap Blocks: exact=982
     ->  Bitmap Index Scan on otus_edu_idx_lex  (cost=0.00..155.20 rows=16393 width=0) (actual time=4.342..4.342 rows=16680 loops=1)
           Index Cond: (info_tsv @@ to_tsquery('text1'::text))
   Planning Time: 0.999 ms
   Execution Time: 11.647 ms
  (7 rows)
  
  postgres=# 
  ```

> 4. Реализовать индекс на часть таблицы или индекс на поле с функцией
  ```sql
  postgres=# 
  postgres=# explain analyse
  select * from otus_edu where info = 'text1';
                                                  QUERY PLAN                                                 
  -----------------------------------------------------------------------------------------------------------
   Seq Scan on otus_edu  (cost=0.00..2072.00 rows=32857 width=33) (actual time=0.021..51.959 rows=33087 loops=1)
     Filter: (info = 'text1'::text)
     Rows Removed by Filter: 66913
   Planning Time: 0.637 ms
   Execution Time: 56.788 ms
  (5 rows)
  
  postgres=# 
  postgres=# explain analyse
  select * from otus_edu where upper(info) = upper('text1');
                                                 QUERY PLAN                                                
  ---------------------------------------------------------------------------------------------------------
   Seq Scan on   (cost=0.00..2322.00 rows=500 width=33) (actual time=0.030..95.340 rows=33087 loops=1)
     Filter: (upper(info) = 'TEXT1'::text)
     Rows Removed by Filter: 66913
   Planning Time: 0.144 ms
   Execution Time: 98.421 ms
  (5 rows)
  
  postgres=# 
  postgres=# explain analyse
  select * from otus_edu where upper(info) = 'TEXT1';
                                                  QUERY PLAN                                                
  ----------------------------------------------------------------------------------------------------------
   Seq Scan on otus_edu  (cost=0.00..2322.00 rows=500 width=33) (actual time=0.043..100.245 rows=33087 loops=1)
     Filter: (upper(info) = 'TEXT1'::text)
     Rows Removed by Filter: 66913
   Planning Time: 0.249 ms
   Execution Time: 103.751 ms
  (5 rows)
  
  postgres=# 
  postgres=# create index otus_edu_idx_func on otus_edu(upper(info));
  CREATE INDEX
  postgres=# 
  postgres=# explain analyse                                  
  select * from otus_edu where info = 'text1';
                                                  QUERY PLAN                                                 
  -----------------------------------------------------------------------------------------------------------
   Seq Scan on otus_edu  (cost=0.00..2072.00 rows=32857 width=33) (actual time=0.011..33.174 rows=33087 loops=1)
     Filter: (info = 'text1'::text)
     Rows Removed by Filter: 66913
   Planning Time: 1.535 ms
   Execution Time: 36.369 ms
  (5 rows)
  
  postgres=# 
  postgres=# explain analyse                                  
  select * from otus_edu where upper(info) = upper('text1');
                                                           QUERY PLAN                                                         
  ----------------------------------------------------------------------------------------------------------------------------
   Bitmap Heap Scan on otus_edu  (cost=8.17..764.29 rows=500 width=33) (actual time=5.277..29.704 rows=33087 loops=1)
     Recheck Cond: (upper(info) = 'TEXT1'::text)
     Heap Blocks: exact=822
     ->  Bitmap Index Scan on otus_edu_idx_func  (cost=0.00..8.04 rows=500 width=0) (actual time=4.620..4.621 rows=33087 loops=1)
           Index Cond: (upper(info) = 'TEXT1'::text)
   Planning Time: 0.249 ms
   Execution Time: 37.834 ms
  (7 rows)
  
  postgres=# 
  postgres=# explain analyse                                  
  select * from otus_edu where upper(info) = 'TEXT1';
                                                           QUERY PLAN                                                         
  ----------------------------------------------------------------------------------------------------------------------------
   Bitmap Heap Scan on otus_edu  (cost=8.17..764.29 rows=500 width=33) (actual time=5.042..28.940 rows=33087 loops=1)
     Recheck Cond: (upper(info) = 'TEXT1'::text)
     Heap Blocks: exact=822
     ->  Bitmap Index Scan on otus_edu_idx_func  (cost=0.00..8.04 rows=500 width=0) (actual time=4.385..4.386 rows=33087 loops=1)
           Index Cond: (upper(info) = 'TEXT1'::text)
   Planning Time: 0.163 ms
   Execution Time: 36.323 ms
  (7 rows)
  
  postgres=# 
  ```
> 5. Создать индекс на несколько полей
  ```sql
  postgres=# 
  postgres=# create index otus_edu_idx2 on otus_edu(pkey,info);
  CREATE INDEX
  postgres=# 
  postgres=# explain analyse
  select * from otus_edu where pkey=10 and info = 'text1';
                                                     QUERY PLAN                                                    
  -----------------------------------------------------------------------------------------------------------------
   Index Scan using otus_edu_idx2 on otus_edu  (cost=0.42..8.44 rows=1 width=33) (actual time=0.059..0.062 rows=1 loops=1)
     Index Cond: ((pkey = 10) AND (info = 'text1'::text))
   Planning Time: 0.651 ms
   Execution Time: 0.101 ms
  (4 rows)
  
  postgres=# 
  ```

> 6. Написать комментарии к каждому из индексов
  * Во всех вариантах индексов, кроме полнотекстового поиска, используется обычный btree(древовидный) индекс. Но используются разные приемы сканирования.
  * В индексе с полнотекстным поиском используется gin индекс
  * bitmap индекс хорошо, когда можно построить матрицу значений, он строит битовую карту.

> 7. Описать что и как делали и с какими проблемами столкнулись
  * Для корректной работы индексов - должна быть актуальная статистика
  * Больше проблемы не было
