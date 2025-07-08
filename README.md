Ошибка возникает из-за неправильного форматирования десятичных чисел в SQL-запросе. В вашем регионе используется запятая как десятичный разделитель, но SQL требует точку. Вот исправления:

**1. Обновите функцию ParsePercentage:**
```vba
Function ParsePercentage(percentText As String) As Double
    On Error GoTo ErrorHandler
    percentText = Replace(Trim(percentText), "%", "")
    percentText = Replace(percentText, " ", "") ' Удаляем пробелы
    percentText = Replace(percentText, ",", ".") ' Заменяем запятую на точку
    ParsePercentage = Val(percentText) / 100
    Exit Function
ErrorHandler:
    ParsePercentage = 0
End Function
```

**2. Исправьте форматирование числа в SQL-запросе:**
Замените строку с формированием значения ставки:
```vba
Format(CDbl(rateValue), "#0.000000")
```
на:
```vba
Replace(Format(CDbl(rateValue), "0.000000"), ",", ".")
```

**Итоговый исправленный блок кода:**
```vba
' ... предыдущий код ...
For row = 6 To 10
    If Not IsEmpty(ws.Cells(row, col)) And ws.Cells(row, col).Value <> "" Then
        rateTypeIndex = row - 6
        rateValue = ParsePercentage(ws.Cells(row, col).Text)
        Debug.Print "Обрабатываем ставку: " & rateValue
        
        Call CheckAndUpdateOldRecords(conn, startDate, termDay, currencyCode, convTypes(rateTypeIndex), rateTypes(rateTypeIndex))
        
        ' Исправленное форматирование числа:
        sql = "INSERT INTO alm_history.interest_rates (" & _
                "dt_from, term, cur, conv, rate_type, value, dt_to, load_dt)" & _
                " VALUES ('" & startDate & "', " & termDay & ", '" & _
                Replace(currencyCode, "'", "''") & "', '" & _
                convTypes(rateTypeIndex) & "', '" & _
                rateTypes(rateTypeIndex) & "', " & _
                Replace(Format(CDbl(rateValue), "0.000000"), ",", ".") & ", '4444-01-01', GETDATE());"
        
        ' ... остальной код ...
```

**Пояснение:**
1. **ParsePercentage:** Теперь корректно обрабатывает числа с запятыми, заменяя их на точки перед преобразованием в число
2. **Форматирование для SQL:** Дополнительная замена запятой на точку в отформатированном числе гарантирует правильный синтаксис SQL
3. **Формат "0.000000":** Всегда выводит ведущий ноль (0.011600 вместо .011600), что требуется для числовых литералов в SQL

После этих изменений запросы будут корректно выполняться, так как числа будут передаваться в SQL с точкой в качестве десятичного разделителя (0.011600 вместо 0,011600).
