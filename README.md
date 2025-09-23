class SCurvesByProduct:
    ...
    @staticmethod
    def _fsname(v):
        # аккуратно превращаем любое значение в безопасное имя папки
        s = str(v)
        return s.replace(os.sep, "_")

    def run(self):
        self.load()

        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_root = self._ensure_dir(os.path.join(self.folder_path, ts))

        global_xmin = self.data['Incentive'].min()
        global_xmax = self.data['Incentive'].max()

        for p in self.products:
            dfp = self.data[self.data[self.product_col] == p].copy()
            if dfp.empty:
                continue

            # >>> ВАЖНО: приводим имя продукта к строке для путей
            p_fs = self._fsname(p)

            prod_root = self._ensure_dir(os.path.join(out_root, p_fs))
            excel_root = self._ensure_dir(os.path.join(prod_root, 'excel'))

            CPR_un, coefs_un = self._fit_unconstrained_one_product(dfp)
            self.cpr_un[p] = CPR_un
            self.coefs_un[p] = coefs_un

            self._to_excel(os.path.join(excel_root, 'coefs_unconstrained.xlsx'),
                           {'coefs_unconstrained': coefs_un})
            self._to_excel(os.path.join(excel_root, 'CPR_fitted_unconstrained.xlsx'),
                           {'CPR_fitted_unconstrained': CPR_un})

            self._plot_one_product(p, dfp, CPR_un, coefs_un, prod_root, global_xmin, global_xmax)

        self._plot_overlays_between_products(out_root)
