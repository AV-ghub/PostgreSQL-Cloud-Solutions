## 📚 **Блок 1: Установка PostgreSQL 14**  
### **1. Ключевые выводы:**

#### **Способы установки (глава 1):**
1. **Пакетная установка** (рекомендовано для production)
   - Ubuntu/Debian: через репозиторий PostgreSQL
   - CentOS/RHEL: через PostgreSQL Yum repository
   - Важно: всегда указывать конкретную версию

2. **Контейнеризация** (для разработки/тестирования)
   - Docker: `postgres:14`
   - Docker Compose для многоконтейнерных сред
   - ⚠️ В production требует careful tuning

3. **Сборка из исходников** (только для специфических патчей)

#### **Важные моменты из книги:**
- **PostgreSQL 14 изменения:**
  - Новый метод шифрования паролей `scram-sha-256` (вместо md5)
  - Проблемы совместимости при апгрейде с версий ≤13
  - Улучшения производительности для высоконагруженных систем
- **Рекомендации по ОС:** Linux (предпочтительно LTS Ubuntu/CentOS)
- **Порты по умолчанию:** 5432, но можно создавать multiple clusters

---

### **2. Современные практики (2024):**

#### **Production-установка на Ubuntu 22.04 LTS:**
```bash
# 1. Импорт ключа и добавление репозитория
sudo apt install -y curl
curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/postgresql.asc
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# 2. Установка PostgreSQL 14 с указанием версии
sudo apt update
sudo apt install -y postgresql-14 postgresql-client-14 postgresql-contrib-14

# 3. Проверка установки
pg_lsclusters
```

#### **Post-install настройки (критически важные):**
```bash
# 1. Настройка пароля для суперпользователя
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'StrongPassword123!';"

# 2. Настройка pg_hba.conf для безопасного доступа
sudo nano /etc/postgresql/14/main/pg_hba.conf
```
**Рекомендуемая конфигурация pg_hba.conf:**
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# Локальные подключения
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# IPv4 подключения (ограничить конкретными IP!)
host    all             all             192.168.1.0/24          scram-sha-256
host    all             all             10.0.0.0/8              scram-sha-256

# IPv6 (отключить если не используется)
host    all             all             ::1/128                 scram-sha-256
```

```bash
# 3. Настройка postgresql.conf
sudo nano /etc/postgresql/14/main/postgresql.conf

# Критические параметры для начала:
listen_addresses = 'localhost,192.168.1.100'  # конкретные IP, не '*'
max_connections = 100  # начальное значение, настроить позже
shared_buffers = 1GB   # 25% от RAM для сервера БД
effective_cache_size = 3GB  # 50-75% от RAM
```

```bash
# 4. Перезапуск и проверка
sudo systemctl restart postgresql
sudo systemctl status postgresql

# 5. Проверка подключения
psql -h localhost -U postgres -d postgres
```

---

### **3. Docker-установка (для разработки):**

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:14-alpine  # alpine для меньшего размера
    container_name: postgres_14
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: SecurePass123!
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    command: >
      postgres 
      -c max_connections=100
      -c shared_buffers=256MB
      -c log_statement=ddl
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

---

### **4. Практическое задание:**

#### **Задание 1: Установка и базовая настройка**
1. Установите PostgreSQL 14 на виртуальную машину (реальную или Vagrant)
2. Настройте безопасный доступ:
   - Измените пароль postgres
   - Настройте pg_hba.conf для доступа только с trusted networks
   - Отключите доступ из интернета (0.0.0.0/0)

#### **Задание 2: Создание тестового кластера**
```sql
-- Создайте нового пользователя и БД
CREATE USER app_user WITH PASSWORD 'UserPass123!';
CREATE DATABASE app_db OWNER app_user;
GRANT ALL PRIVILEGES ON DATABASE app_db TO app_user;

-- Создайте отдельную схему
\c app_db
CREATE SCHEMA app_schema AUTHORIZATION app_user;

