    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        Расчитанные S-кривые и β-коэффициенты.
        Теперь scurve_from_arctan принимает готовый xgrid = idx.
        """
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])

        # 1) Глобальная сетка Incentive по всему датасету
        x_min = data['Incentive'].min()
        x_max = data['Incentive'].max()
        idx   = np.round(np.linspace(x_min, x_max,
                                     int((x_max - x_min)*10 + 1)), 1)

        # Итоговые датафреймы
        CPR_fitted_full = pd.DataFrame()
        coefs_full      = pd.DataFrame()

        dates = sorted(data['Date'].dt.normalize().unique())

        # Вложенный расчёт для одной выдержки
        def scurve_from_arctan(inp: dict):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])

            # границы стимула
            lb = inp.get('lb_rate', -100)
            ub = inp.get('ub_rate',  40)
            mask = (x >= lb) & (x <= ub)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum() == 0 else d / d.sum()

            # формула S-кривой
            def f(b, xx):
                return (b[0]
                        + b[1] * np.arctan(b[2] + b[3] * xx)
                        + b[4] * np.arctan(b[5] + b[6] * xx))

            # целевая
            def func2min(b):
                return np.sum(w * (y - f(b, x))**2)

            # ограничения на points_for_constraints (не меняем)
            points = [ -100, -30, -25, -20, -19, -18, -17, -16,
                       -15, -14, -13, -12, -11, -10,  -9,  -8,
                        -7,  -6,  -5,  -4,  -3,  -2,  -1,   0,
                         1, 1.25, 1.5,   2,   3,   4,   5,   6,
                         7,   8,  30 ]

            c_prev = inp.get('coefs', [])
            direct = inp.get('direction', 'down')

            def constraint_f(b):
                return np.array([f(b, p) for p in points])

            if len(c_prev) == 7:
                prev_vals = np.array([f(c_prev, p) for p in points])
                if direct == 'up':
                    lb_nonlin, ub_nonlin = prev_vals, np.ones_like(prev_vals)
                else:
                    lb_nonlin, ub_nonlin = np.zeros_like(prev_vals), prev_vals
            else:
                lb_nonlin, ub_nonlin = 0, 1

            nlc = NonlinearConstraint(constraint_f, lb_nonlin, ub_nonlin)

            bounds = [[-np.inf, np.inf],
                      [0, np.inf],
                      [-np.inf, 0],
                      [0, 4],
                      [0, np.inf],
                      [0, np.inf],
                      [0, 1]]
            start  = [0.2, 0.05, -2, 2.2, 0.07, 2, 0.2]

            res = minimize(func2min,
                           x0=start,
                           bounds=bounds,
                           constraints=nlc,
                           method='SLSQP',
                           options={'ftol':1e-9})
            betas = res.x

            # здесь вместо собственной сетки используем внешнюю idx
            xgrid = inp['xgrid']
            # «обрезаем» хвосты до [lb,ub]:
            xclip = np.array([max(lb, min(xi, ub)) for xi in xgrid])
            s_curve = f(betas, xclip).tolist()

            return {'s_curve': s_curve, 'coefs': betas}

        # Основной цикл по датам
        for dt in dates:
            dfp = data[data['Date'] == dt].copy()
            # результирующий DF для этой даты
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt

            coefs_list = []
            # наиболее весомая выдержка
            rep = dfp.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(dfp['LoanAge'].unique())

            # 1) базовая
            aux = dfp[dfp['LoanAge']==rep]
            inp = dict(
                Incentive   = aux['Incentive'],
                CPR         = aux['CPR'],
                TotalDebtBln= aux['TotalDebtBln'],
                xgrid       = idx,      # <-- передаём сюда
            )
            r = scurve_from_arctan(inp)
            cpr_dt[f'CPR_fitted_{rep}'] = r['s_curve']
            coefs_main = r['coefs']
            coefs_list.append([dt, rep, *coefs_main])

            # 2) 이하 ('up')
            prev = coefs_main
            for t in sorted([t for t in tenors if t < rep], reverse=True):
                aux = dfp[dfp['LoanAge']==t]
                inp = dict(
                    Incentive   = aux['Incentive'],
                    CPR         = aux['CPR'],
                    TotalDebtBln= aux['TotalDebtBln'],
                    xgrid       = idx,
                    coefs       = prev,
                    direction   = 'up'
                )
                r = scurve_from_arctan(inp)
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                prev = r['coefs']
                coefs_list.append([dt, t, *prev])

            # 3) выше ('down')
            prev = coefs_main
            for t in [t for t in tenors if t > rep]:
                aux = dfp[dfp['LoanAge']==t]
                inp = dict(
                    Incentive   = aux['Incentive'],
                    CPR         = aux['CPR'],
                    TotalDebtBln= aux['TotalDebtBln'],
                    xgrid       = idx,
                    coefs       = prev,
                    direction   = 'down'
                )
                r = scurve_from_arctan(inp)
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                prev = r['coefs']
                coefs_list.append([dt, t, *prev])

            # соберём в полные DF
            cpr_dt = cpr_dt[['Date'] + [f'CPR_fitted_{t}' for t in tenors]]
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_dt])

            cf = pd.DataFrame(coefs_list,
                              columns=['Date','LoanAge'] + [f'b{i}' for i in range(7)])
            coefs_full = pd.concat([coefs_full, cf])

        # финальная сортировка
        coefs_full = coefs_full.sort_values(['Date','LoanAge']).reset_index(drop=True)
        coefs_full['ID'] = coefs_full.index + 1

        # сброс индексa у CPR_fitted_full
        CPR_fitted_full = CPR_fitted_full.rename_axis('Incentive').reset_index()

        return CPR_fitted_full, coefs_full
