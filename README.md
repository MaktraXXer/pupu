/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654)
   СНАПШОТ @Anchor = как в mail.usp_fill_balance_metrics_savings
   (ULTRA +1/+2; LIQ vs balance; те же фильтры) + ZeroContracts.

   Ключевые отличия:
     • Помесячные промо-спреды: Promo(month,segment) − KEY(1-е числа)
     • Полный перелив Σ клиента на base-day и 1-го числа (по max ставке)
     • На @Anchor (опционально) ставка подменяется «фактом» из DCR

   v.2025-09-07
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
SET NOCOUNT ON;

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-09-03',
    @HorizonTo  date         = '2025-12-31',
    @BaseRate   decimal(9,4) = 0.0450;   -- базовая ставка на base-day

/* Как далеко моделируем промо-переоткрытия по месяцам */
DECLARE
    @MonthsAhead int = DATEDIFF(month, EOMONTH(@Anchor), @HorizonTo) + 1;

/* Справочник входных промо-ставок (можно задать по месяцам) */
DECLARE @PromoRates TABLE(
    month1        date         NOT NULL,
    TSEGMENTNAME  nvarchar(40) NOT NULL,
    promo_rate    decimal(9,4) NOT NULL,
    CONSTRAINT PK_PromoRates PRIMARY KEY (month1, TSEGMENTNAME)
);

/* заполняем одинаковыми значениями на весь горизонт;
   если помесячные ставки отличаются — просто добавь нужные строки */
;WITH months AS (
    SELECT DATEFROMPARTS(YEAR(@Anchor), MONTH(@Anchor), 1) AS m1
    UNION ALL
    SELECT DATEADD(month,1,m1) FROM months
    WHERE m1 < DATEFROMPARTS(YEAR(@HorizonTo), MONTH(@HorizonTo), 1)
)
INSERT @PromoRates(month1,TSEGMENTNAME,promo_rate)
SELECT m.m1, v.seg,
       CASE v.seg WHEN N'ДЧБО' THEN 0.1660 ELSE 0.1640 END
FROM months m
CROSS APPLY (VALUES (N'ДЧБО'),(N'Розничный бизнес')) v(seg)
OPTION (MAXRECURSION 0);

/* ── 0) Календарь горизонта + KEY(TERM=1) ───────────────────── */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d = @Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario = @Scenario AND TERM = 1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP = @Anchor;

/* ── 0b) Спреды по месяцам (Promo − KEY(1-е числа месяцев)) ── */
IF OBJECT_ID('tempdb..#spreads') IS NOT NULL DROP TABLE #spreads;
SELECT pr.month1,
       pr.TSEGMENTNAME,
       spread = pr.promo_rate - k.KEY_RATE
INTO   #spreads
FROM   @PromoRates pr
JOIN   #key k ON k.DT_REP = pr.month1;

/* ── 1) СНАПШОТ promo-портфеля на @Anchor (как в SP) ───────── */
IF OBJECT_ID('tempdb..#bal_src') IS NOT NULL DROP TABLE #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open  AS date)          AS dt_open,
    CAST(t.dt_close AS date)          AS dt_close,
    CAST(t.con_id   AS bigint)        AS con_id,
    CAST(t.cli_id   AS bigint)        AS cli_id,
    CAST(t.prod_id  AS int)           AS prod_id,
    CAST(t.out_rub  AS decimal(20,2)) AS out_rub,
    CAST(t.rate_con AS decimal(9,4))  AS rate_balance,
    t.rate_con_src,
    t.TSEGMENTNAME,
    CAST(r.rate     AS decimal(9,4))  AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)   -- ULTRA «нулевой» день → +1
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.prod_id      = 654;                    -- промо-продукт

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src (con_id, dt_rep);

/* rate_use на @Anchor */
IF OBJECT_ID('tempdb..#bal_prom') IS NOT NULL DROP TABLE #bal_prom;
;WITH bal_pos AS (
    SELECT *,
           MIN(CASE WHEN rate_balance > 0
                     AND rate_con_src = N'счет ультра,вручную'
                    THEN rate_balance END)
               OVER (PARTITION BY con_id ORDER BY dt_rep
                     ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep = @Anchor
),
rate_calc AS (
    SELECT *,
           CASE
             WHEN rate_liq IS NULL THEN
                  CASE WHEN rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
                       ELSE rate_balance END
             WHEN rate_liq < 0  AND rate_balance > 0 THEN rate_balance
             WHEN rate_liq < 0  AND rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
             WHEN rate_liq >= 0 AND rate_balance >= 0 THEN rate_liq
             WHEN rate_liq > 0  AND rate_balance < 0 THEN rate_liq
             ELSE rate_liq
           END AS rate_use
    FROM bal_pos
)
SELECT
    con_id,
    cli_id,
    out_rub,
    rate_anchor = CAST(rate_use AS decimal(9,4)),
    dt_open,
    TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL;

/* ── 1b) Нулевые договоры (кандидаты для перелива) ─────────── */
IF OBJECT_ID('tempdb..#bal_zero') IS NOT NULL DROP TABLE #bal_zero;
SELECT
    CAST(z.con_id   AS bigint)        AS con_id,
    CAST(z.cli_id   AS bigint)        AS cli_id,
    CAST(0.00       AS decimal(20,2)) AS out_rub,
    CAST(COALESCE(z.rate_con,
                  CASE COALESCE(NULLIF(z.TSEGMENTNAME,N''),N'Розничный бизнес')
                       WHEN N'ДЧБО' THEN (SELECT TOP(1) promo_rate FROM @PromoRates pr
                                           WHERE pr.month1=DATEFROMPARTS(YEAR(@Anchor),MONTH(@Anchor),1)
                                             AND pr.TSEGMENTNAME=N'ДЧБО')
                       ELSE (SELECT TOP(1) promo_rate FROM @PromoRates pr
                             WHERE pr.month1=DATEFROMPARTS(YEAR(@Anchor),MONTH(@Anchor),1)
                               AND pr.TSEGMENTNAME=N'Розничный бизнес') END)
         AS decimal(9,4))              AS rate_anchor,
    CAST(z.dt_open AS date)            AS dt_open,
    COALESCE(NULLIF(z.TSEGMENTNAME,N''),N'Розничный бизнес') AS TSEGMENTNAME
INTO #bal_zero
FROM ALM.ehd.import_ZeroContracts z
WHERE z.dt_open <= @Anchor;

/* общий пул (в объёмы нулевые не попадают) */
IF OBJECT_ID('tempdb..#bal_prom_all') IS NOT NULL DROP TABLE #bal_prom_all;
SELECT * INTO #bal_prom_all FROM (
    SELECT con_id, cli_id, out_rub, rate_anchor, dt_open, TSEGMENTNAME FROM #bal_prom
    UNION ALL
    SELECT con_id, cli_id, out_rub, rate_anchor, dt_open, TSEGMENTNAME FROM #bal_zero
) u;

DECLARE @PromoTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #bal_prom);

/* ── 2) Циклы (base_day = EOM(next), promo_end = base_day-1) ── */
IF OBJECT_ID('tempdb..#cycles') IS NOT NULL DROP TABLE #cycles;
;WITH seed AS (
    SELECT
        b.con_id, b.cli_id, b.TSEGMENTNAME, b.out_rub,
        CAST(0 AS int) AS cycle_no,
        b.dt_open AS win_start,
        EOMONTH(DATEADD(month,1, b.dt_open))                   AS base_day,
        DATEADD(day,-1, EOMONTH(DATEADD(month,1, b.dt_open)))  AS promo_end
    FROM #bal_prom_all b
),
seq AS (
    SELECT * FROM seed
    UNION ALL
    SELECT
        s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
        s.cycle_no + 1,
        DATEADD(day,1, s.base_day) AS win_start,
        EOMONTH(DATEADD(month,1, DATEADD(day,1, s.base_day))) AS base_day,
        DATEADD(day,-1, EOMONTH(DATEADD(month,1, DATEADD(day,1, s.base_day)))) AS promo_end
    FROM seq s
    WHERE DATEADD(day,1, s.base_day) <= @HorizonTo
)
SELECT * INTO #cycles FROM seq OPTION (MAXRECURSION 0);