-- Настройте search_path
ALTER ROLE app_user SET search_path TO app_schema, public;
```

#### **Задание 3: Проверка совместимости**
1. Подключитесь с клиента, используя:
   - Старый драйвер (с md5)
   - Новый драйвер (scram-sha-256)
2. Проверьте журнал на предмет ошибок аутентификации

---

### **5. Мониторинг установки:**

```sql
-- Проверка версии и сборки
SELECT version();

-- Проверка параметров
SELECT name, setting, unit, context 
FROM pg_settings 
WHERE name IN ('port', 'listen_addresses', 'max_connections', 'shared_buffers');

-- Проверка активных подключений
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- Проверка журналов
sudo tail -f /var/log/postgresql/postgresql-14-main.log
```

---

### **6. Частые проблемы и решения:**

| Проблема | Решение |
|----------|---------|
| Ошибка аутентификации | Проверить pg_hba.conf и метод шифрования |
| Не удается подключиться | Проверить listen_addresses и firewall |
| Нехватка памяти | Настроить shared_buffers (25% RAM) |
| Медленная работа после установки | Проверить параметры shared_buffers, effective_cache_size |

---

### **🎯 Ключевые тезисы для собеседования:**

1. **PostgreSQL 14+ требует scram-sha-256** для паролей
2. **listen_addresses никогда не должно быть '*'** в production
3. **pg_hba.conf - минимальные необходимые права** (принцип least privilege)
4. **Всегда устанавливайте конкретную версию** (не просто 'postgresql')
5. **Docker подходит только для dev/test** - для production нужна тонкая настройка


## 📊 **Параметры памяти: shared_buffers и effective_cache_size**

---

### **1. shared_buffers**

#### **Назначение:**
- **Буферный кэш PostgreSQL** - память для хранения часто используемых страниц данных (по 8KB каждая)
- **Первый уровень кэширования** перед обращением к операционной системе
- Хранит **грязные страницы** (измененные, но еще не записанные на диск)
- Ускоряет **повторное чтение** одних и тех же данных

#### **Архитектурный контекст:**
```
Приложение → PostgreSQL shared_buffers → OS Page Cache → Диск
```

#### **Рекомендации по настройке:**

| Окружение | Рекомендация | Обоснование |
|-----------|--------------|-------------|
| **Development** | 128MB - 1GB | Для тестирования, минимум нагрузки |
| **Dedicated DB Server** | **25% от RAM** (макс 8GB) | Оптимальный баланс для PostgreSQL |
| **High-memory Server** (≥64GB) | 8GB - 16GB | Уменьшение отдачи от shared_buffers после 8GB |
| **Mixed-use Server** (БД + приложение) | 15-20% от RAM | Оставить память для ОС и приложений |

#### **Практические примеры:**

```bash
# Расчет для сервера с 32GB RAM
total_ram=32
shared_buffers=$((total_ram / 4))  # 8GB
echo "Рекомендовано: ${shared_buffers}GB"

# В postgresql.conf
shared_buffers = 8GB  # для 32GB RAM
```

#### **Мониторинг использования:**
```sql
-- Текущее использование shared_buffers
SELECT 
    round(100.0 * (SELECT setting::bigint 
                   FROM pg_settings 
                   WHERE name = 'shared_buffers') / 
          (SELECT setting::bigint 
           FROM pg_settings 
           WHERE name = 'shared_buffers')) as buffers_allocated_percent,
    
    round(100.0 * count(*) * 8192 / 
          (SELECT setting::bigint 
           FROM pg_settings 
           WHERE name = 'shared_buffers'), 2) as buffers_used_percent
FROM pg_buffercache;

-- Детальный анализ
SELECT 
    c.relname,
    count(*) * 8192 / 1024 / 1024 as mb_in_buffer,
    round(100.0 * count(*) / (SELECT setting FROM pg_settings WHERE name='shared_buffers')::integer, 2) as percent_of_buffer
