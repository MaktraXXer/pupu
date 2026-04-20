=IF($AN19<=decr_term;IF($AN19=1;(base_rate-discount-$B$4)*100;((base_rate-discount-$B$4)+($AQ$18-INDEX($AQ:$AQ;ROW()-1)))*100);IF($AN19=1;(base_rate-$B$4)*100;((base_rate-$B$4)+($AQ$18-INDEX($AQ:$AQ;ROW()-1)))*100))

=IF($B$1=TRUE;cpr_const;SUMPRODUCT(('Параметры S-curve'!$AQ$3:$AZ$503)*('Параметры S-curve'!$AP$3:$AP$503=ROUND(IF($AN19<=decr_term;IF($AN19=1;(base_rate-discount-$B$4)*100;((base_rate-discount-$B$4)+($AQ$18-INDEX($AQ:$AQ;ROW()-1)))*100);IF($AN19=1;(base_rate-$B$4)*100;((base_rate-$B$4)+($AQ$18-INDEX($AQ:$AQ;ROW()-1)))*100));1))*('Параметры S-curve'!$AQ$2:$AZ$2=$AX19)))
