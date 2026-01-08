## 🔄 **MVCC, VACUUM и Autovacuum**

### **1. Основы MVCC (Multi-Version Concurrency Control)**

#### **Ключевые концепции из книги:**
- **Каждая транзакция видит "снимок" (snapshot)** данных на момент ее начала
- **UPDATE = DELETE старой + INSERT новой версии**
- **DELETE = пометить как удаленную (не физическое удаление)**
- **Служебные поля в каждой строке:**
  ```sql
  -- Псевдо-структура строки
  ROW (
    xmin,      -- ID транзакции, создавшей строку
    xmax,      -- ID транзакции, удалившей/обновившей строку
    cmin,      -- ID команды внутри транзакции (создание)
    cmax,      -- ID команды внутри транзакции (удаление)
    ctid,      -- физическое расположение (page, tuple)
    infomask,  -- флаги состояния
    data...    -- пользовательские данные
  )
  ```

#### **Практическое исследование:**
```sql
-- Установка расширения для просмотра служебных данных
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Создаем тестовую таблицу
CREATE TABLE mvcc_test (
    id SERIAL PRIMARY KEY,
    data TEXT
);

-- Вставляем данные
INSERT INTO mvcc_test (data) VALUES ('first'), ('second');

-- Смотрим служебные поля
SELECT 
    xmin,          -- транзакция создания
    xmax,          -- транзакция удаления (0 если живая)
    ctid,          -- физическое расположение
    id, 
    data 
FROM mvcc_test;

-- Получаем ID текущей транзакции
SELECT txid_current();

-- Обновляем строку
BEGIN;
UPDATE mvcc_test SET data = 'updated' WHERE id = 1;
SELECT * FROM mvcc_test WHERE id = 1;
-- xmax будет установлен в ID текущей транзакции
COMMIT;
```

---

### **2. Проблема "мертвых" строк (Dead Tuples)**

#### **Из книги:**
```sql
-- Мертвые строки накапливаются после UPDATE/DELETE
-- Пример с обновлением:
-- Старая версия: xmax = 123 (помечена удаленной)
-- Новая версия: xmin = 123 (создана той же транзакцией)

-- Мониторинг мертвых строк
SELECT 
    schemaname,
    relname,
    n_live_tup,      -- живые строки
    n_dead_tup,      -- мертвые строки
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_ratio,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

#### **Практическое задание:**
```sql
-- Создаем таблицу с историей изменений
CREATE TABLE user_actions (
    id BIGSERIAL PRIMARY KEY,
    user_id INT,
    action TEXT,
    created_at TIMESTAMP DEFAULT now()
);

-- Генерируем мертвые строки
DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..1000 LOOP
        INSERT INTO user_actions (user_id, action) 
        VALUES (i % 100, 'login');
        
        -- Обновляем каждую 10-ю строку
        IF i % 10 = 0 THEN
            UPDATE user_actions 
            SET action = 'logout' 
            WHERE id = i;
        END IF;
        
        -- Удаляем каждую 20-ю строку
        IF i % 20 = 0 THEN
            DELETE FROM user_actions WHERE id = i;
        END IF;
    END LOOP;
END $$;

-- Проверяем мертвые строки
SELECT 
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_ratio
FROM pg_stat_user_tables 
WHERE relname = 'user_actions';
```

---

### **3. VACUUM - ручная очистка**

#### **Из книги:**
```sql
-- Базовый VACUUM
VACUUM;  -- очистка всей БД

-- VACUUM конкретной таблицы
VACUUM user_actions;

-- С анализом статистики
VACUUM ANALYZE user_actions;

-- Полная очистка (блокирует таблицу!)
VACUUM FULL user_actions;  -- ОПАСНО на production!

-- С подробным выводом
VACUUM (VERBOSE, ANALYZE) user_actions;