FROM pg_class c
JOIN pg_buffercache b ON b.relfilenode = c.relfilenode
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
GROUP BY c.relname
ORDER BY mb_in_buffer DESC
LIMIT 10;
```

#### **Проблемы при неправильной настройке:**

**Слишком мало shared_buffers:**
- Частые чтения с диска (disk I/O)
- Высокий `buffer hit ratio` в OS cache, но низкий в PostgreSQL cache
- Медленные запросы

**Слишком много shared_buffers:**
- Недостаток памяти для ОС и других процессов
- Увеличение времени checkpoint'ов
- Возможен OOM (Out Of Memory)

---

### **2. effective_cache_size**

#### **Назначение:**
- **Не аллоцирует память!** Это **статическая оценка** для планировщика запросов
- Помогает оптимизатору **выбирать между последовательным и индексным сканированием**
- Сообщает PostgreSQL, сколько памяти **доступно для кэширования** (PostgreSQL + OS cache)

#### **Как работает:**
```
effective_cache_size = shared_buffers + OS page cache + файловый кэш
```
Оптимизатор использует эту информацию: "Если данные могут быть в кэше - используем индекс, если нет - возможно последовательное сканирование"

#### **Рекомендации по настройке:**

| Окружение | Рекомендация | Обоснование |
|-----------|--------------|-------------|
| **Dedicated DB Server** | **50-75% от RAM** | Учитывает shared_buffers + OS cache |
| **SSD/NVMe системы** | Можно ближе к 75% | Быстрый доступ к диску снижает важность точной оценки |
| **HDD системы** | Точнее настраивать (50-60%) | Неверная оценка сильно влияет на производительность |

#### **Практические примеры:**

```bash
# Расчет для сервера с 32GB RAM
total_ram=32
effective_cache_size=$((total_ram * 3 / 4))  # 24GB
echo "Рекомендовано: ${effective_cache_size}GB"

# В postgresql.conf
effective_cache_size = 24GB  # для 32GB RAM
```

#### **Влияние на планы запросов:**

```sql
-- Пример: Без правильного effective_cache_size
EXPLAIN ANALYZE SELECT * FROM large_table WHERE id = 12345;
-- Может выбрать Seq Scan вместо Index Scan

-- С правильным effective_cache_size
SET effective_cache_size = '24GB';
EXPLAIN ANALYZE SELECT * FROM large_table WHERE id = 12345;
-- Выберет Index Scan, если данные в кэше
```

#### **Как проверить текущую эффективность:**
```sql
-- Проверка hit ratio (должно быть > 99% для production)
SELECT 
    round(heap_blks_hit * 100.0 / (heap_blks_hit + heap_blks_read), 2) as heap_hit_ratio,
    round(idx_blks_hit * 100.0 / (idx_blks_hit + idx_blks_read), 2) as idx_hit_ratio
FROM pg_statio_user_tables 
WHERE relname = 'your_table';

-- Мониторинг выбора планов
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;
```

---

### **3. Взаимосвязь параметров**

#### **Балансировка для сервера 32GB RAM:**
```ini
# postgresql.conf
shared_buffers = 8GB           # 25% от RAM
effective_cache_size = 24GB    # 75% от RAM
work_mem = 16MB                # для сортировок
maintenance_work_mem = 512MB   # для VACUUM, CREATE INDEX
wal_buffers = 16MB             # обычно 1/32 от shared_buffers
```

#### **Формула проверки:**
```
shared_buffers + work_mem * max_connections + maintenance_work_mem + wal_buffers + <другие процессы> 
≤ 80% от общей RAM
```

#### **Проверочный скрипт:**
```sql
-- Расчет использования памяти
WITH memory_params AS (
    SELECT 
        name,
        setting::bigint as value,
        unit
    FROM pg_settings 
    WHERE name IN (
        'shared_buffers', 
        'work_mem', 
        'maintenance_work_mem',
        'wal_buffers',
        'max_connections'
    )
    UNION ALL
    SELECT 
        'estimated_total' as name,
        (SELECT setting::bigint FROM pg_settings WHERE name = 'shared_buffers') +
        (SELECT setting::bigint FROM pg_settings WHERE name = 'work_mem') * 
        (SELECT setting::bigint FROM pg_settings WHERE name = 'max_connections') +
        (SELECT setting::bigint FROM pg_settings WHERE name = 'maintenance_work_mem') +
        (SELECT setting::bigint FROM pg_settings WHERE name = 'wal_buffers') as value,
        'bytes' as unit
)
SELECT 
    name,
    pg_size_pretty(value) as size,
    unit,
    round(100.0 * value / (SELECT value FROM memory_params WHERE name = 'estimated_total'), 2) as percent_of_total
