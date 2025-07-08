Отлично, что витрина работает. Внесу исправления по вашим замечаниям:

### 1. Обновление даты окончания на '4444-01-01'
```sql
-- Обновление дефолтного значения для будущих записей
ALTER TABLE alm_history.interest_rates
DROP CONSTRAINT [DF__interest___dt_to__xxxx]; -- заменить xxxx на реальное имя констрейнта

ALTER TABLE alm_history.interest_rates
ADD DEFAULT '4444-01-01' FOR dt_to;
GO

-- Обновление существующих записей
UPDATE alm_history.interest_rates
SET dt_to = '4444-01-01'
WHERE dt_to = '9999-12-31';
GO
```

### 2. Корректировка формата данных (проценты → десятичные дроби)
```sql
-- Обновление существующих данных
UPDATE alm_history.interest_rates
SET value = value / 100.0;
GO

-- Обновление дефолтного представления для проверки
ALTER VIEW alm_report.vw_daily_rates AS
WITH DateRange AS (
    SELECT 
        DATEADD(DAY, seq.n, (SELECT MIN(dt_from) FROM alm_history.interest_rates)) AS report_date
    FROM (
        SELECT TOP (DATEDIFF(DAY, (SELECT MIN(dt_from) FROM alm_history.interest_rates), GETDATE()) + 1)
            ROW_NUMBER() OVER (ORDER BY object_id) - 1 AS n
        FROM sys.all_objects
    ) seq
)
SELECT
    d.report_date,
    r.term,
    r.cur,
    r.conv,
    MAX(CASE WHEN r.rate_type = 'nadbavka' THEN r.value END) AS nadbavka,
    MAX(CASE WHEN r.rate_type = 'rate_trf_controlling' THEN r.value END) AS rate_trf_controlling,
    MAX(CASE WHEN r.rate_type = 'rate_trf_with_nadbavka' THEN r.value END) AS rate_trf_with_nadbavka
FROM DateRange d
JOIN alm_history.interest_rates r 
    ON d.report_date BETWEEN r.dt_from AND r.dt_to
GROUP BY d.report_date, r.term, r.cur, r.conv;
GO
```

### 3. Проверка данных после изменений
```sql
-- Проверка данных за 18.02.2025 (должны быть в десятичном формате)
SELECT 
    report_date,
    term,
    conv,
    nadbavka,
    rate_trf_controlling,
    rate_trf_with_nadbavka
FROM alm_report.vw_daily_rates
WHERE report_date = '2025-02-18'
  AND term = 91
  AND conv = 'AT_THE_END';
```

**Ожидаемый результат:**
| report_date | term | conv        | nadbavka | rate_trf_controlling | rate_trf_with_nadbavka |
|-------------|------|-------------|----------|-----------------------|------------------------|
| 2025-02-18  | 91   | AT_THE_END | 0.0116   | 0.2189               | 0.2305                |

### 4. Адаптация под импорт из Excel
Для корректной работы с процентными значениями из Excel:
1. Макрос будет преобразовывать проценты в десятичные дроби
2. Примеры значений в таблице уже приведены к корректному формату

**Пример обработки в VBA:**
```vba
' Преобразование процентной строки в десятичное число
Function ParsePercentage(value As String) As Double
    Dim cleanValue As String
    cleanValue = Replace(value, "%", "")       ' Удаляем символ %
    cleanValue = Replace(cleanValue, ",", ".") ' Заменяем запятую на точку
    ParsePercentage = CDbl(cleanValue) / 100   ' Конвертируем в число и делим на 100
End Function

' Пример использования в коде вставки
value = ParsePercentage(ws.Cells(i, colIdx + 1).Value)
```

### Финализированный код создания таблицы
```sql
USE ALM_TEST;
GO

-- Создание схемы
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'alm_history')
    EXEC('CREATE SCHEMA alm_history');
GO

-- Создание таблицы с корректными настройками
CREATE TABLE alm_history.interest_rates (
    id INT IDENTITY(1,1) PRIMARY KEY,
    dt_from DATE NOT NULL,
    dt_to DATE NOT NULL DEFAULT '4444-01-01',
    term INT NOT NULL,
    cur CHAR(3) NOT NULL DEFAULT '810',
    conv VARCHAR(20) NOT NULL,
    rate_type VARCHAR(50) NOT NULL,
    value DECIMAL(18,6) NOT NULL, -- Увеличили точность для процентных ставок
    load_dt DATETIME DEFAULT GETDATE()
);
GO

-- Индексы
CREATE INDEX idx_dt_range ON alm_history.interest_rates (dt_from, dt_to);
CREATE INDEX idx_term_conv ON alm_history.interest_rates (term, conv, rate_type);
GO
```

### Преимущества решения:
1. **Корректное хранение процентных ставок**:
   - Все значения хранятся в десятичном формате (0.23 вместо 23%)
   - Увеличена точность до 6 знаков после запятой
   - Автоматическое преобразование при импорте

2. **Гибкая система периодов**:
   - Используется '4444-01-01' как дата по умолчанию
   - Периоды не пересекаются
   - Легкое управление историей

3. **Оптимизированная витрина**:
   - Быстрый доступ к данным за любой день
   - Автоматическая генерация календаря дат
   - Корректная обработка переходов между периодами

4. **Подготовка к импорту**:
   - Структура таблицы соответствует формату Excel
   - Процентные значения автоматически конвертируются
   - Поддерживаются разные форматы чисел (с запятой/точкой)

Для полной готовности к импорту осталось:
1. Реализовать VBA-макрос с функцией `ParsePercentage`
2. Настроить соединение с SQL Server
3. Добавить обработку ошибок и логирование

Готов приступить к разработке макроса импорта, как только вы подтвердите корректность обновленной структуры и витрины.
