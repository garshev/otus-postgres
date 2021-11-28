project mimetic-math-328316 
instance homework-12-postgres
instance homework-12-clickhouse

Датасет - гугловый - такси NY, копируем все 80 ГБ на отедльный диск. Потом подключаем его к ВМ для импорта данных в таблицы
    gsutil cp -rv gs://taxidata1 /mnt/disks/csvtaxi/

Импортируем через COPY 36 csv файлов на 10 ГБ

set shared_buffers = 11 GB # all table in buffers

    taxi=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
    from taxi_trips
    group by payment_type
    order by 3;
     payment_type | tips_percent |    c
    --------------+--------------+----------
     Prepaid      |            0 |       24
     Way2ride     |           15 |       78
     Pcard        |            3 |     5311
     Dispute      |            0 |    12878
     Prcard       |            1 |    75859
     Mobile       |           16 |    76782
     Unknown      |            1 |    91267
     No Charge    |            3 |   108535
     Credit Card  |           17 |  9989583
     Cash         |            0 | 13694666
    (10 rows)

    Time: 12450.248 ms (00:12.450)

результат - 12 секунд

Создаём колоночную базу clickhouse и импортируем теже данные
for F in taxi_0000000000{00..36}*; do echo "Proccessing $F .."; awk FNR-1 $F | clickhouse-client --query="INSERT INTO taxi.taxi_trips_ram FORMAT CSV"; done
homework-12-clickhouse.us-central1-c.c.mimetic-math-328316.internal :) 
engine = MergeTree()

SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c FROM taxi.taxi_trips  group by payment_type order by 3;

SELECT
    payment_type,
    round((sum(tips) / sum(trip_total)) * 100, 0) + 0 AS tips_percent,
    count(*) AS c
FROM taxi.taxi_trips
GROUP BY payment_type
ORDER BY 3 ASC

┌─payment_type─┬─tips_percent─┬────────c─┐
│ Prcard       │            1 │    86101 │
│ Pcard        │            3 │     5342 │
│ Way2ride     │           15 │       78 │
│ Unknown      │            1 │    98327 │
│ Cash         │            0 │ 14023853 │
│ Prepaid      │            0 │       53 │
│ Dispute      │            0 │    13206 │
│ Mobile       │           16 │    86899 │
│ Credit Card  │           17 │ 10301471 │
│ No Charge    │            3 │   109880 │
└──────────────┴──────────────┴──────────┘

10 rows in set. Elapsed: 0.600 sec. Processed 24.73 million rows, 592.58 MB (41.23 million rows/s., 988.17 MB/s.)
И на премушествах колоночной базы для OLAP запроса получим выиигрыш в 20 раз - 0.6 секунды против 12 в Постгрес.

Создадим таблицу в памяти - Engine = Memory :
    create table taxi.taxi_trips 
    (
        unique_key text, \
        taxi_id text, \
        trip_start_timestamp text, \
        trip_end_text text, \
        trip_seconds bigint, \
        trip_miles FLOAT, \
        pickup_census_tract bigint, \
        dropoff_census_tract bigint, \
        pickup_community_area bigint, \
        dropoff_community_area bigint, \
        fare FLOAT, \
        tips FLOAT, \
        tolls FLOAT, \
        extras FLOAT, \
        trip_total FLOAT, \
        payment_type text, \
        company text, \
        pickup_latitude FLOAT, \
        pickup_longitude FLOAT, \
        pickup_location text, \
        dropoff_latitude FLOAT, \
        dropoff_longitude FLOAT, \
        dropoff_location text \
    ) engine = Memory;

    homework-12-clickhouse.us-central1-c.c.mimetic-math-328316.internal :) SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c FROM taxi.taxi_trips_ram  group by payment_type order by 3;

    SELECT
        payment_type,
        round((sum(tips) / sum(trip_total)) * 100, 0) + 0 AS tips_percent,
        count(*) AS c
    FROM taxi.taxi_trips_ram
    GROUP BY payment_type
    ORDER BY 3 ASC

    ┌─payment_type─┬─tips_percent─┬────────c─┐
    │ Prcard       │            1 │    75859 │
    │ Pcard        │            3 │     5311 │
    │ Way2ride     │           15 │       78 │
    │ Unknown      │            1 │    91267 │
    │ Cash         │            0 │ 13694666 │
    │ Prepaid      │            0 │       24 │
    │ Dispute      │            0 │    12878 │
    │ Mobile       │           16 │    76782 │
    │ Credit Card  │           17 │  9989583 │
    │ No Charge    │            3 │   108535 │
    └──────────────┴──────────────┴──────────┘

    10 rows in set. Elapsed: 0.282 sec. Processed 24.05 million rows, 576.25 MB (85.20 million rows/s., 2.04 GB/s.)

Ускорение в два раза - 0.282 sec. 