-- Мониторинг прогресса VACUUM
SELECT * FROM pg_stat_progress_vacuum;
```

#### **VACUUM FULL vs обычный VACUUM:**
| **Аспект** | **VACUUM** | **VACUUM FULL** |
|------------|------------|-----------------|
| **Блокировка** | Не блокирует (concurrent) | Эксклюзивная блокировка |
| **Место на диске** | Освобождает для повторного использования | Переписывает таблицу, нужен дополнительный disk space |
| **Дефрагментация** | Нет | Да (полная реорганизация) |
| **Когда использовать** | Регулярно | Только при сильной фрагментации, в downtime |

---

### **4. Autovacuum - автоматическая очистка**

#### **Ключевые параметры:**
```sql
-- Основные настройки autovacuum
SELECT 
    name, 
    setting, 
    unit,
    short_desc
FROM pg_settings 
WHERE name LIKE 'autovacuum%' 
ORDER BY name;

-- Критические параметры:
-- autovacuum = on/off
-- autovacuum_max_workers = 3 (макс одновременно)
-- autovacuum_naptime = 1min (пауза между запусками)
-- autovacuum_vacuum_threshold = 50 (мин мертвых строк)
-- autovacuum_vacuum_scale_factor = 0.2 (20% от таблицы)
-- autovacuum_analyze_threshold = 50
-- autovacuum_analyze_scale_factor = 0.1
-- autovacuum_vacuum_cost_limit = 200
-- autovacuum_vacuum_cost_delay = 20ms
```

#### **Настройка для больших таблиц:**
```sql
-- Проблема: большая таблица (1M строк), 20% = 200K мертвых строк
-- Решение: уменьшить scale factor

ALTER TABLE huge_table SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- 1% вместо 20%
    autovacuum_vacuum_threshold = 10000,    -- минимум 10K строк
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_analyze_threshold = 5000,
    autovacuum_vacuum_cost_limit = 2000     -- более агрессивный
);

-- Для очень активных таблиц:
ALTER TABLE active_table SET (
    autovacuum_enabled = true,
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold = 5000,
    toast.autovacuum_vacuum_scale_factor = 0.05
);
```

---

### **5. Мониторинг Autovacuum**

```sql
-- Текущие autovacuum процессы
SELECT 
    datname,
    usename,
    pid,
    current_timestamp - query_start as duration,
    query,
    state
FROM pg_stat_activity 
WHERE query LIKE 'autovacuum%'
ORDER BY duration DESC;

-- Статистика по autovacuum
SELECT 
    schemaname,
    relname,
    last_autovacuum,
    last_autoanalyze,
    autovacuum_count,
    autoanalyze_count,
    n_dead_tup,
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 10;

-- Настройки журналирования autovacuum
-- log_autovacuum_min_duration = 0 (логировать все) или -1 (не логировать)
```

#### **Практический мониторинг:**
```sql
-- Создаем представление для мониторинга
CREATE VIEW autovacuum_monitor AS
SELECT 
    schemaname,
    relname,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) as size,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_percent,
    CASE 
        WHEN n_dead_tup > (n_live_tup * 0.1 + 50) THEN 'NEEDS VACUUM'
        ELSE 'OK'
    END as status,
    last_autovacuum,
    age(now(), last_autovacuum) as since_last_vacuum,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Использование
SELECT * FROM autovacuum_monitor WHERE dead_percent > 5;
```

---

### **6. Практическое задание по MVCC/VACUUM**

#### **Задание 1: Исследование MVCC**
```sql
-- 1. Создайте таблицу и вставьте данные
-- 2. Изучите xmin/xmax до и после UPDATE
-- 3. Используйте pageinspect для просмотра физической структуры
-- 4. Проверьте мертвые строки после нескольких обновлений

SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask
FROM heap_page_items(get_raw_page('your_table', 0));
```

#### **Задание 2: Настройка Autovacuum**
```sql
-- 1. Создайте таблицу с разными паттернами обновления
-- 2. Настройте autovacuum параметры для каждой:
--    - Часто обновляемая маленькая таблица
--    - Большая редко обновляемая таблица
--    - Таблица с регулярными DELETE
-- 3. Мониторьте эффективность настроек

