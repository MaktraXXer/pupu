=IF($AN19>1;IF(AN19<=decr_term;AU18;AT18)+($AQ17-$AQ18)*100;IF(AN19<=decr_term;(base_rate-discount-$B$4)*100;(base_rate-$B$4)*100))



=IF($B$1=TRUE;cpr_const;SUMPRODUCT(('Параметры S-curve'!$AQ$3:$AZ$503)*('Параметры S-curve'!$AP$3:$AP$503=ROUND(<ячейка_со_стимулом>;1))*('Параметры S-curve'!$AQ$2:$AZ$2=$AX19)))

=IF($B$1=TRUE;cpr_const;SUMPRODUCT(('Параметры S-curve'!$AQ$3:$AZ$503)*('Параметры S-curve'!$AP$3:$AP$503=ROUND(IF($AN19>1;IF(AN19<=decr_term;AU18;AT18)+($AQ17-$AQ18)*100;IF(AN19<=decr_term;(base_rate-discount-$B$4)*100;(base_rate-$B$4)*100));1))*('Параметры S-curve'!$AQ$2:$AZ$2=$AX19)))
