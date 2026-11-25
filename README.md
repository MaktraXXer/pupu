;WITH t AS (
    SELECT
        SUM(amount_in) AS full_in,
        SUM(amount_out) AS full_out,

        SUM(incoming_sr + incoming_ns + incoming_dvs
            + incoming_other + incoming_unknown) AS parts_in,

        SUM(outgoing_sr + outgoing_ns + outgoing_dvs
            + outgoing_other + outgoing_unknown) AS parts_out

    FROM ALM_TEST.WORK.RK_Oct2026_PaymentAgg
)
SELECT
    full_in / 1e9 AS full_in,
    parts_in / 1e9 AS parts_in,

    full_out / 1e9 AS full_out,
    parts_out / 1e9 AS parts_out,

    (full_in - parts_in) / 1e9 AS diff_in,
    (parts_out - full_out) / 1e9 AS diff_out,

    (full_in + full_out) / 1e9 AS full_saldo,
    (parts_in + parts_out) / 1e9 AS parts_saldo,

    (full_in + full_out - parts_in - parts_out)
        / 1e9 AS diff_saldo
FROM t
OPTION (MAXDOP 1);


;WITH t AS (
    SELECT
        SUM(amount_in) AS full_in,
        SUM(amount_out) AS full_out,

        SUM(incoming_sr + incoming_ns + incoming_dvs
            + incoming_other + incoming_unknown) AS parts_in,

        SUM(outgoing_sr + outgoing_ns + outgoing_dvs
            + outgoing_other + outgoing_unknown) AS parts_out

    FROM ALM_TEST.WORK.RK_Oct2026_PaymentAgg
)
SELECT
    full_in / 1e9 AS full_in,
    parts_in / 1e9 AS parts_in,

    full_out / 1e9 AS full_out,
    parts_out / 1e9 AS parts_out,

    (full_in - parts_in) / 1e9 AS diff_in,
    (parts_out - full_out) / 1e9 AS diff_out,

    (full_in + full_out) / 1e9 AS full_saldo,
    (parts_in + parts_out) / 1e9 AS parts_saldo,

    (full_in + full_out - parts_in - parts_out)
        / 1e9 AS diff_saldo
FROM t
OPTION (MAXDOP 1);