-- Пример для таблицы логов (частые INSERT, редкий DELETE)
ALTER TABLE logs SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.02,
    autovacuum_analyze_threshold = 500
);
```

#### **Задание 3: Анализ проблем**
```sql
-- Найти таблицы с проблемами
SELECT 
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_ratio,
    pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) as size,
    last_autovacuum,
    CASE 
        WHEN n_dead_tup > 10000 AND age(now(), last_autovacuum) > interval '1 day'
        THEN 'CRITICAL'
        WHEN n_dead_tup > 5000 
        THEN 'WARNING'
        ELSE 'OK'
    END as alert_level
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

### **🎯 Ключевые тезисы для собеседования:**

1. **MVCC позволяет конкурентный доступ** без блокировок читателей
2. **UPDATE = DELETE + INSERT** - старые версии остаются как "мертвые строки"
3. **VACUUM очищает мертвые строки**, но не возвращает space файловой системе
4. **Autovacuum критически важен** для поддержания производительности
5. **Мертвые строки > 10-20%** = проблема, требующая вмешательства
6. **VACUUM FULL опасен на production** - используйте pg_repack вместо него

---

## ❓ **Вопросы для самопроверки по MVCC/VACUUM:**

1. Почему после DELETE таблица не уменьшается в размере?
2. Как VACUUM влияет на производительность во время работы?
3. Когда использовать ручной VACUUM вместо autovacuum?
4. Что такое "freeze" и зачем оно нужно?
5. Как настроить autovacuum для таблицы с 100 млн строк?

---

Отлично! Сначала отвечу на ваши вопросы, затем на вопросы самопроверки.

---

## 🗑️ **Как вернуть системе свободное место после VACUUM?**

### **Проблема:**
Обычный `VACUUM` только помечает пространство как **доступное для повторного использования PostgreSQL**, но **не возвращает операционной системе**.

```sql
-- Демонстрация
CREATE TABLE test_space (id SERIAL, data TEXT);
INSERT INTO test_space (data) SELECT generate_series(1, 100000)::TEXT;
SELECT pg_size_pretty(pg_total_relation_size('test_space')); -- ~6MB

DELETE FROM test_space WHERE id % 2 = 0; -- Удаляем половину строк
VACUUM test_space;
SELECT pg_size_pretty(pg_total_relation_size('test_space')); -- ВСЕ ЕЩЕ ~6MB!
```

### **Решение 1: VACUUM FULL (НЕ РЕКОМЕНДУЕТСЯ на production)**
```sql
VACUUM FULL test_space;  -- Блокирует таблицу, требует доп. место
-- Теперь размер уменьшится до ~3MB
```
**Проблемы VACUUM FULL:**
- Эксклюзивная блокировка таблицы
- Требует дополнительного места на диске (до 2x размера таблицы)
- Долгое выполнение для больших таблиц

### **Решение 2: pg_repack (рекомендуется)**
```bash
# Установка
sudo apt-get install postgresql-14-repack

# Использование
pg_repack -k --table test_space -d your_database -U postgres
```
**Преимущества pg_repack:**
- Минимальная блокировка (только в конце операции)
- Не требует дополнительного места на диске
- Можно выполнять на работающей БД

### **Решение 3: CLUSTER или REINDEX**
```sql
-- CLUSTER перезаписывает таблицу по индексу
CLUSTER test_space USING test_space_pkey;

-- REINDEX для индексов
REINDEX TABLE test_space;
```
**Ограничения:** Требуют эксклюзивной блокировки

### **Решение 4: Секционирование + TRUNCATE**
```sql
-- Для таблиц с историческими данными
-- Создаем секционированную таблицу
CREATE TABLE logs_partitioned (...) PARTITION BY RANGE (created_at);

-- Старые секции можно DETACH и удалить файлы
ALTER TABLE logs_partitioned DETACH PARTITION old_logs;
DROP TABLE old_logs;  -- Возвращает место системе
```

### **Решение 5: Периодическая пересоздание**
```sql
-- Для небольших таблиц
BEGIN;
CREATE TABLE new_table AS SELECT * FROM old_table;
DROP TABLE old_table;
ALTER TABLE new_table RENAME TO old_table;
-- Восстановить индексы, права и т.д.
COMMIT;
```