/* ── 3) Дневная лента (кроме 1-х чисел) ─────────────────────── */
IF OBJECT_ID('tempdb..#daily_pre') IS NOT NULL DROP TABLE #daily_pre;
SELECT
    c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub,
    c.cycle_no, c.win_start, c.promo_end, c.base_day,
    d.d AS dt_rep,
    CASE
      WHEN d.d <= c.promo_end THEN
           CASE WHEN c.cycle_no = 0
                THEN bp.rate_anchor
                ELSE (sp.spread + k_open.KEY_RATE)  -- промо окна
           END
      WHEN d.d = c.base_day THEN @BaseRate
    END AS rate_con
INTO   #daily_pre
FROM   #cycles c
JOIN   #cal d        ON d.d BETWEEN c.win_start AND c.base_day
LEFT  JOIN #key k_open
       ON k_open.DT_REP = c.win_start          -- 1-е числа
LEFT  JOIN #spreads sp
       ON sp.month1 = DATEFROMPARTS(YEAR(c.win_start),MONTH(c.win_start),1)
      AND sp.TSEGMENTNAME = c.TSEGMENTNAME
JOIN   #bal_prom_all bp ON bp.con_id = c.con_id
WHERE  DAY(d.d) <> 1;

/* (опция) подменяем ставки на самом @Anchor «фактом» из DCR,
   чтобы «портфель на якоре» совпадал со SP */
IF OBJECT_ID('tempdb..#dcr_anchor') IS NOT NULL DROP TABLE #dcr_anchor;
SELECT dp.con_id, CAST(r.rate AS decimal(9,4)) AS rate_fact
INTO   #dcr_anchor
FROM   #daily_pre dp
JOIN   LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = dp.con_id
       AND @Anchor BETWEEN r.dt_from AND r.dt_to
WHERE  dp.dt_rep = @Anchor;

UPDATE dp
SET dp.rate_con = da.rate_fact
FROM #daily_pre dp
JOIN #dcr_anchor da ON da.con_id = dp.con_id
WHERE dp.dt_rep = @Anchor;

/* ── 4) BASE-DAY FULL SHIFT: Σ клиента → max ставка ────────── */
IF OBJECT_ID('tempdb..#base_dates') IS NOT NULL DROP TABLE #base_dates;
SELECT DISTINCT dp.cli_id, dp.dt_rep
INTO   #base_dates
FROM   #daily_pre dp
JOIN   #cycles c ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
WHERE  dp.dt_rep = c.base_day;

IF OBJECT_ID('tempdb..#daily_base_adj') IS NOT NULL DROP TABLE #daily_base_adj;
;WITH base_pool AS (
    SELECT dp.*
    FROM   #daily_pre dp
    JOIN   #base_dates bd ON bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep
),
ranked AS (
    SELECT bp.*,
           ROW_NUMBER() OVER (PARTITION BY bp.cli_id, bp.dt_rep
                              ORDER BY bp.rate_con DESC, bp.TSEGMENTNAME, bp.con_id) AS rn
    FROM   base_pool bp
),
sums AS (
    SELECT cli_id, dt_rep, SUM(out_rub) AS total_out
    FROM   base_pool
    GROUP  BY cli_id, dt_rep
)
SELECT
    r.con_id,
    r.cli_id,
    r.TSEGMENTNAME,
    out_rub  = CASE WHEN r.rn=1 THEN s.total_out ELSE CAST(0.00 AS decimal(20,2)) END,
    r.dt_rep,
    rate_con = r.rate_con
