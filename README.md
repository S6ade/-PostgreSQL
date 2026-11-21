# -PostgreSQL
# Отказоустойчивый кластер PostgreSQL (Active/Standby)

Этот проект демонстрирует развертывание отказоустойчивой инфраструктуры баз данных с использованием **PostgreSQL 14** и потоковой репликации (**Streaming Replication**). Проект имитирует реальную задачу системного администратора: настройку резервного сервера (реплики), который непрерывно синхронизирует данные с основного сервера (мастера) и готов взять на себя нагрузку в случае аварии (**Failover**).

---

## 🏗️ Архитектура

Для реализации проекта используются **три виртуальные машины** в Yandex Cloud на базе **Ubuntu 22.04 LTS**:

1. **Master (`IP_Master`) — Основной сервер:** Принимает все запросы на чтение и запись, генерирует WAL-логи (Write-Ahead Logging) для передачи на реплику.
2. **Replica (`IP_Replica`) — Резервный сервер:** Непрерывно копирует данные с Master. Работает в режиме `hot_standby` (доступен только для чтения).
3. **Client (`IP_Client`) — Клиентская машина:** Рабочая станция администратора, с которой выполняется импорт данных и проверка работы БД (установлен только пакет `postgresql-client-14`).

### Схема взаимодействия
```text
[ Client ] ──(SQL / 5432)──> [ Master ] ══(Streaming Replication / 5432)══> [ Replica (Standby) ]
```

---

## 📋 Что реализовано в рамках проекта

* **Настройка сети:** Конфигурация групп безопасности (Security Groups) Yandex Cloud для обеспечения сетевой связности по портам SSH (`22`) и PostgreSQL (`5432`).
* **Настройка Master:** Конфигурация `postgresql.conf` (`wal_level = replica`), создание выделенной роли для репликации (`repuser`), настройка правил доступа в `pg_hba.conf`.
* **Управление БД и данными:** Создание базы данных `sysadmin_db` и пользователя `sysadmin`, импорт схемы и тестового набора данных `students.sql` (таблица `студенты` с записями формата `(id, 'ФИО', группа)` с ID от `4` до `9999` — всего 9996 строк).
* **Настройка Replica:** Инициализация резервного узла с помощью утилиты `pg_basebackup -R` и запуск фоновой репликации.
* **Симуляция аварии (Failover):** Принудительная остановка Master, перевод Replica в режим нового главного сервера (`pg_ctlcluster promote`) и проверка работоспособности на запись.

---

## 🛠️ Технологический стек

* **Облачная платформа:** Yandex Cloud (Compute Cloud, VPC Security Groups)
* **СУБД:** PostgreSQL 14
* **ОС:** Linux (Ubuntu 22.04 LTS)
* **Инструменты:** Bash, SQL, `psql`, `pg_basebackup`, `pg_ctlcluster`

---

## 🚀 Пошаговое развертывание

> **Примечание:** Перед выполнением команд замените плейсхолдеры `IP_Master`, `IP_Replica` и `IP_Client` на реальные внутренние IP-адреса ваших виртуальных машин.

### Шаг 1. Подготовка инфраструктуры и сети

1. Создайте 3 виртуальные машины с **Ubuntu 22.04**.
2. В настройках **Групп безопасности (Security Groups)** Yandex Cloud откройте входящий и исходящий трафик для подсети ваших ВМ (например, `10.128.0.0/15`):
   * `TCP`, порт `22` (SSH)
   * `TCP`, порт `5432` (PostgreSQL)
3. Установите сервер PostgreSQL на **Master** и **Replica**:
   ```bash
   sudo apt update && sudo apt install postgresql -y
   ```
4. Установите только клиентские утилиты на **Client**:
   ```bash
   sudo apt update && sudo apt install postgresql-client-14 -y
   ```

---

### Шаг 2. Настройка основного сервера (Master)

*Выполняется на машине **Master**.*

#### 2.1. Конфигурация `postgresql.conf`
Откройте файл настроек:
```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```
Раскомментируйте и измените следующие параметры:
```ini
listen_addresses = '*'      # Разрешаем подключения по сети
wal_level = replica         # Включаем запись WAL-логов для репликации
```

#### 2.2. Создание пользователя для репликации
Зайдите в консоль PostgreSQL:
```bash
sudo -u postgres psql
```
Создайте роль `repuser` с правами только на репликацию:
```sql
CREATE ROLE repuser WITH REPLICATION LOGIN PASSWORD 'Super1';
\q
```

#### 2.3. Настройка правил доступа `pg_hba.conf`
Откройте файл:
```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```
Добавьте в **самое начало файла** (выше остальных правил) разрешения для реплики и клиента:
```text
# Разрешаем потоковую репликацию с сервера Replica
host    replication     repuser         IP_Replica/32           scram-sha-256

# Разрешаем подключение клиента к рабочей базе
host    sysadmin_db     sysadmin        IP_Client/32            scram-sha-256
```