FROM memory_params;
```

---

### **4. Практическое задание**

#### **Задание 1: Анализ текущей конфигурации**
1. Подключитесь к PostgreSQL и выполните:
```sql
SELECT 
    name, 
    setting, 
    unit,
    context,
    short_desc
FROM pg_settings 
WHERE name IN (
    'shared_buffers', 
    'effective_cache_size',
    'work_mem',
    'maintenance_work_mem',
    'wal_buffers'
);
```

2. Рассчитайте оптимальные значения для вашего сервера

#### **Задание 2: Мониторинг hit ratio**
```sql
-- Установите расширение (если не установлено)
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- Проверьте buffer hit ratio за последний час
SELECT 
    now() - query_start as running_time,
    query,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) as hit_ratio
FROM pg_stat_activity 
WHERE state = 'active' 
AND shared_blks_hit + shared_blks_read > 0;
```

#### **Задание 3: Тест влияния effective_cache_size**
1. Создайте тестовую таблицу:
```sql
CREATE TABLE test_cache AS 
SELECT generate_series(1, 1000000) as id, 
       md5(random()::text) as data;

CREATE INDEX idx_test_cache ON test_cache(id);
```

2. Протестируйте с разными значениями:
```sql
SET effective_cache_size = '1GB';
EXPLAIN ANALYZE SELECT * FROM test_cache WHERE id = 500000;

SET effective_cache_size = '24GB';
EXPLAIN ANALYZE SELECT * FROM test_cache WHERE id = 500000;
```

---

### **5. Частые ошибки и решения**

| Ошибка | Симптомы | Решение |
|--------|----------|---------|
| **Слишком большой shared_buffers** | Частые checkpoint'ы, OOM killer | Уменьшить до 25% RAM, не более 8GB |
| **Слишком маленький effective_cache_size** | Неоптимальные планы запросов, Seq Scan вместо Index Scan | Увеличить до 50-75% RAM |
| **Несбалансированная память** | Конфликты с ОС, swapping | Проверить: `shared_buffers + work_mem * max_connections ≤ 80% RAM` |
| **Игнорирование файлового кэша** | Неверная оценка effective_cache_size на SSD | Учитывать быстрый доступ к SSD |

---

### **🎯 Ключевые тезисы для собеседования:**

1. **shared_buffers - это реальная выделенная память**, effective_cache_size - **только оценка** для оптимизатора
2. **25% RAM для shared_buffers** - золотое правило для dedicated серверов
3. **effective_cache_size помогает выбирать между Index Scan и Seq Scan**
4. **Всегда проверяйте hit ratio** - должен быть >99% для production
5. **На SSD можно быть менее точным** с effective_cache_size, на HDD - критически важно

---

### **📈 Мониторинг в production:**

```bash
# Скрипт для мониторинга памяти
#!/bin/bash
echo "=== PostgreSQL Memory Usage ==="
psql -U postgres -c "
SELECT 
    'shared_buffers' as parameter,
    setting as current_value,
    pg_size_pretty(setting::bigint) as pretty,
    '8GB' as recommended  -- измените под ваш сервер
FROM pg_settings WHERE name = 'shared_buffers'
UNION ALL
SELECT 
    'effective_cache_size',
    setting,
    pg_size_pretty(setting::bigint),
    '24GB'  -- измените под ваш сервер
FROM pg_settings WHERE name = 'effective_cache_size'
UNION ALL
SELECT 
    'buffer_hit_ratio',
    round(100 * blks_hit / (blks_hit + blks_read)::numeric, 2)::text,
    '%',
    '>99%'