INTO   #daily_base_adj
FROM   ranked r
JOIN   sums   s ON s.cli_id=r.cli_id AND s.dt_rep=r.dt_rep;

/* ── 5) 1-е числа: кандидаты и full shift ───────────────────── */
IF OBJECT_ID('tempdb..#day1_candidates') IS NOT NULL DROP TABLE #day1_candidates;
;WITH marks AS (
    SELECT
      c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub, c.cycle_no, c.win_start, c.promo_end, c.base_day,
      DATEFROMPARTS(YEAR(DATEADD(month,1,c.win_start)), MONTH(DATEADD(month,1,c.win_start)), 1) AS m1_in,
      DATEADD(day,1,c.base_day) AS m1_new
    FROM #cycles c
),
cand AS (
    /* 1-е внутри текущего окна — ставка фикс окна */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no,
        dt_rep = m.m1_in,
        rate_con =
            CASE WHEN m.cycle_no=0
                 THEN bp.rate_anchor
                 ELSE (sp.spread + k_open.KEY_RATE)
            END
    FROM marks m
    JOIN #bal_prom_all bp ON bp.con_id = m.con_id
    LEFT JOIN #key     k_open ON k_open.DT_REP = m.win_start
    LEFT JOIN #spreads sp
           ON sp.month1 = DATEFROMPARTS(YEAR(m.win_start),MONTH(m.win_start),1)
          AND sp.TSEGMENTNAME = m.TSEGMENTNAME
    WHERE m.m1_in BETWEEN m.win_start AND m.promo_end
      AND m.m1_in BETWEEN @Anchor AND @HorizonTo

    UNION ALL

    /* 1-е нового окна — новая промо: KEY(1-е)+spread(month,segment) */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no + 1 AS cycle_no,
        dt_rep = m.m1_new,
        rate_con = sp2.spread + k_new.KEY_RATE
    FROM marks m
    LEFT JOIN #key     k_new ON k_new.DT_REP = m.m1_new
    LEFT JOIN #spreads sp2
           ON sp2.month1 = m.m1_new
          AND sp2.TSEGMENTNAME = m.TSEGMENTNAME
    WHERE m.m1_new BETWEEN @Anchor AND @HorizonTo
)
SELECT * INTO #day1_candidates FROM cand;

IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, SUM(out_rub) AS out_rub_sum
INTO   #cli_sum
FROM   #bal_prom
GROUP  BY cli_id;

IF OBJECT_ID('tempdb..#firstday_assigned') IS NOT NULL DROP TABLE #firstday_assigned;
;WITH ranked AS (
    SELECT c.*, ROW_NUMBER() OVER (PARTITION BY c.cli_id, c.dt_rep
                                   ORDER BY c.rate_con DESC, c.TSEGMENTNAME, c.con_id) AS rn
    FROM #day1_candidates c
)
SELECT
    con_id       = MAX(CASE WHEN rn=1 THEN con_id END),
    r.cli_id,
    TSEGMENTNAME = MAX(CASE WHEN rn=1 THEN TSEGMENTNAME END),
    out_rub      = cs.out_rub_sum,
    r.dt_rep,
    rate_con     = MAX(CASE WHEN rn=1 THEN rate_con END)
INTO   #firstday_assigned
FROM   ranked r
JOIN   #cli_sum cs ON cs.cli_id = r.cli_id
GROUP  BY r.cli_id, r.dt_rep, cs.out_rub_sum;

/* ── 6) Финальная promo-лента ───────────────────────────────── */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    /* все дни, кроме base-day у отмеченных клиентов (их заменим) */
    SELECT dp.con_id, dp.cli_id, dp.TSEGMENTNAME, dp.out_rub, dp.dt_rep, dp.rate_con
    FROM   #daily_pre dp
    WHERE  NOT EXISTS (SELECT 1 FROM #base_dates bd
                       WHERE bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep)

    UNION ALL
    /* base-day — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con FROM #daily_base_adj

    UNION ALL
    /* 1-е числа — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con FROM #firstday_assigned
) u
GROUP BY dt_rep;

/* ── 7) Контрольки ─────────────────────────────────────────── */
PRINT N'=== Promo spreads by month ===';  SELECT * FROM #spreads ORDER BY month1, TSEGMENTNAME;