### **Решение 6: Настройка autovacuum для агрессивного освобождения**
```sql
-- Увеличить FSM (Free Space Map)
ALTER TABLE your_table SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_freeze_max_age = 100000000
);

-- Проверка свободного пространства в таблице
SELECT 
    schemaname,
    relname,
    pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) as size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname) - 
                   pg_relation_size(schemaname||'.'||relname)) as free_space
FROM pg_stat_user_tables
WHERE pg_total_relation_size(schemaname||'.'||relname) > 
      pg_relation_size(schemaname||'.'||relname);
```

---

## ❓ **Ответы на вопросы самопроверки по MVCC/VACUUM:**

### **1. Почему после DELETE таблица не уменьшается в размере?**
**PostgreSQL использует "on-disk" структуру:**
- DELETE помечает строки как "мертвые" (устанавливает xmax)
- Физическое место остается в файле таблицы
- VACUUM помечает это место как "доступное для повторного использования"
- **Чтобы вернуть место ОС:** нужен VACUUM FULL, pg_repack или пересоздание

**Аналогия:** Книга, где страницы вырваны, но остаются в книге. VACUUM = можно писать на вырванных страницах. VACUUM FULL = переписать книгу без вырванных страниц.

### **2. Как VACUUM влияет на производительность во время работы?**
```sql
-- VACUUM может замедлять систему:
-- 1. I/O нагрузка (чтение/запись страниц)
-- 2. Использование CPU
-- 3. Конкуренция за блокировки

-- Настройка стоимости VACUUM:
vacuum_cost_page_hit = 1      # чтение из shared_buffers
vacuum_cost_page_miss = 10    # чтение с диска
vacuum_cost_page_dirty = 20   # запись грязной страницы
vacuum_cost_limit = 200       # лимит "стоимости" перед паузой
vacuum_cost_delay = 0         # задержка между работой (ms)

-- Автовакуум замедляется при достижении лимита:
-- autovacuum_vacuum_cost_delay = 20ms
-- autovacuum_vacuum_cost_limit = 200

-- Мониторинг влияния:
SELECT 
    datname,
    blks_read,
    blks_hit,
    tup_returned,
    tup_fetched,
    tup_updated,
    tup_deleted
FROM pg_stat_database;
```

### **3. Когда использовать ручной VACUUM вместо autovacuum?**
**Ручной VACUUM нужен когда:**
1. **Сразу после массового DELETE/UPDATE**
   ```sql
   DELETE FROM logs WHERE created_at < '2023-01-01';
   VACUUM ANALYZE logs;  -- Немедленно
   ```

2. **Перед/после больших миграций**
   ```sql
   -- Перед:
   VACUUM ANALYZE;
   -- После изменения схемы:
   VACUUM ANALYZE changed_table;
   ```

3. **Когда autovacuum не справляется**
   ```sql
   -- Проверка
   SELECT * FROM autovacuum_monitor 
   WHERE dead_percent > 30 
   AND since_last_vacuum > interval '1 hour';
   
   -- Ручной запуск
   VACUUM (VERBOSE, ANALYZE) problem_table;
   ```

4. **Для системных таблиц**
   ```sql
   VACUUM pg_catalog.pg_class;
   VACUUM pg_catalog.pg_attribute;
   ```

5. **Перед большими запросами** (чтобы статистика была актуальной)

### **4. Что такое "freeze" и зачем оно нужно?**
**Проблема wraparound:**
- XID (ID транзакции) - 32-битное число (~4 млрд)
- Через 4 млрд транзакций счетчик обнуляется
- Старые транзакции становятся "в будущем"

**Решение - freeze:**
```sql
-- Замораживание помечает старые строки как "видимые всем"
-- Настройки:
vacuum_freeze_min_age = 50000000     -- мин возраст для freeze
vacuum_freeze_table_age = 150000000  -- макс возраст перед агрессивным vacuum
autovacuum_freeze_max_age = 200000000 -- автовакуум при приближении к лимиту

-- Проверка возраста транзакций
SELECT 
    datname,
    age(datfrozenxid) as tx_age,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Критический уровень (требует немедленного VACUUM):
-- age > 1.5 млрд - предупреждение
-- age > 2 млрд - автоэкстренный vacuum
-- age > 2.1 млрд - БД переходит в read-only!
```

