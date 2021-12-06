1 вариант:  
Создать индексы на БД, которые ускорят доступ к данным.  
В данном задании тренируются навыки:  

    определения узких мест
    написания запросов для создания индекса
    оптимизации Необходимо:

    Создать индекс к какой-либо из таблиц вашей БД
    Прислать текстом результат команды explain, в которой используется данный индекс
      
      create table t(a integer, b text, c boolean);
      
      insert into t(a,b,c)
      select s.id, chr((32+random()*94)::integer), random() < 0.01
      from generate_series(1,100000) as s(id)
      order by random();
      
      create index on t(a);
      
      explain analyze select * from t where a = 1;
                                                QUERY PLAN                                                 
      -----------------------------------------------------------------------------------------------------------
       Index Scan using t_a_idx on t  (cost=0.29..8.31 rows=1 width=7) (actual time=0.058..0.059 rows=1 loops=1)
         Index Cond: (a = 1)
       Planning Time: 0.064 ms
       Execution Time: 0.075 ms
      (4 rows)

        
    Реализовать индекс для полнотекстового поиска
    
        create table ts(doc text, doc_tsv tsvector);
        copy ts(doc) from '/home/slava/puh.txt'; -- Бесы Пушкина
        set default_text_search_config = russian;
        update ts set doc_tsv = to_tsvector(doc); -- Создаём лексемы
        create index on ts using gin(doc_tsv);
        select doc from ts where doc_tsv @@ to_tsquery('туч');
                        doc            
            ---------------------------
             Мчатся тучи, вьются тучи;
             Мчатся тучи, вьются тучи,
             Мчатся тучи, вьются тучи;
            (3 rows)
            
         explain (costs off)  select doc from ts where doc_tsv @@ to_tsquery('туч');
                        QUERY PLAN                        
            ----------------------------------------------------------
             Bitmap Heap Scan on ts
               Recheck Cond: (doc_tsv @@ to_tsquery('туч'::text))
               ->  Bitmap Index Scan on ts_doc_tsv_idx
                     Index Cond: (doc_tsv @@ to_tsquery('туч'::text))
            (4 rows)

    
    Реализовать индекс на часть таблицы или индекс на поле с функцией
    
        create index on t(lower(b));
        
        explain (costs off) select * from t where lower(b) = 'a';
                 QUERY PLAN                 
        --------------------------------------------
         Bitmap Heap Scan on t
           Recheck Cond: (lower(b) = 'a'::text)
           ->  Bitmap Index Scan on t_lower_idx
                 Index Cond: (lower(b) = 'a'::text)
        (4 rows)

    
    Создать индекс на несколько полей
    
        create index on t(a,b);
    
    Написать комментарии к каждому из индексов
    
        \di+
                                             List of relations
         Schema |    Name     | Type  |  Owner   | Table |  Size   |          Description           
        --------+-------------+-------+----------+-------+---------+--------------------------------
         public | t_a_b_idx   | index | postgres | t     | 2208 kB | Индекс по столбцам a и b
         public | t_a_idx     | index | postgres | t     | 2208 kB | Индекс по числовому столбцу a
         public | t_b_idx     | index | postgres | t     | 2208 kB | Индекс по текстовому столбцу b
         public | t_lower_idx | index | postgres | t     | 2208 kB | Индекс по функции на столбец b
        (4 rows)

    
    Описать что и как делали и с какими проблемами столкнулись
    
2 вариант:  
В результате выполнения ДЗ вы научитесь пользоваться различными вариантами соединения таблиц.  
В данном задании тренируются навыки:

    написания запросов с различными типами соединений  
    Необходимо:

    Реализовать прямое соединение двух или более таблиц
    
    
        create table a(id serial, fname text);
        create table b(id serial, lname text);
        create table kinders(b_id int, fname text);
        insert into a(fname) values ('Ivan'), ('Valera'), ('Nina'), ('Lera');
        insert into b(lname) values ('Petrov'), ('Ivanov'), ('Popova');
        insert into kinders(b_id, fname) values (1, 'Peter'), (2, 'Nikolay'), (3, 'Sveta'), (3, 'Dasha');
        
        select * from a join b on a.id = b.id;   -- выводит совпадающие по ключам строки обоих таблиц
         id | fname  | id | lname
        ----+--------+----+--------
          1 | Ivan   |  1 | Petrov
          2 | Valera |  2 | Ivanov
          3 | Nina   |  3 | Popova
        (3 rows)
        
    Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
    
         select * from a left join b on a.id = b.id;  -- выводит все строки левой таблицы и все и совпадающие по ключам строки правой таблицы
         id | fname  | id | lname
        ----+--------+----+--------
          1 | Ivan   |  1 | Petrov
          2 | Valera |  2 | Ivanov
          3 | Nina   |  3 | Popova
          4 | Lera   |    |
        (4 rows)
    
    Реализовать кросс соединение двух или более таблиц
    
        select * from a cross join b;       -- перемножение строк
         id | fname  | id | lname
        ----+--------+----+--------
          1 | Ivan   |  1 | Petrov
          1 | Ivan   |  2 | Ivanov
          1 | Ivan   |  3 | Popova
          2 | Valera |  1 | Petrov
          2 | Valera |  2 | Ivanov
          2 | Valera |  3 | Popova
          3 | Nina   |  1 | Petrov
          3 | Nina   |  2 | Ivanov
          3 | Nina   |  3 | Popova
          4 | Lera   |  1 | Petrov
          4 | Lera   |  2 | Ivanov
          4 | Lera   |  3 | Popova
        (12 rows)
    
    Реализовать полное соединение двух или более таблиц
    
        select * from a full join b on a.id = b.id;  -- Строки обеих таблиц в результате
         id | fname  | id | lname
        ----+--------+----+--------
          1 | Ivan   |  1 | Petrov
          2 | Valera |  2 | Ivanov
          3 | Nina   |  3 | Popova
          4 | Lera   |    |
        (4 rows)
    
    Реализовать запрос, в котором будут использованы разные типы соединений
    
        select * from a full join b on a.id = b.id left join kinders on b.id = kinders.b_id;  -- К полному соединение таблиц a и b добавляем строки таблицы kinders
         id | fname  | id | lname  | b_id |  fname
        ----+--------+----+--------+------+---------
          1 | Ivan   |  1 | Petrov |    1 | Peter
          2 | Valera |  2 | Ivanov |    2 | Nikolay
          3 | Nina   |  3 | Popova |    3 | Sveta
          3 | Nina   |  3 | Popova |    3 | Dasha
          4 | Lera   |    |        |      |
        (5 rows)
    
    Сделать комментарии на каждый запрос
    К работе приложить структуру таблиц, для которых выполнялись соединения

    Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке

