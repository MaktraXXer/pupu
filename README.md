Option Explicit

Public Sub RunBothForecastMacros()

    On Error GoTo ErrHandler

    Application.StatusBar = "Запуск первого расчета..."

    Call CalculateForecastRows

    Application.StatusBar = "Запуск второго расчета..."

    Call CalculateForecastRows_Second

    Application.StatusBar = False

    MsgBox _
        "Оба расчета выполнены.", _
        vbInformation

    Exit Sub

ErrHandler:

    Application.StatusBar = False

    MsgBox _
        "Выполнение остановлено." & vbCrLf & _
        "Ошибка: " & Err.Description, _
        vbCritical

End Sub