FROM pg_stat_database WHERE datname = current_database();
"
```

---

## ❓ **Вопросы для самопроверки:**

1. Почему shared_buffers не должен превышать 8GB даже на серверах с 128GB RAM?
2. Как effective_cache_size влияет на выбор между Index Scan и Seq Scan?
3. Как рассчитать максимальное значение work_mem, чтобы не исчерпать память?
4. Что такое "double caching" и как его избежать?
5. Как мониторить эффективность current configuration в реальном времени?


## ❓ **Ответы на вопросы самопроверки:**

### 1. **Почему shared_buffers не должен превышать 8GB даже на серверах с 128GB RAM?**
- **Закон убывающей отдачи** - после 8GB прирост производительности минимален
- **ОС лучше управляет кэшем** для разнообразных рабочих нагрузок
- **Checkpoint'ы становятся длиннее** - нужно сбрасывать больше грязных страниц
- **Память нужна другим процессам** - ВОУ, autovacuum, соединениям
- **Эмпирическое правило** - проверено на production-нагрузках

### 2. **Как effective_cache_size влияет на выбор между Index Scan и Seq Scan?**
- **Большое значение** → оптимизатор думает: "данные в кэше" → выбирает **Index Scan**
- **Маленькое значение** → оптимизатор думает: "данные на диске" → выбирает **Seq Scan** (меньше random I/O)
- **Пример**: Если effective_cache_size = 1GB, а таблица 2GB - планировщик предпочтет Seq Scan
- **Критично для HDD**, менее важно для SSD/NVMe

### 3. **Как рассчитать максимальное значение work_mem, чтобы не исчерпать память?**
```
Максимальный work_mem = (Доступная RAM * 0.8 - shared_buffers) / max_connections
```
**Пример для 32GB RAM, 100 соединений:**
```
Доступно: 32GB * 0.8 = 25.6GB
shared_buffers: 8GB
Остаток: 17.6GB
work_mem max: 17.6GB / 100 = ~180MB
```
**Но лучше:** Настраивать на уровне запросов: `SET LOCAL work_mem = '256MB'`

### 4. **Что такое "double caching" и как его избежать?**
- **Double caching** = данные в shared_buffers И в OS page cache → лишнее использование памяти
- **Возникает когда:** O_DIRECT не используется, данные читаются через ОС
- **Решение в PostgreSQL:** Страницы читаются напрямую в shared_buffers (нет double caching)
- **Но есть нюанс:** WAL файлы и временные файлы всё равно кэшируются ОС

### 5. **Как мониторить эффективность конфигурации в реальном времени?**
```sql
-- 1. Hit ratio (должен быть > 99%)
SELECT 
    datname,
    blks_hit,
    blks_read,
    round(100 * blks_hit / (blks_hit + blks_read + 1)::numeric, 2) as hit_ratio
FROM pg_stat_database;

-- 2. Буферный кэш
SELECT 
    c.relname,
    count(*) as buffers,
    round(count(*) * 100.0 / (SELECT setting FROM pg_settings WHERE name='shared_buffers')::integer, 2) as percent
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 10;

-- 3. Память на соединения
SELECT 
    COUNT(*) as active_connections,
    SUM(CASE WHEN state = 'active' THEN 1 ELSE 0 END) as truly_active,
    setting::integer as max_connections,
    round(100.0 * COUNT(*) / setting::integer, 2) as connection_usage_percent
FROM pg_stat_activity, 
     (SELECT setting FROM pg_settings WHERE name = 'max_connections') as max_conn
GROUP BY setting;
```

---

## 📊 **Шпаргалка по памяти PostgreSQL:**

| Параметр | Назначение | Формула | Пример для 32GB |
|----------|------------|---------|----------------|
| **shared_buffers** | Кэш данных | 25% RAM, макс 8GB | 8GB |
| **effective_cache_size** | Планировщику | 50-75% RAM | 24GB |
| **work_mem** | Сортировки/хеши | (RAM*0.8 - shared_buffers)/max_conn | 16-256MB |
| **maintenance_work_mem** | VACUUM, INDEX | 1-2GB, макс 8GB | 1GB |
| **wal_buffers** | WAL буфер | 16MB (1/32 shared_buffers) | 16MB |

---
