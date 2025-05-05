    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        Точная копия вашей оригинальной функции scurve_by_tenor + вложенного scurve_from_arctan,
        без внешней передачи xgrid — внутри каждый раз строит свою сетку по min/max.
        """
        import numpy as np
        from scipy.optimize import minimize, NonlinearConstraint

        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])

        # глобальный диапазон Incentive (для справки)
        x_min_glob = data['Incentive'].min()
        x_max_glob = data['Incentive'].max()

        # итоговые DataFrame
        CPR_fitted_full = pd.DataFrame()
        coefs_full      = pd.DataFrame()

        # ваша вложенная scurve_from_arctan
        def scurve_from_arctan(scurve_input: dict):
            x = np.array(scurve_input['Incentive'])
            y = np.array(scurve_input['CPR'])
            d = np.array(scurve_input['TotalDebtBln'])

            lb_rate = scurve_input.get('lb_rate', -100)
            ub_rate = scurve_input.get('ub_rate',  40)
            mask    = (x >= lb_rate) & (x <= ub_rate)
            x, y, d = x[mask], y[mask], d[mask]
            w       = np.ones_like(d) if d.sum()==0 else d/d.sum()

            def fit_formula(b, xx):
                return (b[0]
                        + b[1]*np.arctan(b[2] + b[3]*xx)
                        + b[4]*np.arctan(b[5] + b[6]*xx))

            def func2min(b):
                return np.sum(w * (y - fit_formula(b, x))**2)

            points = [
                -100.0, -30.0, -25.0, -20.0, -19.0, -18.0, -17.0, -16.0,
                -15.0, -14.0, -13.0, -12.0, -11.0, -10.0,  -9.0,  -8.0,
                 -7.0,  -6.0,  -5.0,  -4.0,  -3.0,  -2.0,  -1.0,   0.0,
                  1.0,   1.25,  1.50,  2.0,   3.0,   4.0,   5.0,   6.0,
                  7.0,   8.0,  30.0
            ]

            c_prev   = scurve_input.get('coefs', [])
            direction= scurve_input.get('direction', 'down')

            def constraint_f(b):
                return np.array([fit_formula(b, p) for p in points])

            if len(c_prev)==7:
                prev_vals = np.array([fit_formula(c_prev, p) for p in points])
                if direction=='up':
                    lb_nonlin, ub_nonlin = prev_vals, np.ones_like(prev_vals)
                else:
                    lb_nonlin, ub_nonlin = np.zeros_like(prev_vals), prev_vals
            else:
                lb_nonlin, ub_nonlin = 0, 1

            nlc = NonlinearConstraint(constraint_f, lb_nonlin, ub_nonlin)

            bounds = [
                [-np.inf, np.inf],
                [0, np.inf],
                [-np.inf, 0],
                [0, 4],
                [0, np.inf],
                [0, np.inf],
                [0, 1]
            ]
            start_values = [0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2]

            res = minimize(
                func2min,
                x0=start_values,
                bounds=bounds,
                constraints=nlc,
                method='SLSQP',
                options={'ftol':1e-9}
            )
            betas = res.x

            # здесь строим *свою* сетку, как в оригинале:
            step   = 0.1
            x_min  = scurve_input.get('min', -40.0)
            x_max  = scurve_input.get('max',  40.0)
            x_grid = np.round(
                np.linspace(x_min, x_max, int((x_max-x_min)/step)+1),
                1
            )
            x_clip = np.array([max(lb_rate, min(xx, ub_rate)) for xx in x_grid])
            s_curve= fit_formula(betas, x_clip).tolist()

            return {
                's_curve': s_curve,
                'min':     x_min,
                'max':     x_max,
                'step':    step,
                'coefs':   betas
            }

        # ====== цикл по датам ======
        dates = sorted(data['Date'].dt.normalize().unique())
        for dt in dates:
            dfp = data[data['Date']==dt].copy()

            # создаём DataFrame с индексом x_grid (локальным!)
            x_min  = dfp['Incentive'].min()
            x_max  = dfp['Incentive'].max()
            idx    = np.round(
                        np.linspace(x_min, x_max, int((x_max-x_min)*10+1)),
                        1
                     )
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt

            coefs_list = []
            # most represented tenor
            rep = dfp.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(dfp['LoanAge'].unique())

            # 1) базовая
            aux = dfp[dfp['LoanAge']==rep]
            inp = {
                'Incentive':   aux['Incentive'],
                'CPR':         aux['CPR'],
                'TotalDebtBln':aux['TotalDebtBln'],
                'min':         x_min,
                'max':         x_max,
                'coefs':       []
            }
            r = scurve_from_arctan(inp)
            cpr_dt[f'CPR_fitted_{rep}'] = r['s_curve']
            coefs_main = r['coefs']
            coefs_list.append([dt, rep, *coefs_main])

            # 2) вниз (up)
            tmp = coefs_main
            for t in sorted([t for t in tenors if t<rep], reverse=True):
                aux = dfp[dfp['LoanAge']==t]
                inp = {
                    'Incentive':   aux['Incentive'],
                    'CPR':         aux['CPR'],
                    'TotalDebtBln':aux['TotalDebtBln'],
                    'min':         x_min,
                    'max':         x_max,
                    'coefs':       tmp,
                    'direction':   'up'
                }
                r = scurve_from_arctan(inp)
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                tmp = r['coefs']
                coefs_list.append([dt, t, *tmp])

            # 3) вверх (down)
            tmp = coefs_main
            for t in [t for t in tenors if t>rep]:
                aux = dfp[dfp['LoanAge']==t]
                inp = {
                    'Incentive':   aux['Incentive'],
                    'CPR':         aux['CPR'],
                    'TotalDebtBln':aux['TotalDebtBln'],
                    'min':         x_min,
                    'max':         x_max,
                    'coefs':       tmp,
                    'direction':   'down'
                }
                r = scurve_from_arctan(inp)
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                tmp = r['coefs']
                coefs_list.append([dt, t, *tmp])

            # сохраняем
            cpr_dt = cpr_dt[['Date'] + [f'CPR_fitted_{t}' for t in tenors]]
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_dt])

            cf = pd.DataFrame(
                coefs_list,
                columns=['Date','LoanAge'] + [f'b{i}' for i in range(7)]
            )
            coefs_full = pd.concat([coefs_full, cf])

        # итоговая упаковка
        coefs_full = coefs_full.sort_values(['Date','LoanAge']).reset_index(drop=True)
        coefs_full['ID'] = coefs_full.index+1

        CPR_fitted_full = CPR_fitted_full.rename_axis('Incentive').reset_index()
        return CPR_fitted_full, coefs_full
