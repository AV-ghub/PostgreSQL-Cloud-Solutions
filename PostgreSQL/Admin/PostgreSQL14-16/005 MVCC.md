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
