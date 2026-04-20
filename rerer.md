=IF($AN19<=1;(base_rate-IF($AN19<=decr_term;discount;0)-$B$4)*100;((base_rate-IF($AN19<=decr_term;discount;0)-$B$4)+($AQ$17-INDEX($AQ:$AQ;ROW()-1)))*100)

=IF($B$1=TRUE;cpr_const;SUMPRODUCT(('Параметры S-curve'!$AQ$3:$AZ$503)*('Параметры S-curve'!$AP$3:$AP$503=ROUND(IF($AN19<=1;(base_rate-IF($AN19<=decr_term;discount;0)-$B$4)*100;((base_rate-IF($AN19<=decr_term;discount;0)-$B$4)+($AQ$17-INDEX($AQ:$AQ;ROW()-1)))*100);1))*('Параметры S-curve'!$AQ$2:$AZ$2=$AX19)))
