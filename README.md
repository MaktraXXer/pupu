    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        Расчет S-кривых и β-коэффициентов для каждого Date и каждого LoanAge.
        Использует единую сетку Incentive (idx), передаваемую в scurve_from_arctan.
        """
        import numpy as np
        from scipy.optimize import minimize, NonlinearConstraint

        # копируем и приводим даты
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])

        # 1) строим единую сетку Incentive
        step  = 0.1
        x_min = data['Incentive'].min()
        x_max = data['Incentive'].max()
        # np.arange с поправкой на плавающую точку
        idx   = np.round(np.arange(x_min, x_max + step/2, step), 1)

        # итоговые фреймы
        CPR_fitted_full = pd.DataFrame()
        coefs_full      = pd.DataFrame()

        # список уникальных дат (нормализованных)
        dates = sorted(data['Date'].dt.normalize().unique())

        # вложенная функция: оптимизация одной S-кривой
        def scurve_from_arctan(inp: dict):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])

            lb = inp.get('lb_rate', -100)
            ub = inp.get('ub_rate',  40)
            mask = (x >= lb) & (x <= ub)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum() == 0 else d / d.sum()

            def f(b, xx):
                return (b[0]
                        + b[1] * np.arctan(b[2] + b[3] * xx)
                        + b[4] * np.arctan(b[5] + b[6] * xx))

            def func2min(b):
                return np.sum(w * (y - f(b, x))**2)

            points = [
                -100.0, -30.0, -25.0, -20.0, -19.0, -18.0, -17.0, -16.0,
                -15.0, -14.0, -13.0, -12.0, -11.0, -10.0,  -9.0,  -8.0,
                 -7.0,  -6.0,  -5.0,  -4.0,  -3.0,  -2.0,  -1.0,   0.0,
                  1.0,   1.25,  1.50,  2.0,   3.0,   4.0,   5.0,   6.0,
                  7.0,   8.0,  30.0
            ]

            c_prev    = inp.get('coefs', [])
            direction = inp.get('direction', 'down')

            def constraint_f(b):
                return np.array([f(b, p) for p in points])

            if len(c_prev) == 7:
                prev_vals = np.array([f(c_prev, p) for p in points])
                if direction == 'up':
                    lb_nl = prev_vals
                    ub_nl = np.ones_like(prev_vals)
                else:
                    lb_nl = np.zeros_like(prev_vals)
                    ub_nl = prev_vals
            else:
                lb_nl, ub_nl = 0, 1

            nlc = NonlinearConstraint(constraint_f, lb_nl, ub_nl)

            bounds       = [
                [-np.inf, np.inf],
                [0,        np.inf],
                [-np.inf,  0],
                [0,        4],
                [0,        np.inf],
                [0,        np.inf],
                [0,        1]
            ]
            start_values = [0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2]

            res   = minimize(func2min,
                             x0=start_values,
                             bounds=bounds,
                             constraints=nlc,
                             method='SLSQP',
                             options={'ftol':1e-9})
            betas = res.x

            # строим S-кривую на единой сетке idx
            xgrid = inp['xgrid']
            xclip = np.clip(xgrid, lb, ub)
            s_curve = f(betas, xclip).tolist()

            return {'s_curve': s_curve, 'coefs': betas}

        # основной цикл по датам
        for dt in dates:
            dfp = data[data['Date'] == dt].copy()

            # DataFrame для текущей даты с индексом idx
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt

            coefs_list = []
            # определяем LoanAge с макс. суммой долга
            rep    = dfp.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(dfp['LoanAge'].unique())

            # 1) S-кривая для most represented tenor
            aux = dfp[dfp['LoanAge'] == rep]
            inp = {
                'Incentive':    aux['Incentive'],
                'CPR':          aux['CPR'],
                'TotalDebtBln': aux['TotalDebtBln'],
                'min':          x_min,
                'max':          x_max,
                'coefs':        [],
                'xgrid':        idx
            }
            r = scurve_from_arctan(inp)
            cpr_dt[f'CPR_fitted_{rep}'] = r['s_curve']
            coefs_main = r['coefs']
            coefs_list.append([dt, rep, *coefs_main])

            # 2) LoanAge < rep («up»)
            prev = coefs_main
            for t in sorted([t for t in tenors if t < rep], reverse=True):
                aux = dfp[dfp['LoanAge'] == t]
                inp = {
                    'Incentive':    aux['Incentive'],
                    'CPR':          aux['CPR'],
                    'TotalDebtBln': aux['TotalDebtBln'],
                    'min':          x_min,
                    'max':          x_max,
                    'coefs':        prev,
                    'direction':    'up',
                    'xgrid':        idx
                }
                r    = scurve_from_arctan(inp)
                prev = r['coefs']
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coefs_list.append([dt, t, *prev])

            # 3) LoanAge > rep («down»)
            prev = coefs_main
            for t in sorted([t for t in tenors if t > rep]):
                aux = dfp[dfp['LoanAge'] == t]
                inp = {
                    'Incentive':    aux['Incentive'],
                    'CPR':          aux['CPR'],
                    'TotalDebtBln': aux['TotalDebtBln'],
                    'min':          x_min,
                    'max':          x_max,
                    'coefs':        prev,
                    'direction':    'down',
                    'xgrid':        idx
                }
                r    = scurve_from_arctan(inp)
                prev = r['coefs']
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coefs_list.append([dt, t, *prev])

            # объединяем результаты
            cpr_dt = cpr_dt[['Date'] + [f'CPR_fitted_{t}' for t in tenors]]
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_dt])

            cf = pd.DataFrame(coefs_list,
                              columns=['Date','LoanAge'] + [f'b{i}' for i in range(7)])
            coefs_full = pd.concat([coefs_full, cf])

        # финальная упаковка
        coefs_full = coefs_full.sort_values(['Date','LoanAge']).reset_index(drop=True)
        coefs_full['ID'] = coefs_full.index + 1

        CPR_fitted_full = CPR_fitted_full.rename_axis('Incentive').reset_index()
        return CPR_fitted_full, coefs_full
