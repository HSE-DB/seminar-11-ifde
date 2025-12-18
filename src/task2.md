## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```sql
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.022..0.022 rows=1 loops=1)
     Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
     Heap Blocks: exact=1
     ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.015..0.015 rows=1 loops=1)
          Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
     Planning Time: 1.346 ms
     Execution Time: 0.051 ms
    ```
    
    *Объясните результат:*
    Видим, что идет поиск по созданному GIN индексу. Он для каждого ключа (слова) хранит IDs строк, поэтому в любом случае потом делаем Bitmap Heap Scan по всей таблице. Поиск очень быстрый - 0.05 милисекунд. 

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```sql
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.075..0.075 rows=1 loops=1)
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.325 ms
     Execution Time: 0.090 ms
     ```
     
     *Объясните результат:*
     Быстрый поиск по индексу - 0.09 милисекунд. Ожидаемый результат. 

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```sql
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.119..0.120 rows=1 loops=1)
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.433 ms
     Execution Time: 0.140 ms
     ```
     
     *Объясните результат:*
     Почти то же самое, используем созданный индекс по колонке. Хотя для этой таблицы мы переупорядочили записи по индексу. Как видим, скорость выполнения это не уменьшило - она даже немного больше, чем в первом запросе.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```sql
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.019..0.020 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.353 ms
     Execution Time: 0.032 ms
     ```
     
     *Объясните результат:*
     Используем созданный индекс, быстрая операция - 0.03 милисекунды

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```sql
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.040..0.041 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.397 ms
     Execution Time: 0.055 ms
     ```
     
     *Объясните результат:*
     Тоже все хорошо, поиск по индексу. Скорость чуть выше, чем в обычной таблице.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Производительность почти одна и та же. Перезаписывание строк по индексу не дает какого-то сильного преимущества для наших запросов. Наверное, это логично, ведь поиск по индексу будет почти такой же в обоих случаях. А когда мы уже нашли указатели на конкретные строки, осталось их просто прочитать. 

     Надеюсь, что это ожидаемый резульат :)