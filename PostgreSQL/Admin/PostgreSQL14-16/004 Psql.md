## 🛠️ **Теперь переходим к утилите psql **

---

### **1. Ключевые возможности psql**

#### **Из книги:**
- **Интерактивный терминал** для работы с PostgreSQL
- **Мощные метакоманды** (backslash commands)
- **Поддержка скриптов** и пакетного выполнения
- **Настройка через .psqlrc**
- **История команд** (.psql_history)

#### **Базовое подключение:**
```bash
# Основные опции
psql -h localhost -p 5432 -U username -d database -W

# Короткие варианты:
psql database username  # локальное подключение
psql "host=localhost dbname=test user=postgres"
psql service=myservice  # через service file
```

#### **Важные метакоманды:**
```
\?                  # Справка по метакомандам
\h [команда]        # Справка по SQL (например: \h SELECT)
\q                  # Выход
\c [db] [user]      # Переподключение
\l                  # Список БД
\dt [pattern]       # Список таблиц
\di                 # Список индексов
\dv                 # Список представлений
\df                 # Список функций
\dn                 # Список схем
\du                 # Список пользователей
\dp [object]        # Права доступа
\d [object]         # Описание объекта
\d+ [object]        # Подробное описание
\watch [sec]        # Периодическое выполнение
\g [file]           # Выполнить запрос/сохранить в файл
\gx                 # Выполнить с расширенным выводом
\x                  # Переключить расширенный вывод
\i file             # Выполнить скрипт из файла
\o [file]           # Перенаправить вывод в файл
\set [var [value]]  # Установить переменную
\unset var          # Удалить переменную
\echo [text]        # Вывести текст
\prompt [var] [text]# Запросить ввод
\timing             # Включить/выключить тайминг
```

---

### **2. Продвинутые техники psql**

#### **Настройка через .psqlrc:**
```sql
-- ~/.psqlrc
\set QUIET 1
\set PROMPT1 '%[%033[1;33m%]%/%[%033[0m%]%# '
\set PROMPT2 '... '
\set COMP_KEYWORD_CASE preserve-lower
\pset null '[NULL]'
\pset border 2
\pset linestyle unicode
\pset format wrapped
\timing on
\x auto
\set QUIET 0

-- Автоматические действия при подключении
\setenv PAGER less
\setenv LESS '-iMSx4 -FX'
```

#### **Переменные psql:**
```sql
-- Установка переменных
\set myvar 100
\set tablename 'users'

-- Использование
SELECT * FROM :tablename WHERE id = :myvar;

-- Системные переменные
\echo :VERSION
\echo :USER
\echo :DBNAME
\echo :HOST
\echo :PORT
```

#### **Расширенный вывод:**
```sql
-- Включить/выключить расширенный вывод
\x auto  -- автоматически при широких результатах

-- Форматирование
\pset border 2        -- двойная рамка
\pset linestyle unicode
\pset format wrapped  -- перенос длинных строк
\pset null '[NULL]'   -- обозначение NULL
\pset pager always    -- всегда использовать pager
```

---

### **3. Работа со скриптами**

#### **Выполнение SQL файлов:**
```bash
# Пакетное выполнение
psql -f script.sql
psql -f script.sql -o output.txt

# Передача SQL через stdin
echo "SELECT version();" | psql
cat script.sql | psql

# С параметрами
psql -v v1=100 -v v2="'text'" -f script.sql
```

#### **Внутри psql:**
```sql
-- Выполнение файла
\i /path/to/script.sql

-- Сохранение результатов
\o results.txt
SELECT * FROM large_table;
\o

-- Логирование сессии
\o session.log
\set ECHO all
\i script.sql
\set ECHO none
\o
```

---

### **4. Полезные трюки**

#### **Автодополнение:**
- **Tab дважды** для подсказки
- Настроить через `\set COMP_KEYWORD_CASE preserve-lower`

#### **История:**
```bash
# Поиск в истории
Ctrl+R  # reverse search
\s      # показать историю
\s pattern  # поиск в истории
```

#### **Копирование данных:**
```sql
-- Экспорт в CSV
\copy (SELECT * FROM table) TO '/path/file.csv' WITH CSV HEADER

-- Импорт из CSV
\copy table FROM '/path/file.csv' WITH CSV HEADER

-- Прямое копирование (серверное)
COPY table TO '/path/file.csv' WITH CSV HEADER;  -- требует прав суперпользователя
```

---

### **5. Отладка и профилирование**

#### **Просмотр планов запросов:**
```sql
-- Простой EXPLAIN
EXPLAIN SELECT * FROM users WHERE active = true;

-- С временем выполнения
EXPLAIN ANALYZE SELECT * FROM users WHERE active = true;

-- Буферизация
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM large_table;

-- Форматирование
EXPLAIN (ANALYZE, FORMAT JSON) SELECT * FROM table;
```

#### **Встроенный профилировщик:**
```sql
-- Включить расширение
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Топ запросов
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) as hit_percent
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;
```

---

### **6. Практическое задание**

#### **Задание 1: Настройка .psqlrc**
```sql
-- Создайте ~/.psqlrc с:
-- 1. Цветным промптом с именем БД
-- 2. Автоматическим включением timing
-- 3. Автоматическим \x для широких результатов
-- 4. Кастомным форматом NULL
-- 5. Установите полезные алиасы
```

#### **Задание 2: Создание утилитных функций**
```sql
-- В psql создайте функцию для быстрого анализа таблицы
\set analyze_table 'SELECT \
    schemaname, \
    relname, \
    n_live_tup, \
    n_dead_tup, \
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_ratio, \
    pg_size_pretty(pg_total_relation_size(schemaname || \'.\' || relname)) as size, \
    last_autovacuum, \
    last_autoanalyze \
FROM pg_stat_user_tables \
WHERE relname LIKE \':1%\' \
ORDER BY n_live_tup DESC;'

-- Использование
:analyze_table 'user'
```

#### **Задание 3: Автоматизация отчетов**
```bash
#!/bin/bash
# Скрипт для ежедневного отчета
REPORT_FILE="/var/log/postgresql/daily_report_$(date +%Y%m%d).txt"

psql << EOF > $REPORT_FILE
\echo '=== DAILY POSTGRESQL REPORT ==='
\echo 'Generated: $(date)'
\echo ''
\echo '--- Database Sizes ---'
SELECT datname, pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database ORDER BY pg_database_size(datname) DESC;

\echo ''
\echo '--- Top Tables by Size ---'
SELECT schemaname, relname, 
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) as size
FROM pg_stat_user_tables 
ORDER BY pg_total_relation_size(schemaname||'.'||relname) DESC 
LIMIT 10;

\echo ''
\echo '--- Connection Summary ---'
SELECT datname, count(*) as connections,
       count(*) FILTER (WHERE state = 'active') as active
FROM pg_stat_activity 
WHERE backend_type = 'client backend'
GROUP BY datname;
EOF
```

---

### **🎯 Ключевые тезисы для собеседования:**

1. **psql - основной инструмент администратора** PostgreSQL
2. **Метакоманды (\команды) экономят время** в повседневной работе
3. **.psqlrc позволяет кастомизировать** рабочее окружение
4. **\copy vs COPY:** \copy работает от клиента, COPY - от сервера
5. **Расширенный вывод (\x) незаменим** для широких таблиц

---

## ❓ **Вопросы для самопроверки по psql:**

1. Как выполнить SQL скрипт из файла и замерить время выполнения?
2. В чем разница между \copy и COPY?
3. Как сохранить результаты запроса в файл?
4. Как настроить автодополнение таблиц?
5. Как просмотреть и отфильтровать историю команд?
