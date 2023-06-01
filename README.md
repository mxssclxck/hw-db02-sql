# Домашнее задание к занятию "6.2. SQL"
# Никоноров Денис - FOPS-6

## Задача 1

Используя docker поднимите инстанс PostgreSQL (версию 12) c 2 volume, 
в который будут складываться данные БД и бэкапы.

Приведите получившуюся команду или docker-compose манифест.

```yml
version: '3'
services:
  postgres:
    image: postgres:12
    container_name: postgres
    volumes:
      - ./pgdata:/var/lib/postgresql/data
      - ./pgbackups:/backups
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
volumes:
  pgdata:
  pgbackups:
```

## Задача 2

В БД из задачи 1: 
- создайте пользователя test-admin-user и БД test_db
- в БД test_db создайте таблицу orders и clients (спeцификация таблиц ниже)
- предоставьте привилегии на все операции пользователю test-admin-user на таблицы БД test_db
- создайте пользователя test-simple-user  
- предоставьте пользователю test-simple-user права на SELECT/INSERT/UPDATE/DELETE данных таблиц БД test_db

Таблица orders:
- id (serial primary key)
- наименование (string)
- цена (integer)

Таблица clients:
- id (serial primary key)
- фамилия (string)
- страна проживания (string, index)
- заказ (foreign key orders)

Приведите:
- итоговый список БД после выполнения пунктов выше,
- описание таблиц (describe)
- SQL-запрос для выдачи списка пользователей с правами над таблицами test_db
- список пользователей с правами над таблицами test_db

Подключаемся docker контейнеру 
```bash
docker exec -it postgres psql -U postgres
```

Создаем пользователя test-admin-user и БД test_db:

```SQL
CREATE USER "test-admin-user" WITH PASSWORD 'test-admin-password';
CREATE DATABASE test_db;
```

Делаем активной базу test_db. Создаем таблицы orders и clients:

```SQL
\c test_db;

CREATE TABLE orders (
  id SERIAL PRIMARY KEY NOT NULL,
  name VARCHAR(255) NOT NULL,
  price INTEGER
);

CREATE TABLE clients (
  id SERIAL PRIMARY KEY NOT NULL,
  last_name VARCHAR(255) NOT NULL,
  country VARCHAR(255) NOT NULL,
  order_id INTEGER REFERENCES orders(id)
);
```

Предоставляем привилегии пользователю test-admin-user на все операции в таблицах БД test_db:

```SQL
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO "test-admin-user";
```

Создаем пользователя test-simple-user и предоставляем ему права на SELECT/INSERT/UPDATE/DELETE данных таблиц БД test_db:

```SQL
CREATE USER "test-simple-user" WITH PASSWORD 'test-simple-password';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO "test-simple-user";
```

1. Итоговый список БД:

```sql
test_db=# \l
                                     List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |       Access privileges        
-----------+----------+----------+------------+------------+--------------------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres                   +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres                   +
           |          |          |            |            | postgres=CTc/postgres
 test_db   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =Tc/postgres                  +
           |          |          |            |            | postgres=CTc/postgres         +
           |          |          |            |            | "test-admin-user"=CTc/postgres
(4 rows)
```

2. Описание таблиц (describe)

```sql
test_db=# \d+ orders
                                                        Table "public.orders"
 Column |          Type          | Collation | Nullable |              Default               | Storage  | Stats target | Description 
--------+------------------------+-----------+----------+------------------------------------+----------+--------------+-------------
 id     | integer                |           | not null | nextval('orders_id_seq'::regclass) | plain    |              | 
 name   | character varying(255) |           | not null |                                    | extended |              | 
 price  | integer                |           |          |                                    | plain    |              | 
Indexes:
    "orders_pkey" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "clients" CONSTRAINT "clients_order_id_fkey" FOREIGN KEY (order_id) REFERENCES orders(id)
Access method: heap

test_db=# \d+ clients
                                                         Table "public.clients"
  Column   |          Type          | Collation | Nullable |               Default               | Storage  | Stats target | Description 
-----------+------------------------+-----------+----------+-------------------------------------+----------+--------------+-------------
 id        | integer                |           | not null | nextval('clients_id_seq'::regclass) | plain    |              | 
 last_name | character varying(255) |           | not null |                                     | extended |              | 
 country   | character varying(255) |           | not null |                                     | extended |              | 
 order_id  | integer                |           |          |                                     | plain    |              | 
Indexes:
    "clients_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "clients_order_id_fkey" FOREIGN KEY (order_id) REFERENCES orders(id)
Access method: heap
```