### **5. Как настроить autovacuum для таблицы с 100 млн строк?**
```sql
-- Проблема: 20% от 100M = 20M мертвых строк перед vacuum!
-- Решение: уменьшить scale factor

ALTER TABLE huge_table SET (
    autovacuum_enabled = true,
    autovacuum_vacuum_scale_factor = 0.01,      -- 1% вместо 20%
    autovacuum_vacuum_threshold = 100000,       -- мин 100K мертвых строк
    autovacuum_analyze_scale_factor = 0.005,    -- 0.5% для analyze
    autovacuum_analyze_threshold = 50000,
    autovacuum_vacuum_cost_limit = 2000,        -- более агрессивный
    autovacuum_freeze_max_age = 150000000,      -- раньше freeze
    toast.autovacuum_vacuum_scale_factor = 0.02 -- для TOAST
);

-- Дополнительные настройки в postgresql.conf:
autovacuum_max_workers = 5                     -- больше воркеров
maintenance_work_mem = 1GB                     -- больше памяти для vacuum
autovacuum_work_mem = -1                       -- использовать maintenance_work_mem

-- Мониторинг для больших таблиц:
CREATE VIEW huge_table_monitor AS
SELECT 
    'huge_table' as table_name,
    pg_size_pretty(pg_total_relation_size('huge_table')) as total_size,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / 1000000, 2) as dead_millions,
    round(100.0 * n_dead_tup / (n_live_tup + 1), 4) as dead_percent,
    last_autovacuum,
    age(now(), last_autovacuum) as since_vacuum,
    CASE 
        WHEN n_dead_tup > 1000000 THEN 'NEEDS ATTENTION'
        ELSE 'OK'
    END as status
FROM pg_stat_user_tables 
WHERE relname = 'huge_table';
```

---

## 📊 **Шпаргалка по освобождению места:**

| Метод | Блокировка | Возвращает место ОС | Рекомендация |
|-------|------------|---------------------|--------------|
| **VACUUM** | Нет | Нет | Ежедневно |
| **VACUUM FULL** | Да (эксклюзивная) | Да | Только в downtime |
| **pg_repack** | Минимальная | Да | Для production |
| **CLUSTER** | Да | Да | При сильной фрагментации |
| **TRUNCATE** | Да | Да | Для очистки всей таблицы |
| **Секционирование** | Частичная | Да | Для исторических данных |

---

## ⚖️ **Параметры стоимости VACUUM: подробное объяснение**

### **Философия "стоимости" (cost-based vacuum)**

PostgreSQL использует **систему "стоимости"** чтобы:
1. **Не перегружать систему** во время VACUUM
2. **Балансировать** между очисткой и обычными запросами
3. **Контролировать I/O нагрузку** автоматически

**Аналогия:** У вас есть бюджет в 200 "единиц". Каждое действие имеет цену:
- Взять книгу с полки (чтение из памяти) = 1 единица
- Сходить в библиотеку (чтение с диска) = 10 единиц
- Написать что-то в книге (запись на диск) = 20 единиц

---

### **1. Базовые параметры стоимости**

#### **vacuum_cost_page_hit = 1**
```sql
-- Когда VACUUM читает страницу ИЗ SHARED_BUFFERS
-- Пример: часто используемые страницы уже в памяти
SELECT setting FROM pg_settings WHERE name = 'vacuum_cost_page_hit';
-- Значение: 1 (минимальная стоимость)
```

**Кейс:** Таблица активно используется, ее данные в shared_buffers.
**Эффект:** VACUUM работает быстро, почти не замедляя систему.

#### **vacuum_cost_page_miss = 10**
```sql
-- Когда VACUUM читает страницу С ДИСКА
-- Пример: холодные данные, не в памяти
SELECT setting FROM pg_settings WHERE name = 'vacuum_cost_page_miss';
-- Значение: 10 (в 10 раз дороже)
```

