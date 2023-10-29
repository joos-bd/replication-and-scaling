# replication-and-scaling
Репликация и масштабирование

**Репликация**

**Репликация** — это процесс, под которым понимается копирование данных из одного источника на другой (или на множество других) и наоборот. С точки зрения базы данных — это механизм копирования базы данных и создания копий или дополнений существующих объектов

**Виды репликации**
- **Синхронная** - если данная реплика обновляется, все другие реплики того же фрагмента данных также должны быть обновлены в одной и той же транзакции. Логически это означает, что существует лишь одна версия данных.
- **Асинхронная** — обновление одной реплики распространяется на другие спустя некоторое время, а не в той же транзакции. Таким образом, при асинхронной репликации вводится задержка или время ожидания, в течение которого отдельные реплики могут быть фактически не идентичными

**Преимущества репликации**
- повышение производительности чтения данных;
- повышение отказоустойчивости;
- распространение данных между серверами повышает надежность, доступность и скорость;
- распределение нагрузки;
- тестирование новых конфигураций;
- резервное копирование.

**Репликация типа Master-Slave**

**Репликация типа Master-Slave** часто используется для обеспечения отказоустойчивости приложений. Кроме этого, она позволяет распределить нагрузку на базу данных между несколькими серверами (репликами)
- **Master** — это основной сервер БД, куда поступают все данные. Все изменения в данных (добавление, обновление, удаление) должны происходить на этом сервере.
- **Slave** — это вспомогательный сервер БД, который копирует все данные с мастера. С этого сервера следует читать данные. Таких серверов может быть несколько.

Репликация выполняется по следующим шагам:
- Master записывает изменения данных в журнал (binary log);
- Slave копирует изменения двоичного журнала в свой, который называется журналом ретрансляции (relay log);
- Slave воспроизводит изменения из журнала ретрансляции,применяя их к собственным данным.

Репликация бывает двух подходов: **покомандная и потоковая**
- **Покомандная репликация** — в журнал master протоколируются запросы изменения данных (INSERT, UPDATE, DELETE), на slave повторяются.
- **Построчной репликации** в журнале запишутся изменения данных, тоже произойдет и на slave

**Репликация master-master**

**Репликация master-master** позволяет копировать данные с одного сервера на другой. Эта конфигурация добавляет избыточность и повышает эффективность при обращении к данным. Master-Master репликации – это настройка обычной Master-Slave репликации, только в обе стороны (каждый сервер является мастером и слейвом одновременно)

**Настройка master-slave репликации**

Установка master:
```
docker run -d --name replication-master -e MYSQL_ALLOW_EMPTY_PASSWORD=true -v
~/path/to/world/dump:/docker-entrypoint-initdb.d mysql:8.0
```
Установка slave:
```
docker run -d --name replication-slave -e MYSQL_ALLOW_EMPTY_PASSWORD=true
mysql:8.0
```
Для реализации взаимодействия создадим мост и сеть:
```
docker network create replication
docker network connect replication replication-master
docker network connect replication replication-slave
```
Для управления реализации настроек обновим и установим в контейнеры инструменты.

Для Master:
```
docker exec replication-master apt-get update && docker exec replication-master
apt-get install -y nano
```
Для Slave:
```
docker exec replication-slave apt-get update && docker exec replication-slave
apt-get install -y nano
```
Создадим учетную запись Master для сервера репликации:
```
docker exec -it replication-master mysql
```
В контейнере выполним:
```
mysql> CREATE USER 'replication'@'%';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
```
Изменим конфигурацию сервера:
```
docker exec -it replication-master bash
~ nano /etc/mysql/my.cnf
```
my.cnf -> секция [mysqld] добавляем следующие параметры:
```
server_id = 1
log_bin = mysql-bin
```
При изменении конфигурации сервера требуется перезагрузка:
```
docker restart replication-master
```
После требуется зайти в контейнер и проверить состояние:
```
docker exec -it replication-master mysql
mysql> SHOW MASTER STATUS:
```
Следующим шагом требуется выполнить слепок системы и заблокировать все изменения на сервер:
```
mysql> FLUSH TABLES WITH READ LOCK;
```
После данных манипуляций выхода из контейнера и выполняем процесс mysqldump для экспорта базы данных, например:
```
docker exec replication-master mysqldump world > /path/to/dump/on/host/world.sql
```
После, следует зайти обратно в контейнер и вывести настройки master сервера (они понадобятся при настройке slave):
```
docker exec -it replication-master mysql
mysql> SHOW MASTER STATUS;
```
**!!! Важно запомнить File и Position**

