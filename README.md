-- неделя: СРЕДА (вкл.) → ВТОРНИК (вкл.)
SET DATEFIRST 1;  -- чтобы не зависеть от регионалки

SELECT
  CONCAT(
    FORMAT(DATEADD(day, 1 - DATEPART(WEEKDAY, DATEADD(day, -2, dt_rep)), dt_rep), 'yyyy.MM.dd'),
    ' : ',
    FORMAT(DATEADD(day, 7 - DATEPART(WEEKDAY, DATEADD(day, -2, dt_rep)), dt_rep), 'yyyy.MM.dd')
  ) AS rep_W
FROM your_table;   -- замените на свой источник