**Кейс:** Архивные данные или большая таблица, не помещающаяся в shared_buffers.
**Эффект:** VACUUM замедляется, так как каждое чтение с диска "дороже".

#### **vacuum_cost_page_dirty = 20**
```sql
-- Когда VACUUM записывает измененную ("грязную") страницу на диск
-- Пример: нашел мертвые строки, пометил место как свободное
SELECT setting FROM pg_settings WHERE name = 'vacuum_cost_page_dirty';
-- Значение: 20 (самое дорогое действие)
```

**Кейс:** Много мертвых строк → много записей на диск.
**Эффект:** Сильное замедление при активной записи.

---

### **2. Лимиты и задержки**

#### **vacuum_cost_limit = 200**
```sql
-- Максимальная "стоимость" перед паузой
-- Пример: vacuum_cost_delay = 20ms, limit = 200
-- VACUUM работает, пока не наберет 200 единиц стоимости, затем ждет 20ms
SELECT setting FROM pg_settings WHERE name = 'vacuum_cost_limit';
-- Значение: 200 (по умолчанию)
```

**Формула работы:**
```
Действия VACUUM → накапливается стоимость → достигли limit → пауза vacuum_cost_delay → повтор
```

#### **vacuum_cost_delay = 0 (или значение в ms)**
```sql
-- Задержка между циклами работы VACUUM
-- 0 = отключить систему стоимости (опасно!)
SELECT setting FROM pg_settings WHERE name = 'vacuum_cost_delay';
-- Значение по умолчанию: 0 (в новых версиях может быть 2ms)
```

---

## 📊 **Практические кейсы и настройки**

### **Кейс 1: OLTP система (много коротких транзакций)**
```sql
-- Проблема: VACUUM мешает быстрым транзакциям
-- Решение: Агрессивные задержки

-- В postgresql.conf:
vacuum_cost_delay = 2ms          -- Частые паузы
vacuum_cost_limit = 200          -- Стандартный лимит
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 200

-- Расчет: 
-- Если все чтения из памяти (hit): 200 страниц → пауза
-- Если с диска (miss): 20 страниц → пауза
-- Если пишем (dirty): 10 страниц → пауза
```

### **Кейс 2: Система отчетов (большие таблицы, редкие обновления)**
```sql
-- Проблема: VACUUM слишком медленный на больших таблицах
-- Решение: Уменьшить задержки, увеличить лимит

vacuum_cost_delay = 0ms          -- Минимальные паузы
vacuum_cost_limit = 1000         -- Больший лимит
autovacuum_vacuum_cost_delay = 0ms
autovacuum_vacuum_cost_limit = 1000

-- Эффект: VACUUM работает почти непрерывно
-- Риск: Может мешать запросам
```

### **Кейс 3: Смешанная нагрузка (баланс)**
```sql
-- Компромисс: настройка по умолчанию обычно оптимальна
vacuum_cost_delay = 2ms
vacuum_cost_limit = 200
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 200

-- ИЛИ более агрессивно для SSD:
vacuum_cost_delay = 1ms          -- SSD быстрее
vacuum_cost_limit = 400          -- Можно больше
```

---

## 🔧 **Практический мониторинг и настройка**

### **1. Как мониторить эффективность текущих настроек**
```sql
-- Мониторинг autovacuum
SELECT 
    schemaname,
    relname,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count,
    -- Если autovacuum_count растет, но n_dead_tup не уменьшается,
    -- значит autovacuum не успевает
    CASE 
        WHEN n_dead_tup > 1000 
             AND age(now(), last_autovacuum) < interval '1 hour'
        THEN 'VACUUM too slow'
        ELSE 'OK'
    END as status
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 10;

-- Мониторинг I/O нагрузки от VACUUM
SELECT 
    datname,
    blks_read,          -- Чтения с диска
    blks_hit,           -- Чтения из кэша
    tup_updated,        -- Обновления (создают мертвые строки)
    tup_deleted         -- Удаления
FROM pg_stat_database;
```

