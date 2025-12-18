# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```sql
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.038..0.038 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.035..0.036 rows=0 loops=1)
         Index Cond: (category IS NULL)
   Planning Time: 0.411 ms
   Execution Time: 0.083 ms
   ```

   (у меня в VSCode отображается только это, то есть без workers и чего-то еще, что, как видел, есть в DataGrip)
   
   *Объясните результат:*
   Используем Индекс BRIN. Так как он хранит только Ranges, то потом делаем BitMap Scan по всей таблице, чтобы достать строки 

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```sql
   Bitmap Heap Scan on t_books  (cost=12.16..2388.04 rows=1 width=33) (actual time=49.678..49.680 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1225
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=76725 width=0) (actual time=0.295..0.295 rows=12250 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 1.083 ms
   Execution Time: 49.805 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Здесь используем индекс для поиску по категории. А вот далее делаем BitMap Scan, как в прошлом примере, с дополнительным условием на author. Думаю, это логично - среди найденных значений category уже просто отобрать автора. Можно было бы отдельно пройтись по индексу для author и потом найти пересечение результата с результатом для индекса по category. Но, видимо, так быстрее, так как нам все равно придется делать BitMap Scan. 

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```sql
   Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=42.088..42.089 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=42.024..42.025 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.011..13.195 rows=150000 loops=1)
   Planning Time: 0.441 ms
   Execution Time: 42.228 ms
   ```
   
   *Объясните результат:*
   Вместо того, чтобы для каждого значения category выполнять поиск по индексу, просто идем по всей таблице, чтобы создать соответствующую группу. Кажется, логично, что так не медленнее, чем искать по индексу для каждого значения category (которых потенциально может быть достаточно много). 
   Операция очень дорогая, целых 42 милисекунды.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```sql
   Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=11.604..11.605 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=11.599..11.600 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 0.400 ms
   Execution Time: 11.644 ms
   ```
   
   *Объясните результат:*
   Здесь индекс не используется. Возможно, каждый block range состоит из множества авторов, и чтобы найти всех, имя которых начинается на S, все равно будем просматривать все блоки, что дороже, чем просканировать таблицу. Либо BRIN индекс в приницпе не работает с оператором LIKE (хотя это немного странно)

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```sql
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=49.626..49.627 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=49.618..49.620 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.454 ms
   Execution Time: 49.655 ms
   ```
   
   *Объясните результат:*
   Очень долгая операция - 49 милисекунд, даже дольше, чем группировка. Вот снова видим сканирование по всей таблице. Как-то странно, что индекс не используется. Кажется, даже статистику таблицу обновил. Загадка

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```sql
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=1.024..1.025 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8830
   Heap Blocks: lossy=73
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.120..0.120 rows=730 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.522 ms
   Execution Time: 1.140 ms
   ```
   
   *Объясните результат:*
   Ура! Теперь ищем по индексу и затем выполняем Bitmap Heap Scan. По сравнения с пунктом 7 время уменьшилось почти в 50 раз с 49 милисекунд до 1 милисекунды