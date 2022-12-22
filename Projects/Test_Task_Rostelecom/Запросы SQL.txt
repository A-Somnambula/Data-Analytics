
----------1----------
-- В таблице sales есть отсутствующие значения в столбце [Номер Заказа] --
-- Посчитаем процент отсутствующих значений для продаж --
-- Получился 1.01% - это низкий показатель --
-- В рамках данного анализа эти данные можно отсеять --

SELECT
	(SELECT COUNT(*)::float AS NUMERATOR
		FROM SALES
		WHERE [Номер Заказа] IS NULL
		AND 
	 	[Признак] like 'Подключение') 
		/
	(SELECT COUNT(*)::float AS DIVIDER
		FROM SALES
		WHERE [Номер Заказа] IS NOT NULL
		AND 
	 	[Признак] like 'Подключение') * 100 AS NULLS_PERCENTAGE;



----------2----------
-- Запрос для выгрузки динамики продаж --

SELECT 
	DATE, 
	COUNT(DISTINCT [Номер Заказа])
FROM SALES
WHERE [Признак] = 'Подключение'
AND 
[Номер Заказа] IS NOT NULL
GROUP BY DATE
ORDER BY DATE ASC;


----------3----------
-- Динамика продаж по месяцам в разрезе услуг --

SELECT 
	DATE_TRUNC('month', Date),
	[Услуга],
	COUNT([Номер Заказа])
FROM SALES
WHERE [Признак] = 'Подключение'
AND [Номер Заказа] IS NOT NULL
GROUP BY 1, 2
ORDER BY 1 ASC;


----------4----------
-- Изменения показателя продаж относительно прошлого года по месяцам (диаграмма водопад) -- 

WITH SALES2021 AS
	(SELECT 
			EXTRACT(MONTH FROM Date) AS MONTH_NUM,
			COUNT(DISTINCT [Номер Заказа]) AS SALES
		FROM SALES
		WHERE [Признак] = 'Подключение'
		AND EXTRACT(YEAR FROM date) = 2021
		GROUP BY 1
		ORDER BY 1 ASC),
		
	SALES2022 AS
	(SELECT 
			EXTRACT(MONTH FROM Date) AS MONTH_NUM,
			COUNT(DISTINCT [Номер Заказа]) AS SALES
		FROM SALES
		WHERE [Признак] = 'Подключение'
			AND EXTRACT(YEAR FROM date) = 2022
		GROUP BY 1
		ORDER BY 1 ASC)
		
SELECT 
	S2.MONTH_NUM,
	S1.SALES AS "2021_sales",
	S2.SALES AS "2022_sales",
	S2.SALES - S1.SALES AS DIFF
FROM SALES2021 S1
INNER JOIN SALES2022 S2 USING(MONTH_NUM);


----------5----------
-- Список адресов с количеством заявок, подключений и конверсией -- 

WITH DISTINCT_ADDRESSES AS
(
SELECT DISTINCT 
	LOCAL_ADDR_ID,
	GLOBAL_ADDR_ID
FROM ID
WHERE GLOBAL_ADDR_ID IS NOT NULL
),

APPLICATIONS_COUNT AS
(
SELECT 
	ADDRESS,
	COUNT(DISTINCT [Номер Заказа]) APP_CNT
FROM SALES
LEFT JOIN DISTINCT_ADDRESSES DA ON SALES.HOUSE_ID = DA.LOCAL_ADDR_ID
LEFT JOIN ADDRESS ON DA.GLOBAL_ADDR_ID = ADDRESS.GLOBAL_ID
WHERE [Признак] = 'Заявка'
AND [Номер Заказа] IS NOT NULL
AND ADDRESS IS NOT NULL
GROUP BY 1
),

SALES_COUNT AS
(
SELECT 
	ADDRESS,
	COUNT(DISTINCT [Номер Заказа]) S_CNT
FROM SALES
LEFT JOIN DISTINCT_ADDRESSES DA ON SALES.HOUSE_ID = DA.LOCAL_ADDR_ID
LEFT JOIN ADDRESS ON DA.GLOBAL_ADDR_ID = ADDRESS.GLOBAL_ID
WHERE [Признак] = 'Подключение'
AND [Номер Заказа] IS NOT NULL
AND ADDRESS IS NOT NULL
GROUP BY 1
)

SELECT DISTINCT 
	SC.ADDRESS,
	APP_CNT,
	S_CNT,
	(S_CNT::float / APP_CNT::float) * 100 CONVERSION
FROM APPLICATIONS_COUNT AC
INNER JOIN SALES_COUNT SC USING(ADDRESS)
ORDER BY 3 DESC, 1

