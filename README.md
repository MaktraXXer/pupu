=1
 + SUMPRODUCT(
       (($A$1:$A$10 > A1) * ISNUMBER($A$1:$A$10)) 
     / IF(ISNUMBER($A$1:$A$10);                     
          COUNTIF($A$1:$A$10; $A$1:$A$10);
          1                                        
       )
   )