### **2. Тестовый скрипт для подбора настроек**
```sql
-- Создаем тестовую таблицу
CREATE TABLE vacuum_test (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT now()
);

-- Нагружаем ее
INSERT INTO vacuum_test (data) 
SELECT generate_series(1, 100000)::TEXT;

-- Запускаем VACUUM с разными настройками
-- Тест 1: Стандартные настройки
SET vacuum_cost_delay = '2ms';
SET vacuum_cost_limit = 200;
VACUUM (VERBOSE) vacuum_test;

-- Тест 2: Более агрессивно
SET vacuum_cost_delay = '0ms';
SET vacuum_cost_limit = 1000;
VACUUM (VERBOSE) vacuum_test;

-- Сравниваем время в логах:
-- grep "autovacuum.*vacuum_test" /var/log/postgresql/postgresql-*.log
```

### **3. Автоматическая корректировка на лету**
```sql
-- Настройка для конкретной таблицы
ALTER TABLE problem_table SET (
    autovacuum_vacuum_cost_delay = 0,
    autovacuum_vacuum_cost_limit = 1000
);

-- Проверка текущих настроек таблицы
SELECT 
    relname,
    reloptions
FROM pg_class 
WHERE relname = 'problem_table';

-- Временное изменение для текущей сессии
SET vacuum_cost_delay = '0ms';
SET vacuum_cost_limit = 500;
-- Выполнить срочный VACUUM
VACUUM problem_table;
```

---

## 🎯 **Best Practices по параметрам стоимости**

### **Рекомендации по умолчанию:**
```ini
# postgresql.conf
# Для HDD:
vacuum_cost_delay = 2ms
vacuum_cost_limit = 200
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 200

# Для SSD (быстрее):
vacuum_cost_delay = 1ms
vacuum_cost_limit = 400
autovacuum_vacuum_cost_delay = 1ms
autovacuum_vacuum_cost_limit = 400

# Для NVMe (очень быстрые):
vacuum_cost_delay = 0ms  # или очень маленькое значение
vacuum_cost_limit = 1000
```

### **Когда изменять настройки:**

#### **Увеличить лимит (сделать VACUUM агрессивнее):**
```sql
-- Когда:
-- 1. VACUUM не успевает за нагрузкой
-- 2. Много мертвых строк накапливается
-- 3. Есть downtime окна

SET autovacuum_vacuum_cost_limit = 1000;
SET autovacuum_vacuum_cost_delay = 1ms;
```

#### **Уменьшить лимит (сделать VACUUM мягче):**
```sql
-- Когда:
-- 1. VACUUM мешает основным запросам
-- 2. High load на дисковую систему
-- 3. Пользователи жалуются на замедления

SET autovacuum_vacuum_cost_limit = 100;
SET autovacuum_vacuum_cost_delay = 10ms;
```

### **Таблица принятия решений:**
| Симптом | Возможная причина | Решение |
|---------|-------------------|---------|
| **Мертвых строк > 20%** | Autovacuum не успевает | Увеличить `autovacuum_vacuum_cost_limit` до 500-1000 |
| **Дисковый I/O > 80%** | VACUUM перегружает диск | Увеличить `autovacuum_vacuum_cost_delay` до 10-20ms |
| **VACUUM работает часами** | Слишком консервативные настройки | Уменьшить `autovacuum_vacuum_cost_delay` до 0-1ms |
| **Пользователи жалуются на лаги** | VACUUM конкурирует за ресурсы | Настроить `vacuum_cost_page_*` стоимости |
| **Autovacuum никогда не запускается** | `vacuum_cost_delay = 0` для autovacuum? | Проверить `autovacuum_vacuum_cost_delay` |

---

## 🛠️ **Практическое задание**

