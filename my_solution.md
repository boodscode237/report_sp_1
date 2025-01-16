## Report

## Хранимая процедура: `sp_report_1`

```sql
-- =============================================
-- Автор:         Дональд
-- Дата создания: 14.01.2025
-- Описание:      Генерирует отчет о продажах на основе указанного диапазона дат и групп товаров.
-- =============================================
CREATE PROCEDURE sp_report_1 
    @date_from DATE,                       -- Начальная дата для отчета о продажах
    @date_to DATE,                         -- Конечная дата для отчета о продажах
    @good_group_name NVARCHAR(MAX)         -- Список имен групп товаров, разделенных запятыми
AS
BEGIN
    -- Отключает отправку сообщений DONE_IN_PROC клиенту для оптимизации производительности
    SET NOCOUNT ON; 

    -- Объявление табличной переменной для хранения отдельных имен групп товаров
    DECLARE @good_groups TABLE (group_name NVARCHAR(MAX));

    -- Разделение строки имен групп товаров и вставка их в таблицу @good_groups
    INSERT INTO @good_groups (group_name)
    SELECT LTRIM(RTRIM(value))               -- Удаление пробелов в начале и конце
    FROM SplitString(@good_group_name, ','); -- Использование функции SplitString для парсинга входной строки

    -- Выборка и агрегация данных о продажах на основе предоставленных параметров
    SELECT 
        d.d AS sale_date,                                  -- Дата продажи
        st.store_name,                                     -- Название магазина, где произошла продажа
        g.group_name,                                      -- Название группы товаров
        SUM(f.sale_grs) AS total_sales_with_vat,           -- Общие продажи с учетом НДС
        SUM(f.sale_net) AS total_sales,                    -- Общие продажи без учета НДС
        SUM(f.cost_net * f.quantity) AS total_purchase,    -- Общие закупочные расходы (цена за единицу * количество)
        SUM(f.sale_net) - SUM(f.cost_net * f.quantity) AS margin_net, -- Чистая маржа (продажи - закупки)
        CASE 
            WHEN SUM(f.cost_net * f.quantity) = 0 THEN NULL
            ELSE ((SUM(f.sale_net) - SUM(f.cost_net * f.quantity)) / 
                   SUM(f.cost_net * f.quantity)) * 100 
        END AS markup_net,                                 -- Процент наценки на чистую маржу
        (SUM(f.sale_grs) / NULLIF(SUM(f.sale_net), 0)) * 100 AS vat_share, -- Доля НДС в общих продажах
        CASE 
            WHEN SUM(f.quantity) = 0 THEN NULL
            ELSE SUM(f.cost_net * f.quantity) / NULLIF(SUM(f.quantity), 0)
        END AS avg_purchase_price                         -- Средняя закупочная цена за единицу
    FROM dbo.fct_cheque AS f
    INNER JOIN dim_date AS d ON f.date_id = d.did            -- Соединение с таблицей измерений дат
    INNER JOIN dim_stores AS st ON f.store_id = st.store_id  -- Соединение с таблицей измерений магазинов
    INNER JOIN dim_goods AS g ON f.good_id = g.good_id      -- Соединение с таблицей измерений товаров
    WHERE 
        g.group_name IN (SELECT group_name FROM @good_groups) -- Фильтрация по указанным группам товаров
        AND d.d BETWEEN @date_from AND @date_to             -- Фильтрация по диапазону дат
    GROUP BY 
        d.d, 
        st.store_name, 
        g.group_name                                       -- Группировка по дате продажи, названию магазина и группе товаров
    ORDER BY 
        vat_share DESC;                                     -- Сортировка результатов по доле НДС в порядке убывания

END;
GO
```


## Вспомогательная функция: `SplitString`

```sql
-- =============================================
-- Автор:         Дональд
-- Дата создания: 14.01.2025
-- Описание:      Разбивает строку с разделителями на таблицу значений.
-- =============================================
CREATE FUNCTION dbo.SplitString 
(
    @str NVARCHAR(MAX),        -- Входная строка для разделения
    @delimiter CHAR(1)         -- Разделительный символ (например, запятая)
)
RETURNS @output TABLE (value NVARCHAR(MAX)) -- Возвращает таблицу с одним столбцом 'value'
AS
BEGIN
    DECLARE @start INT, @end INT
    SET @start = 1
    -- Поиск позиции первого разделителя
    SET @end = CHARINDEX(@delimiter, @str)

    -- Цикл для перебора строки и извлечения подстрок на основе разделителя
    WHILE @end > 0
    BEGIN
        -- Вставка подстроки в выходную таблицу после удаления пробелов
        INSERT INTO @output (value) 
        VALUES (SUBSTRING(@str, @start, @end - @start))

        -- Обновление начальной позиции на символ после текущего разделителя
        SET @start = @end + 1

        -- Поиск позиции следующего разделителя
        SET @end = CHARINDEX(@delimiter, @str, @start)
    END

    -- Вставка оставшейся подстроки после последнего разделителя
    INSERT INTO @output (value) 
    VALUES (SUBSTRING(@str, @start, LEN(@str) - @start + 1))

    RETURN
END;
GO
```