3. SQL-запрос для выдачи списка пользователей с правами над таблицами test_db

```sql
SELECT table_name, grantee, privilege_type 
FROM information_schema.role_table_grants 
WHERE table_name IN ('clients', 'orders')
  AND grantee <> 'postgres';
```

4. Список пользователей с правами над таблицами test_db

```sql
 table_name |     grantee      | privilege_type 
------------+------------------+----------------
 clients    | test-admin-user  | INSERT
 clients    | test-admin-user  | SELECT
 clients    | test-admin-user  | UPDATE
 clients    | test-admin-user  | DELETE
 clients    | test-admin-user  | TRUNCATE
 clients    | test-admin-user  | REFERENCES
 clients    | test-admin-user  | TRIGGER
 clients    | test-simple-user | INSERT
 clients    | test-simple-user | SELECT
 clients    | test-simple-user | UPDATE
 clients    | test-simple-user | DELETE
 orders     | test-admin-user  | INSERT
 orders     | test-admin-user  | SELECT
 orders     | test-admin-user  | UPDATE
 orders     | test-admin-user  | DELETE
 orders     | test-admin-user  | TRUNCATE
 orders     | test-admin-user  | REFERENCES
 orders     | test-admin-user  | TRIGGER
 orders     | test-simple-user | INSERT
 orders     | test-simple-user | SELECT
 orders     | test-simple-user | UPDATE
 orders     | test-simple-user | DELETE
(22 rows)

```

## Задача 3

Используя SQL синтаксис - наполните таблицы следующими тестовыми данными:

Таблица orders

|Наименование|цена|
|------------|----|
|Шоколад| 10 |
|Принтер| 3000 |
|Книга| 500 |
|Монитор| 7000|
|Гитара| 4000|

Таблица clients

|ФИО|Страна проживания|
|------------|----|
|Иванов Иван Иванович| USA |
|Петров Петр Петрович| Canada |
|Иоганн Себастьян Бах| Japan |
|Ронни Джеймс Дио| Russia|
|Ritchie Blackmore| Russia|

Используя SQL синтаксис:
- вычислите количество записей для каждой таблицы 
- приведите в ответе:
    - запросы 
    - результаты их выполнения.

Запросы на добавление данных в таблицы:

```sql
INSERT INTO orders VALUES (1, 'Шоколад', 10), (2, 'Принтер', 3000), (3, 'Книга', 500), (4, 'Монитор', 7000), (5, 'Гитара', 4000);
```

```sql
INSERT INTO clients VALUES (1, 'Иванов Иван Иванович', 'USA'), (2, 'Петров Петр Петрович', 'Canada'), (3, 'Иоганн Себастьян Бах', 'Japan'), (4, 'Ронни Джеймс Дио', 'Russia'), (5, 'Ritchie Blackmore', 'Russia');
```
Запросы вычисляющие количество записей в каждой таблице:

```sql
test_db=# select count(1) from clients;
 count 
-------
     5
(1 row)

test_db=# select count(1) from orders;
 count 
-------
     5
(1 row)
```

## Задача 4

Часть пользователей из таблицы clients решили оформить заказы из таблицы orders.

Используя foreign keys свяжите записи из таблиц, согласно таблице:

|ФИО|Заказ|
|------------|----|
|Иванов Иван Иванович| Книга |
|Петров Петр Петрович| Монитор |
|Иоганн Себастьян Бах| Гитара |

Приведите SQL-запросы для выполнения данных операций.

Приведите SQL-запрос для выдачи всех пользователей, которые совершили заказ, а также вывод данного запроса.
 
Подсказк - используйте директиву `UPDATE`.

