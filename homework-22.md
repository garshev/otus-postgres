-- ДЗ тема: триггеры, поддержка заполнения витрин

-- товары:

Создаём таблицы с продажами, товарами и витриной:

	CREATE TABLE goods
	(
	    goods_id    integer PRIMARY KEY,
	    good_name   varchar(63) NOT NULL,
	    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
	);
	INSERT INTO goods (goods_id, good_name, good_price)
	VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи

	CREATE TABLE sales
	(
	    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	    good_id     integer REFERENCES goods (goods_id),
	    sales_time  timestamp with time zone DEFAULT now(),
	    sales_qty   integer CHECK (sales_qty > 0)
	);


-- отчет:

	SELECT G.good_name, sum(G.good_price * S.sales_qty)
	FROM goods G
	INNER JOIN sales S ON S.good_id = G.goods_id
	GROUP BY G.good_name;

-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу

	CREATE TABLE good_sum_mart
	(
		good_name   varchar(63) NOT NULL,
		sum_sale	numeric(16, 2)NOT NULL
	);
и инициализируем

	INSERT INTO good_sum_mart (good_name, sum_sale) VALUES ('Спички хозайственные',0),('Автомобиль Ferrari FXX K',0);
	
-- Создать триггер (на таблице sales) для поддержки.
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

	CREATE OR REPLACE FUNCTION public.edit_mart()
	 RETURNS trigger
	 LANGUAGE plpgsql
	AS $function$
	BEGIN
		IF    TG_OP = 'INSERT' THEN
		update good_sum_mart SET sum_sale = sum_sale + (SELECT good_price from goods WHERE NEW.good_id = goods.goods_id) * NEW.sales_qty
			WHERE good_name = (SELECT good_name FROM goods WHERE NEW.good_id = goods.goods_id);
		RETURN NEW;

		ELSIF TG_OP = 'DELETE' THEN
		update good_sum_mart SET sum_sale = sum_sale - (SELECT good_price from goods WHERE OLD.good_id = goods.goods_id) * OLD.sales_qty
			WHERE good_name = (SELECT good_name FROM goods WHERE OLD.good_id = goods.goods_id);
		RETURN OLD;

		ELSIF TG_OP = 'UPDATE' THEN
		update good_sum_mart SET sum_sale = sum_sale - (SELECT good_price from goods WHERE NEW.good_id = goods.goods_id) * OLD.sales_qty
			WHERE good_name = (SELECT good_name FROM goods WHERE OLD.good_id = goods.goods_id);
		update good_sum_mart SET sum_sale = sum_sale + (SELECT good_price from goods WHERE NEW.good_id = goods.goods_id) * NEW.sales_qty
			WHERE good_name = (SELECT good_name FROM goods WHERE NEW.good_id = goods.goods_id);
		RETURN NEW;
		END IF;
	END
	$function$

	create trigger tr_edit_mart BEFORE INSERT OR DELETE OR UPDATE ON sales FOR EACH ROW EXECUTE FUNCTION edit_mart ();

Тестируем:

	postgres=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
	INSERT 0 4
	postgres=# select * from good_sum_mart ;
		good_name         |   sum_sale
	--------------------------+--------------
	 Спички хозайственные     |        65.50
	 Автомобиль Ferrari FXX K | 185000000.01
	(2 rows)
Тригер сработал, добавилось 131 коробок спичек и одна машина.

	postgres=# select * from sales;
	 sales_id | good_id |          sales_time           | sales_qty
	----------+---------+-------------------------------+-----------
		5 |       1 | 2022-01-27 12:46:35.276963+00 |        10
		6 |       1 | 2022-01-27 12:46:35.276963+00 |         1
		7 |       1 | 2022-01-27 12:46:35.276963+00 |       120
		8 |       2 | 2022-01-27 12:46:35.276963+00 |         1
	(4 rows)

	postgres=# DELETE FROM sales WHERE sales_id = 5;
	DELETE 1
	postgres=# select * from good_sum_mart ;
		good_name         |   sum_sale
	--------------------------+--------------
	 Автомобиль Ferrari FXX K | 185000000.01
	 Спички хозайственные     |        60.50
	(2 rows)
Удалил 10 коробков.

	postgres=# UPDATE sales SET sales_qty = 2 WHERE sales_time = '2022-01-27 12:46:35.276963+00';
	UPDATE 3
	postgres=# select * from sales;
	 sales_id | good_id |          sales_time           | sales_qty
	----------+---------+-------------------------------+-----------
		6 |       1 | 2022-01-27 12:46:35.276963+00 |         2
		7 |       1 | 2022-01-27 12:46:35.276963+00 |         2
		8 |       2 | 2022-01-27 12:46:35.276963+00 |         2
	(3 rows)

	postgres=# select * from good_sum_mart ;
		good_name         |   sum_sale
	--------------------------+--------------
	 Спички хозайственные     |         2.00
	 Автомобиль Ferrari FXX K | 370000000.02
	(2 rows)
Обновили все продажи за 27е число - стало 4 коробка спичек и 2 машины.


-- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
-- Подсказка: В реальной жизни возможны изменения цен.

По идее в таком случае в витрине по сравнению с отчётом по sales будут реальные суммы продаж. Если задним числом не будем редактировать sales.
