# 03 Развертывание и базовая конфигурация, интерфейсы и инструменты

## Описание/Пошаговая инструкция выполнения домашнего задания:
```
1. Установить ClickHouse.
2. Подгрузить датасет для примера и сделать селект из таблицы.
3. Для проверки отправить скрины работающего инстанса ClickHouse, созданной
виртуальной машины и результата запроса select.
4. Провести тестирование производительности и сохранить результаты;
a. echo "SELECT * FROM default.cell_towers LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark -i 10
5. Изучить конфигурационные файлы БД;
6. Произвести наиболее оптимальную настройку системы на основании характеристик
вашей ОС и провести повторное тестирование;
7. Подготовить отчет касательно прироста/изменения производительности системы на
основе проведенных настроек.
```

1. docker compose up -d 
2. CREATE DATABASE trips_db;
```
CREATE TABLE trips_db.trips (
    trip_id             UInt32,
    pickup_datetime     DateTime,
    dropoff_datetime    DateTime,
    pickup_longitude    Nullable(Float64),
    pickup_latitude     Nullable(Float64),
    dropoff_longitude   Nullable(Float64),
    dropoff_latitude    Nullable(Float64),
    passenger_count     UInt8,
    trip_distance       Float32,
    fare_amount         Float32,
    extra               Float32,
    tip_amount          Float32,
    tolls_amount        Float32,
    total_amount        Float32,
    payment_type        Enum('CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4, 'UNK' = 5),
    pickup_ntaname      LowCardinality(String),
    dropoff_ntaname     LowCardinality(String)
)
ENGINE = MergeTree
PRIMARY KEY (pickup_datetime, dropoff_datetime);

INSERT INTO trips_db.trips
SELECT
    trip_id,
    pickup_datetime,
    dropoff_datetime,
    pickup_longitude,
    pickup_latitude,
    dropoff_longitude,
    dropoff_latitude,
    passenger_count,
    trip_distance,
    fare_amount,
    extra,
    tip_amount,
    tolls_amount,
    total_amount,
    payment_type,
    pickup_ntaname,
    dropoff_ntaname
FROM s3(
    'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{0..2}.gz',
    'TabSeparatedWithNames'
);

Ok.

0 rows in set. Elapsed: 75.186 sec. Processed 3.00 million rows, 244.69 MB (39.91 thousand rows/s., 3.25 MB/s.)
Peak memory usage: 293.59 MiB.
```
3.
```
pool80@pool80-N7400PC:~/ch-docker$ docker compose ps
NAME      IMAGE                                           COMMAND            SERVICE     CREATED      STATUS          PORTS
ch        clickhouse/clickhouse-server:23.8.14.6-alpine   "/entrypoint.sh"   ch_server   4 days ago   Up 12 minutes   127.0.0.1:8123->8123/tcp, 127.0.0.1:9000->9000/tcp, 9009/tcp
```
и
```
ch01 :) select count() from trips where payment_type = 1

SELECT count()
FROM trips
WHERE payment_type = 1

Query id: 242d3f7e-7544-4fb0-85f3-cf463393c1f3


0 rows in set. Elapsed: 0.002 sec. 

Received exception from server (version 23.8.14):
Code: 60. DB::Exception: Received from localhost:9000. DB::Exception: Table default.trips does not exist. (UNKNOWN_TABLE)
```
4. 
```
ch01:/# echo "SELECT * FROM trips_db.trips LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark -i 10 --user default --password qwerty123
Loaded 1 queries.

Queries executed: 10.

localhost:9000, queries: 10, QPS: 15.240, RPS: 45723640.201, MiB/s: 3168.239, result RPS: 0.000, result MiB/s: 0.000.

0.000%		0.050 sec.	
10.000%		0.050 sec.	
20.000%		0.051 sec.	
30.000%		0.053 sec.	
40.000%		0.053 sec.	
50.000%		0.053 sec.	
60.000%		0.053 sec.	
70.000%		0.053 sec.	
80.000%		0.053 sec.	
90.000%		0.053 sec.	
95.000%		0.074 sec.	
99.000%		0.074 sec.	
99.900%		0.074 sec.	
99.990%		0.074 sec.
```
5. Правки размещаем
файлы конфигов размещены в именованном томе /var/lib/docker/volumes/ch-docker_db-etc/ . Правки делаем в каталогах   config.d и users.d

6. 

- По умолчанию КХ можетобрабатывать 4096 соединений(eep_alive_timeout setting), но одновремнно может выполняться только 100 (max_concurrent_queries) запросов, поэтому остальные ждут в очереди. Максимальная длительность, в течение которой клиентские запросы могут оставаться в очереди, определяется установкой параметра queue_max_wait_ms (по умолчанию 5000 или 5 секунд). Это настройка пользователя/профиля, поэтому пользователи могут определить меньшее значение, чтобы вызвать исключение в случаях, когда очередь слишком длинная.  Тайм-аут Keepalive для http-соединения по умолчанию — 3 секунды (keep_alive_timeout).
- По умолчанию <level>trace</level>  и для каждого запроса он записывает несколько строк в файл журнала, что удобно для отладки, но, создает некоторые дополнительные задержки.
- max_memory_usage - можно ограничить количество памяти выделяемое для одного запроса
- use_uncompressed_cache - включите эту настройку для пользователей, от которых идут частые короткие запросы. 
Также обратите внимание на конфигурационный параметр uncompressed_cache_size (настраивается только в конфигурационном файле) – размер кэша разжатых блоков. По умолчанию - 8 GiB. Кэш разжатых блоков заполняется по мере надобности, а наиболее невостребованные данные автоматически удаляются. Для запросов, читающих хоть немного приличный объём данных (миллион строк и больше), кэш разжатых блоков автоматически выключается, чтобы оставить место для действительно мелких запросов. Поэтому, можно держать настройку use_uncompressed_cache всегда выставленной в 1.

7.  Много памяти, мощные процы, и nvme диски в рейдах лучшие улучшение для КХ)
На самом деле непонятно каким образом оптимизировать КХ от системы, что бы в тестовом запросе на тестовом датасете увидеть реальную прибавку) Я получаю примерно теже результаты с небольшой погрешностью
```
ch01:/# echo "SELECT * FROM trips_db.trips LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark -i 10 --user default --password qwerty123
Loaded 1 queries.

Queries executed: 10.

localhost:9000, queries: 10, QPS: 15.304, RPS: 45916394.923, MiB/s: 3181.596, result RPS: 0.000, result MiB/s: 0.000.

0.000%		0.051 sec.	
10.000%		0.052 sec.	
20.000%		0.052 sec.	
30.000%		0.052 sec.	
40.000%		0.052 sec.	
50.000%		0.052 sec.	
60.000%		0.052 sec.	
70.000%		0.053 sec.	
80.000%		0.053 sec.	
90.000%		0.053 sec.	
95.000%		0.069 sec.	
99.000%		0.069 sec.	
99.900%		0.069 sec.	
99.990%		0.069 sec.
```