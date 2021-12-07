Секционировать большую таблицу из демо базы flights

    -- Будем секционировать таблицу с данными - ticket_flights
    
    -- Создадим новую таблицу LIKE старую с декларативным секциорнирование по хешу:
    create table ticket_flights_new (like ticket_flights including all) partition by hash(ticket_no); 
    
    -- Добавим внешние ограничения, чтобы было как в исходной таблице:
    ALTER TABLE bookings.ticket_flights_new
        ADD CONSTRAINT ticket_flights_flight_id_fkey FOREIGN KEY (flight_id) REFERENCES bookings.flights(flight_id);
    ALTER TABLE bookings.ticket_flights_new
        ADD CONSTRAINT ticket_flights_ticket_no_fkey FOREIGN KEY (ticket_no) REFERENCES bookings.tickets(ticket_no);
    
    -- Создадим 3 секции:
    create table ticket_flights_0 partition of ticket_flights_new for values with (modulus 3, remainder 0);
    create table ticket_flights_1 partition of ticket_flights_new for values with (modulus 3, remainder 1);
    create table ticket_flights_2 partition of ticket_flights_new for values with (modulus 3, remainder 2);

    -- Скопируем данные в новую таблицу
    insert into ticket_flights_new select * from ticket_flights;
    
    -- Удалим старую таблицу
    drop table ticket_flights cascade;
    
    -- Переименуем
    alter table ticket_flights_new rename TO ticket_flights;

    -- Посмотрим как распределились строки
     select tableoid::regclass as table_parts, count(*) from ticket_flights group by 1;
       table_parts    |  count
    ------------------+---------
     ticket_flights_0 | 2796313
     ticket_flights_1 | 2798707
     ticket_flights_2 | 2796832
    (3 rows)

    \d+ ticket_flights;
                                              Table "bookings.ticket_flights"
         Column      |         Type          | Collation | Nullable | Default | Storage  | Stats target |  Description
    -----------------+-----------------------+-----------+----------+---------+----------+--------------+---------------
     ticket_no       | character(13)         |           | not null |         | extended |              | Ticket number
     flight_id       | integer               |           | not null |         | plain    |              | Flight ID
     fare_conditions | character varying(10) |           | not null |         | extended |              | Travel class
     amount          | numeric(10,2)         |           | not null |         | main     |              | Travel cost
    Partition key: HASH (ticket_no)
    Indexes:
        "ticket_flights_new_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    Check constraints:
        "ticket_flights_amount_check" CHECK (amount >= 0::numeric)
        "ticket_flights_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
    Foreign-key constraints:
        "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
        "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
    Partitions: ticket_flights_0 FOR VALUES WITH (modulus 3, remainder 0),
                ticket_flights_1 FOR VALUES WITH (modulus 3, remainder 1),
                ticket_flights_2 FOR VALUES WITH (modulus 3, remainder 2)