Снимаем блокировку базы данных:
```
mysql> UNLOCK TABLES;
```
Master готов, переходим к slave:
```
docker cp /path/to/dump/on/host/world.sql replication-slave:/tmp/world.sql
docker exec -it replication-slave mysql
mysql> CREATE DATABASE `world`;
docker exec -it replication-slave bash
~ mysql world < /tmp/world.sql
```
Открываем конфигурационный файл на Slave my.cnf
```
docker exec -it replication-slave bash
~ nano /etc/mysql/my.cnf
```
my.cnf -> секция [mysqld] добавляем следующие параметры:
```
log_bin = mysql-bin
server_id = 2
relay-log = /var/lib/mysql/mysql-relay-bin
relay-log-index = /var/lib/mysql/mysql-relay-bin.index
read_only = 1
```
Перезагружаем Slave:
```
docker restart replication-slav
```
Следующим шагом требуется прописать в базе данных на сервер slave, кто является master и данные полученные в File и Position:
```
docker exec -it replication-slave mysql
mysql> CHANGE MASTER TO MASTER_HOST='replication-master',
MASTER_USER='replication', MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=156;
```
Далее запускаем журнал ретрансляции, и проверим статус операций
```
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G
```
**Ключевые настройки**
- Slave_IO_State, Slave_SQL_State — состояние IO потока, принимающего двоичный журнал с мастера, и состояние потока, применяющего журнал ретрансляции.
- Read_Master_Log_Pos — последняя позиция, прочитанная из журнала мастера.
- Relay_Master_Log_File — текущий файл журнала мастера.
- Seconds_Behind_Master — отставание данных в секундах.
- Last_IO_Error, Last_SQL_Error — ошибки репликации, если они есть

**Настройка master-master репликации**

Режим Master-Master:
```
docker run -d --name replication-master-one -e MYSQL_ALLOW_EMPTY_PASSWORD=true
-v ~/path/to/world/dump:/docker-entrypoint-initdb.d mysql:8.0
docker run -d --name replication-master-two -e MYSQL_ALLOW_EMPTY_PASSWORD=true
-v ~/path/to/world/dump:/docker-entrypoint-initdb.d mysql:8.0
```
Конфигурационный файл my.conf должен содержать:
```
server_id = 1
log_bin = mysql-bin #на первом
server_id = 2
log_bin = mysql-bin #на втором
```
На втором сервере выполнить:
```
slave stop;
CHANGE MASTER TO MASTER_HOST = 'replication-master-one', MASTER_USER =
'replicator', MASTER_PASSWORD = 'password', MASTER_LOG_FILE =
'mysql-bin.000001', MASTER_LOG_POS = 107;
slave start;
```
На первом:
```
slave stop;
CHANGE MASTER TO MASTER_HOST = 'replication-master-two', MASTER_USER =
'replicator', MASTER_PASSWORD = 'password', MASTER_LOG_FILE =
'mysql-bin.000001', MASTER_LOG_POS = 107;
slave start;
```
Важно добавить пользователей на обоих серверах:
```
create user 'replicator'@'%' identified by 'password';
create database example;
grant replication slave on *.* to 'replicator'@'%';
```
Перезагружаемся и проверяем.

**Тестирование**

Меняем данные на Server-Master:
```
docker exec -it replication-master mysql
mysql> USE world;
mysql> INSERT INTO city (Name, CountryCode, District, Population) VALUES
('Test-Replication', 'ALB', 'Test', 42);
```
Проверяем на slave:
```
docker exec -it replication-slave mysql
mysql> USE world;
mysql> SELECT * FROM city ORDER BY ID DESC LIMIT 1;
```

# 2 часть

**Масштабирование**

**Масштабирование** — это процесс разделения данных на группы и выделение их на отдельные сервера. Основные виды масштабирования:
- репликация (Master-slave, master-master),
- партицирование,
- шардинг.
Кластеризация — это как репликация с каким-либо методом масштабирования

Типы масштабирования:
- scaling up — масштабирование вверх,
- scaling out — масштабирование горизонтальное,
- scaling back — часть данных открыты, часть данных заархивированы,
- federation — доступ к удаленным данным

**Scaling out - Горизонтальное масштабирование**

При масштабировании по горизонтали данные распределяются по нескольким серверам с помощью репликации, а далее используются slave-серверы на чтение. При этом происходит разделение (partition) данных по нескольким нодам. Нода – функциональный блок в MySQL, это может быть отдельный сервер.

Ноды бывают четырех типов:
-  активный master-сервер и пассивный репликационный slave-сервер (настраивали),
- master-сервер и несколько slave-серверов (настраивали),
- активный сервер со специальным механизмом репликации – distributed replicated block device (DRBD),
- SAN-кластер

**Функциональное разделение** — данные разбиваются на таблицы так, что они никогда не соединяются между собой (портирование).
**Data sharding** — механическое разделение огромных объемов однотипных данных на несколько частей (shard — шардинг). С точки зрения реализации, это наиболее трудоемкий вариант

**Шардинг** - (shard — сегментировать)

Шардинг (иногда шардирование) — это техника масштабирования при работе с данными. Его суть в разделении (партиционирование) базы данных на отдельные части так, чтобы каждую из них можно было вынести на отдельный сервер. Существует два вида шардинга:
- вертикальный;
- горизонтальный

**Вертикальный шардинг** — это выделение таблицы или группы таблиц на отдельный сервер. Например, существует две таблицы users и password. Таблицу users оставляем на одном сервере, а таблицу password переводим на другой.

**Горизонтальный шардинг** — это разделение одной таблицы на разные сервера. Это необходимо использовать для огромных таблиц, которые не умещаются на одном сервере. 

**Разделение таблицы на части делается по следующему принципу:**
- на нескольких серверах создается одна и та же таблица (только структура, без данных);
- в приложении выбирается условие, по которому будет определяться нужное соединение (например, четные на один сервер, а нечетные — на другой);
- перед каждым обращением к таблице происходит выбор нужного соединения

**Storage Area Network**

Сеть хранения данных представляет собой архитектурное решение для подключения внешних устройств хранения данных (дисковые массивы, ленточные библиотеки) к серверам таким образом, чтобы операционная система распознала подключённые ресурсы как локальные