PRINT N'=== Σ объёма (должен равняться Σ на Anchor) ===';
SELECT TOP (120) f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @PromoTotal AS decimal(20,2))
FROM WORK.Forecast_NS_Promo f ORDER BY f.dt_rep;
GO



Option Explicit

Sub BatchFill_FastAndSafe()
    Const SHEET_SVOD As String = "СВОД"
    Const SHEET_DATA As String = "ТАБЛИЦА"
    Const COL_AC As Long = 29      ' C2 берём из AC
    Const COL_J  As Long = 10      ' C3 обычно из J
    Const COL_AA As Long = 27      ' если AA=1 ...
    Const COL_B  As Long = 2       ' ... или B="MIN_BAL" => C3=30
    Const COL_AF As Long = 32      ' результат в AF

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim lastRow As Long, n As Long, i As Long
    Dim oldC2 As Variant, oldC3 As Variant, res As Variant
    Dim calcMode As XlCalculation
    Dim arrAC As Variant, arrJ As Variant, arrAA As Variant, arrB As Variant
    Dim outArr() As Variant
    Dim vC2 As Variant, vC3 As Variant, flagAA As Variant, txtB As String
    Dim key As String
    Dim dict As Object

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    lastRow = wsData.Cells(wsData.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    arrAC = wsData.Range(wsData.Cells(2, COL_AC), wsData.Cells(lastRow, COL_AC)).Value
    arrJ  = wsData.Range(wsData.Cells(2, COL_J),  wsData.Cells(lastRow, COL_J)).Value
    arrAA = wsData.Range(wsData.Cells(2, COL_AA), wsData.Cells(lastRow, COL_AA)).Value
    arrB  = wsData.Range(wsData.Cells(2, COL_B),  wsData.Cells(lastRow, COL_B)).Value
    ReDim outArr(1 To n, 1 To 1)

    Set dict = CreateObject("Scripting.Dictionary")
    dict.CompareMode = 1  ' TextCompare

    oldC2 = wsSvod.Range("C2").Value2
    oldC3 = wsSvod.Range("C3").Value2

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.DisplayAlerts = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.Cursor = xlWait

    Application.CalculateFullRebuild

    For i = 1 To n
        On Error GoTo RowSoft

        vC2 = arrAC(i, 1)
        flagAA = arrAA(i, 1)
        txtB = UCase$(Trim$(CStr(arrB(i, 1))))
        If (flagAA = 1) Or (txtB = "MIN_BAL") Then
            vC3 = 30
        Else
            vC3 = arrJ(i, 1)
        End If

        key = CStr(vC2) & "|" & CStr(vC3)

        If dict.Exists(key) Then
            res = dict(key)
        Else
            wsSvod.Range("C2").Value2 = vC2
            wsSvod.Range("C3").Value2 = vC3

            Application.Calculate
            Do While Application.CalculationState <> xlDone
                DoEvents
            Loop

            res = wsSvod.Range("C35").Value2
            If IsError(res) Or LenB(res) = 0 Then
                res = Empty
            End If

            dict.Add key, res
        End If

        outArr(i, 1) = res

        If (i Mod 500 = 0) Then DoEvents
        GoTo NextI

RowSoft:
        outArr(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    wsData.Range(wsData.Cells(2, COL_AF), wsData.Cells(lastRow, COL_AF)).Value = outArr

Done:
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3

    Application.Cursor = xlDefault
    Application.Calculation = calcMode
    Application.DisplayAlerts = True
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

Fatal:
    On Error Resume Next
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Resume Done
End Sub
