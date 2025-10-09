> Mark:
﻿
 
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
Cell In[26], line 1
----> 1 res05 = run_step05_exploratory(
      2     df_raw_program=df_raw_program,
      3     out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",  # куда сохранять отчёт
      4     program_name="Семейная ипотека",                           # название программы для заголовков
      5     stim_bin_width=0.5,                                        # шаг по стимулу, п.п.
      6     last_months=6,                                             # сколько последних месяцев анализировать
      7     top_k_bins_for_stack=10                                    # сколько верхних бинов выводить в стэк-графике
      8 )
 
Cell In[25], line 113, in run_step05_exploratory(df_raw_program, out_root, program_name, stim_bin_width, last_months, top_k_bins_for_stack)
    107 agg_all = gb_all.agg(
    108     sum_od=("od_after_plan", "sum"),
    109     sum_premat=("premat_payment", "sum")
    110 ).reset_index()
    112 # количество строк (устойчиво к отсутствию con_id)
--> 113 agg_all = agg_all.join(gb_all.size().rename("n_rows"), how="left").reset_index(drop=True)
    115 # CPR и доли — строго векторно (никаких списков)
    116 agg_all = agg_all.assign(
    117     CPR_agg=np.where(
    118         agg_all["sum_od"] > 0,
  (...)
    121     )
    122 )
 
File ~\AppData\Local\anaconda3\Lib\site-packages\pandas\core\frame.py:10757, in DataFrame.join(self, other, on, how, lsuffix, rsuffix, sort, validate)
  10747     if how == "cross":
  10748         return merge(
  10749             self,
  10750             other,
   (...)
  10755             validate=validate,
  10756         )
> 10757     return merge(
  10758         self,
  10759         other,
  10760         left_on=on,
  10761         how=how,
  10762         left_index=on is None,
  10763         right_index=True,
  10764         suffixes=(lsuffix, rsuffix),
  10765         sort=sort,
  10766         validate=validate,
  10767     )
  10768 else:
  10769     if on is not None:
 
File ~\AppData\Local\anaconda3\Lib\site-packages\pandas\core\reshape\merge.py:184, in merge(left, right, how, on, left_on, right_on, left_index, right_index, sort, suffixes, copy, indicator, validate)
    169 else:
    170     op = _MergeOperation(
    171         left_df,
    172         right_df,
   (...)
    182         validate=validate,
    183     )
--> 184     return op.get_result(copy=copy)
 
File ~\AppData\Local\anaconda3\Lib\site-packages\pandas\core\reshape\merge.py:886, in _MergeOperation.get_result(self, copy)
    883 if self.indicator:
    884     self.left, self.right = self._indicator_pre_merge(self.left, self.right)
--> 886 join_index, left_indexer, right_indexer = self._get_join_info()
    888 result = self._reindex_and_concat(
    889     join_index, left_indexer, right_indexer, copy=copy
    890 )
    891 result = result.__finalize__(self, method=self._merge_type)
 
File ~\AppData\Local\anaconda3\Lib\site-packages\pandas\core\reshape\merge.py:1137, in _MergeOperation._get_join_info(self)
   1134 right_ax = self.right.index
   1136 if self.left_index and self.right_index and self.how != "asof":
-> 1137     join_index, left_indexer, right_indexer = left_ax.join(
   1138         right_ax, how=self.how, return_indexers=True, sort=self.sort
   1139     )
   1141 elif self.right_index and self.how == "left":
   1142     join_index, left_indexer, right_indexer = _left_join_on_index(
   1143         left_ax, right_ax, self.left_join_keys, sort=self.sort
   1144     )
 
File ~\AppData\Local\anaconda3\Lib\site-packages\pandas\core\indexes\base.py:279, in _maybe_return_indexers.<locals>.join(self, other, how, level, return_indexers, sort)
    269 @functools.wraps(meth)
    270 def join(
    271     self,
   (...)
    277     sort: bool = False,
    278 ):
--> 279     join_index, lidx, ridx = meth(self, other, how=how, level=level, sort=sort)
    280     if not return_indexers:
    281         return join_index
 

> Mark:
File ~\AppData\Local\anaconda3\Lib\site-packages\pandas\core\indexes\base.py:4615, in Index.join(self, other, how, level, return_indexers, sort)
   4613         pass
   4614     else:
-> 4615         return self._join_multi(other, how=how)
   4617 # join on the level
   4618 if level is not None and (self._is_multi or other._is_multi):
 
File ~\AppData\Local\anaconda3\Lib\site-packages\pandas\core\indexes\base.py:4739, in Index._join_multi(self, other, how)
   4737 # need at least 1 in common
   4738 if not overlap:
-> 4739     raise ValueError("cannot join with no overlapping index names")
   4741 if isinstance(self, MultiIndex) and isinstance(other, MultiIndex):
   4742     # Drop the non-matching levels from left and right respectively
   4743     ldrop_names = sorted(self_names - overlap, key=self_names_order)
 
ValueError: cannot join with no overlapping index names
 
