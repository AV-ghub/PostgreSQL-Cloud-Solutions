В PostgreSQL есть несколько способов быстро получить скрипт создания всех индексов и первичных ключей. Вот самые эффективные методы:

## 1. Использование системных представлений (pg_indexes + pg_constraint)

```sql
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes 
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename, indexname;
```

## 2. Полный скрипт для всех индексов и PK

```sql
-- Все индексы (кроме первичных ключей)
SELECT 
    'CREATE INDEX ' || schemaname || '.' || indexname || ' ON ' || 
    schemaname || '.' || tablename || ' USING ' || 
    CASE WHEN indexdef LIKE '%USING btree%' THEN 'btree' 
         WHEN indexdef LIKE '%USING gin%' THEN 'gin'
         WHEN indexdef LIKE '%USING gist%' THEN 'gist'
         ELSE 'btree' END ||
    ' (' || 
    CASE WHEN indexdef LIKE '%(%' THEN 
        SUBSTRING(indexdef FROM '\(([^)]+)\)')
    ELSE 
        REPLACE(SUBSTRING(indexdef FROM 'ON [^ ]+ \(([^)]+)\)'), '"', '')
    END || ');' AS create_index_sql
FROM pg_indexes 
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
AND indexname NOT LIKE '%pkey%'
ORDER BY schemaname, tablename, indexname;

-- Первичные ключи
SELECT 
    'ALTER TABLE ' || n.nspname || '.' || t.relname || 
    ' ADD CONSTRAINT ' || c.conname || 
    ' PRIMARY KEY (' || 
    (SELECT STRING_AGG(a.attname, ', ' ORDER BY k.ordinality)
     FROM pg_attribute a
     JOIN UNNEST(c.conkey) WITH ORDINALITY AS k(attnum, ordinality) 
        ON a.attnum = k.attnum AND a.attrelid = c.conrelid
    ) || ');' AS create_pk_sql
FROM pg_constraint c
JOIN pg_class t ON c.conrelid = t.oid
JOIN pg_namespace n ON t.relnamespace = n.oid
WHERE c.contype = 'p'
AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, t.relname;
```

## 3. Улучшенная версия с учетом условий WHERE

```sql
-- Все ограничения (PK, UK, FK)
SELECT 
    'ALTER TABLE ' || n.nspname || '.' || t.relname || 
    ' ADD CONSTRAINT ' || c.conname || ' ' ||
    CASE c.contype 
        WHEN 'p' THEN 'PRIMARY KEY'
        WHEN 'u' THEN 'UNIQUE'
        WHEN 'f' THEN 'FOREIGN KEY'
    END || ' (' || 
    (SELECT STRING_AGG(a.attname, ', ' ORDER BY k.ordinality)
     FROM pg_attribute a
     JOIN UNNEST(c.conkey) WITH ORDINALITY AS k(attnum, ordinality) 
        ON a.attnum = k.attnum AND a.attrelid = c.conrelid
    ) || ')' ||
    CASE WHEN c.contype = 'f' THEN 
        ' REFERENCES ' || (SELECT n2.nspname || '.' || t2.relname 
                          FROM pg_class t2 
                          JOIN pg_namespace n2 ON t2.relnamespace = n2.oid 
                          WHERE t2.oid = c.confrelid) ||
        ' (' || (SELECT STRING_AGG(a2.attname, ', ' ORDER BY k2.ordinality)
                FROM pg_attribute a2
                JOIN UNNEST(c.confkey) WITH ORDINALITY AS k2(attnum, ordinality) 
                   ON a2.attnum = k2.attnum AND a2.attrelid = c.confrelid
               ) || ')'
    ELSE '' END || ';' AS create_constraint_sql
FROM pg_constraint c
JOIN pg_class t ON c.conrelid = t.oid
JOIN pg_namespace n ON t.relnamespace = n.oid
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, t.relname, c.contype, c.conname;
```

## 4. Для конкретной схемы или таблицы

```sql
-- Для конкретной схемы
SELECT indexdef || ';' 
FROM pg_indexes 
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- Для конкретной таблицы
SELECT indexdef || ';' 
FROM pg_indexes 
WHERE schemaname = 'public' AND tablename = 'your_table'
ORDER BY indexname;
```

## 5. Экспорт в файл

```sql
-- Экспорт в CSV
COPY (
    SELECT indexdef || ';' AS sql_command
    FROM pg_indexes 
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    UNION ALL
    SELECT 'ALTER TABLE ' || n.nspname || '.' || t.relname || 
           ' ADD CONSTRAINT ' || c.conname || 
           ' PRIMARY KEY (' || 
           (SELECT STRING_AGG(a.attname, ', ' ORDER BY k.ordinality)
            FROM pg_attribute a
            JOIN UNNEST(c.conkey) WITH ORDINALITY AS k(attnum, ordinality) 
               ON a.attnum = k.attnum AND a.attrelid = c.conrelid
           ) || ');' AS sql_command
    FROM pg_constraint c
    JOIN pg_class t ON c.conrelid = t.oid
    JOIN pg_namespace n ON t.relnamespace = n.oid
    WHERE c.contype = 'p'
    AND n.nspname NOT IN ('pg_catalog', 'information_schema')
) TO '/tmp/indexes_and_pks.sql' WITH (FORMAT text);
```

## 6. Использование pg_dump (самый простой способ)

```bash
# Экспорт только схемы (включая индексы и ограничения)
pg_dump -d your_database -s -t 'public.*' > schema_with_indexes.sql

# Только индексы и ограничения
pg_dump -d your_database -t 'public.*' --section=pre-data --section=post-data > indexes_and_constraints.sql
```

## 7. Практический пример для копирования

```sql
-- Простой скрипт для повседневного использования
SELECT 
    '-- Index: ' || schemaname || '.' || indexname || E'\n' ||
    indexdef || ';' || E'\n'
FROM pg_indexes 
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename, indexname;
```

## Советы:
1. **Для полного экспорта** используйте `pg_dump` - это самый надежный способ
2. **Для отдельных объектов** используйте системные представления
3. **Проверяйте синтаксис** - некоторые индексы могут иметь дополнительные опции
4. **Сохраняйте порядок** - сначала таблицы, потом индексы

Выберите тот метод,
