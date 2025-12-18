## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```sql
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=18.091..95.619 rows=499852 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=8334
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=16.792..16.792 rows=499852 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.449 ms
    Execution Time: 115.867 ms
    ```
    
    *Объясните результат:*
    Поиск по индексу с созданием BitMap. Стоимость - 59, время - 115 милисекунд.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```sql
    {"command":"CLUSTER","rowCount":null,"oid":null,"rows":[],"fields":[],"_types":{"_types":{"arrayParser":{},"builtins":{"BOOL":16,"BYTEA":17,"CHAR":18,"INT8":20,"INT2":21,"INT4":23,"REGPROC":24,"TEXT":25,"OID":26,"TID":27,"XID":28,"CID":29,"JSON":114,"XML":142,"PG_NODE_TREE":194,"SMGR":210,"PATH":602,"POLYGON":604,"CIDR":650,"FLOAT4":700,"FLOAT8":701,"ABSTIME":702,"RELTIME":703,"TINTERVAL":704,"CIRCLE":718,"MACADDR8":774,"MONEY":790,"MACADDR":829,"INET":869,"ACLITEM":1033,"BPCHAR":1042,"VARCHAR":1043,"DATE":1082,"TIME":1083,"TIMESTAMP":1114,"TIMESTAMPTZ":1184,"INTERVAL":1186,"TIMETZ":1266,"BIT":1560,"VARBIT":1562,"NUMERIC":1700,"REFCURSOR":1790,"REGPROCEDURE":2202,"REGOPER":2203,"REGOPERATOR":2204,"REGCLASS":2205,"REGTYPE":2206,"UUID":2950,"TXID_SNAPSHOT":2970,"PG_LSN":3220,"PG_NDISTINCT":3361,"PG_DEPENDENCIES":3402,"TSVECTOR":3614,"TSQUERY":3615,"GTSVECTOR":3642,"REGCONFIG":3734,"REGDICTIONARY":3769,"JSONB":3802,"REGNAMESPACE":4089,"REGROLE":4096}},"text":{},"binary":{}},"RowCtor":null,"rowAsArray":true}
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```sql
    Bitmap Heap Scan on test_cluster  (cost=5537.07..20078.58 rows=496600 width=39) (actual time=13.342..88.334 rows=499852 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=4166
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5412.93 rows=496600 width=0) (actual time=12.292..12.292 rows=499852 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.482 ms
    Execution Time: 132.301 ms
    ```
    
    *Объясните результат:*
    Стоимость - 5537. Очень большой рост по сравнению с первым запроом. Хотя за счет кластеризации уменьшилось время BitMap scan примерно на 5 милисекунд по сравнению с первым запросом (с 18 до 13). Скорее всего, потому что теперь все "1" после кластеризации расположены рядом и их можно все сразу прочитать с диска. А вот время выполнения увеличилось с 115 мс до 132 мс. Это как-то странно... Мне кажеся, погрешность для большого числа строк. Попробовал выполнить еще раз - теперь стоимость стала 82 милисекунды. 

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    Производительность увеличилась при использовании кластеризации. То есть, для больших таблиц это эффективно. Отлично!