### **Задание 1: Анализ текущих настроек**
```sql
-- 1. Получите текущие настройки стоимости
SELECT 
    name, 
    setting, 
    unit,
    category,
    short_desc
FROM pg_settings 
WHERE name LIKE '%vacuum_cost%' 
   OR name LIKE '%autovacuum%cost%'
ORDER BY name;

-- 2. Проверьте, как они работают
CREATE TEMP TABLE vacuum_monitor AS
SELECT now() as check_time, * 
FROM pg_stat_user_tables 
WHERE relname = 'your_table';

-- 3. Запустите VACUUM и сравните
VACUUM VERBOSE your_table;

-- 4. Посмотрите разницу
SELECT 
    now() - check_time as time_passed,
    current.n_dead_tup - monitor.n_dead_tup as cleaned_tuples
FROM vacuum_monitor monitor,
     (SELECT * FROM pg_stat_user_tables WHERE relname = 'your_table') current;
```

### **Задание 2: Настройка для конкретного случая**
```sql
-- Ситуация: Таблица orders (10 млн строк, 1000 updates/сек)
-- Проблема: 30% мертвых строк, autovacuum не справляется

-- 1. Анализ
SELECT 
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_percent,
    last_autovacuum,
    age(now(), last_autovacuum) as since_vacuum
FROM pg_stat_user_tables 
WHERE relname = 'orders';

-- 2. Агрессивная настройка autovacuum для этой таблицы
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 50000,
    autovacuum_vacuum_cost_delay = 0,      -- без задержек
    autovacuum_vacuum_cost_limit = 1000,   -- высокий лимит
    autovacuum_freeze_max_age = 100000000
);

-- 3. Глобальная настройка для других таблиц
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '2ms';
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 200;
SELECT pg_reload_conf();
```

### **Задание 3: Мониторинг влияния**
```bash
# Скрипт мониторинга vacuum
#!/bin/bash
echo "=== VACUUM Cost Monitoring ==="
echo "Time: $(date)"
echo ""

psql -U postgres << EOF
-- Текущие настройки
SELECT 
    'Global Settings' as section,
    name,
    setting
FROM pg_settings 
WHERE name IN (
    'vacuum_cost_delay',
    'vacuum_cost_limit',
    'autovacuum_vacuum_cost_delay',
    'autovacuum_vacuum_cost_limit'
)
UNION ALL
SELECT 
    'Table-specific' as section,
    'orders.autovacuum_vacuum_cost_limit' as name,
    split_part(reloptions::text, '=', 2) as setting
FROM pg_class 
WHERE relname = 'orders' 
AND reloptions::text LIKE '%autovacuum_vacuum_cost_limit%'
ORDER BY section, name;

-- Эффективность
SELECT 
    schemaname,
    relname,
    n_dead_tup,
    round(100.0 * n_dead_tup / (n_live_tup + 1), 2) as dead_percent,
    last_autovacuum,
    CASE 
        WHEN n_dead_tup > n_live_tup * 0.1 THEN 'NEEDS ATTENTION'
        ELSE 'OK'
    END as status
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 5;
EOF
```

---

## 📈 **Формулы для расчета**

### **Расчет времени VACUUM:**
```
Предположим:
- Таблица: 1000 страниц
- 50% hit (память), 30% miss (диск), 20% dirty (запись)

Стоимость = (500 * 1) + (300 * 10) + (200 * 20) 
          = 500 + 3000 + 4000 = 7500 единиц

При vacuum_cost_limit = 200:
Циклов = 7500 / 200 = 37.5 ≈ 38 циклов

При vacuum_cost_delay = 2ms:
Общее время пауз = 38 * 2ms = 76ms
```

### **Оптимальные значения для вашей системы:**
```sql
-- Эмпирическое правило:
-- 1. Начните с значений по умолчанию
-- 2. Мониторьте n_dead_tup
-- 3. Если растет - уменьшайте delay или увеличивайте limit
-- 4. Если система лагает - увеличивайте delay

-- Формула для начальной настройки на SSD:
vacuum_cost_limit = shared_buffers / 1000  -- примерно
vacuum_cost_delay = 1ms  -- для SSD
```

---

## 🚨 **Предупреждения**

1. **`vacuum_cost_delay = 0`** отключает систему стоимости → VACUUM может перегрузить систему
2. **Слишком высокий `vacuum_cost_limit`** может вызвать I/O шторм
3. **Настройки для autovacuum и ручного VACUUM раздельны**
4. **Всегда тестируйте на staging** перед production