#### 2.4. Применение настроек и создание рабочей БД
Перезапустите службу PostgreSQL и создайте базу данных:
```bash
sudo systemctl restart postgresql
sudo -u postgres psql
```
Выполните в консоли `psql`:
```sql
CREATE DATABASE sysadmin_db;
CREATE USER sysadmin WITH LOGIN PASSWORD 'ВашСложныйПароль';
GRANT ALL PRIVILEGES ON DATABASE sysadmin_db TO sysadmin\(; \c\) sysadmin_db
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO sysadmin;
\q
```

---

### Шаг 3. Импорт данных с клиентской машины (Client)

*Выполняется на машине **Client**.*

1. Скачайте SQL-скрипты структуры и данных:
   ```bash
   wget https://sysadmin.education-services.ru/downloads/schema.sql
   wget https://sysadmin.education-services.ru/downloads/students.sql
   ```
2. Импортируйте структуру таблицы и массив данных на **Master** от имени пользователя `sysadmin`:
   ```bash
   psql -h IP_Master -U sysadmin -d sysadmin_db -f schema.sql
   psql -h IP_Master -U sysadmin -d sysadmin_db -f students.sql
   ```
3. Проверьте количество загруженных записей (в файле `students.sql` содержится **9996** записей с ID от 4 до 9999):
   ```bash
   psql -h IP_Master -U sysadmin -d sysadmin_db -c "SELECT count(*) FROM студенты;"
   ```

---

### Шаг 4. Настройка резервного сервера (Replica)

*Выполняется на машине **Replica**.*

1. Остановите службу PostgreSQL и очистите каталог данных:
   ```bash
   sudo systemctl stop postgresql
   sudo -u postgres rm -rf /var/lib/postgresql/14/main/*
   ```
2. Выполните базовое резервное копирование с Master (флаг `-R` автоматически создаст файл `standby.signal` и пропишет параметры подключения в `postgresql.auto.conf`):
   ```bash
   cd /tmp
   sudo -u postgres pg_basebackup -h IP_Master -D /var/lib/postgresql/14/main -U repuser -P -v -R
   ```
   *(При запросе введите пароль пользователя `repuser`)*
3. Запустите PostgreSQL на реплике:
   ```bash
   sudo systemctl start postgresql
   ```
4. Убедитесь, что данные синхронизировались, а сервер находится в режиме восстановления (Standby):
   ```bash
   sudo -u postgres psql -d sysadmin_db -c "SELECT count(*) FROM студенты;"
   sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
   ```
   > Ожидаемый результат для `pg_is_in_recovery()`: `t` (`true`).

---

### Шаг 5. Симуляция аварии (Failover)

#### 5.1. Имитация отказа Master
На машине **Master** принудительно остановите СУБД:
```bash
sudo systemctl stop postgresql
```

#### 5.2. Повышение статуса Replica до Master (Promote)
На машине **Replica** переведите сервер в основной режим:
```bash
sudo pg_ctlcluster 14 main promote
```

#### 5.3. Проверка работы нового Master на запись
1. Убедитесь, что бывшая реплика вышла из режима восстановления:
   ```bash
   sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
   ```
   > Ожидаемый результат: `f` (`false`).

2. Подключитесь к базе и выполните операции записи и обновления (`INSERT` / `UPDATE`):
   ```bash
   sudo -u postgres psql -d sysadmin_db
   ```
   ```sql
   INSERT INTO студенты VALUES (3, 'Тестовый пользователь', 1);
   UPDATE студенты SET фио = 'Иванов Иван' WHERE id = 3;
   SELECT * FROM студенты WHERE id = 3;
   \q
   ```
   > Если запрос `SELECT` возвращает обновленную запись, переключение прошло успешно и кластер снова готов к работе на запись.

---

## ⚠️ Возможные ошибки и их решение (Troubleshooting)

| Ошибка | Причина | Решение |
| :--- | :--- | :--- |
| `nc: connect to ... failed: Connection refused` | Закрыт порт `5432` в облаке или PostgreSQL слушает только `localhost`. | Откройте порт `5432` в Security Groups Yandex Cloud. В `postgresql.conf` задайте `listen_addresses = '*'`. |
| `no pg_hba.conf entry for host ...` | Изменился внутренний IP реплики/клиента или правило добавлено в конец файла. | Проверьте актуальные IP машин, перенесите правила в самый верх `pg_hba.conf` на Master и выполните `sudo systemctl reload postgresql`. |
| `must be member of role "sysadmin"` | Скрипт `schema.sql` запускается от имени пользователя без прав на выдачу привилегий. | Запускайте импорт с флагом `-U sysadmin` (или от суперпользователя `postgres`). |
| Количество строк меньше 9999 (`9996`) | В основном дампе `students.sql` нумерация записей начинается с `id = 4` (заканчивается `9999`). | Это штатное поведение. Наличие `9996` строк подтверждает корректный импорт и работу репликации. |