```sql
test_db=# UPDATE clients SET order_id = (SELECT id FROM orders WHERE name = 'Книга') WHERE last_name = 'Иванов Иван Иванович'; 
UPDATE clients SET order_id = (SELECT id FROM orders WHERE name = 'Монитор') WHERE last_name = 'Петров Петр Петрович'; 
UPDATE clients SET order_id = (SELECT id FROM orders WHERE name = 'Гитара') WHERE last_name = 'Иоганн Себастьян Бах';
UPDATE 1
UPDATE 1
UPDATE 1


test_db=# SELECT clients.last_name, orders.name       
FROM clients                                                                                                           
JOIN orders ON clients.order_id = orders.id;
      last_name       |  name   
----------------------+---------
 Иванов Иван Иванович | Книга
 Петров Петр Петрович | Монитор
 Иоганн Себастьян Бах | Гитара
(3 rows)

```

```sql
select last_name, country, name from clients c join orders o on c.order_id = o.id;


      last_name       | country |  name   
----------------------+---------+---------
 Иванов Иван Иванович | USA     | Книга
 Петров Петр Петрович | Canada  | Монитор
 Иоганн Себастьян Бах | Japan   | Гитара
(3 rows)

```



## Задача 5

Получите полную информацию по выполнению запроса выдачи всех пользователей из задачи 4 
(используя директиву EXPLAIN).

Приведите получившийся результат и объясните что значат полученные значения.

```sql
test_db=# explain select last_name, country, name from clients c join orders o on c.order_id = o.id;
                                QUERY PLAN                                
--------------------------------------------------------------------------
 Hash Join  (cost=11.57..24.20 rows=70 width=1548)
   Hash Cond: (o.id = c.order_id)
   ->  Seq Scan on orders o  (cost=0.00..11.40 rows=140 width=520)
   ->  Hash  (cost=10.70..10.70 rows=70 width=1036)
         ->  Seq Scan on clients c  (cost=0.00..10.70 rows=70 width=1036)
(5 rows)

```

Видим план выполнения запроса. Ключевые операторы: 
- `Seq Scan` — последовательный перебор строк таблицы;
- `Hash Cond` — условие для соединения с помощью хеш-таблицы;
- `Hash Join` — соединение с помощью хеш-таблицы.

План запроса измеряется в так называемых (условных) единицах стоимости (`cost`). Чем выше значения стоимости, тем дольше будет выполняться запрос. Атрибут `rows` указывает на количество обрабатываемых строк. Запрос читается снизу вверх.

## Задача 6

Создайте бэкап БД test_db и поместите его в volume, предназначенный для бэкапов (см. Задачу 1).

Остановите контейнер с PostgreSQL (но не удаляйте volumes).

Поднимите новый пустой контейнер с PostgreSQL.

Восстановите БД test_db в новом контейнере.

Приведите список операций, который вы применяли для бэкапа данных и восстановления. 

Создадим dump БД test_db в контейнере netology_postgres:
  
```bash
 docker exec -it postgres /bin/bash
 ```

```bash
pg_dump -U postgres -d test_db > /backups/test_db.backup
```
создаем новый контейнер 
```bash
docker run --rm -d -e POSTGRES_USER=test-admin-user -e POSTGRES_PASSWORD=test-admin-password -e POSTGRES_DB=test_db -v ~/NETOLOGY/db02-sql/pgbackups/:/backups -p 5433:5432 --name psql2 postgres:12
```
Вывод контейнеров

```bash
docker ps -a
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS                     PORTS                                       NAMES
c16a1203ef83   postgres:12            "docker-entrypoint.s…"   2 seconds ago    Up 1 second                0.0.0.0:5433->5432/tcp, :::5433->5432/tcp   psql2
684c4a38fa7d   postgres:12            "docker-entrypoint.s…"   18 minutes ago   Exited (0) 9 minutes ago                                               postgres12
531e8ee0338a   chatgpt-telegram-bot   "python bot/main.py"     5 days ago       Up 2 days                                                              chatgpt-telegram-bot
```
```bash
docker exec -it psql2 /bin/bash
```
Проверяем наличие бэкапа в volume. Разворачиваем бэкап

```bash
root@c16a1203ef83:/# ls backups/
test_db.backup
root@c16a1203ef83:/# psql -h localhost -U test-admin-user -f /backups/test_db.backup test_db
```