# SQL
SQL queries for DB2


These are a few SQL queries/procedures that I've built and used for manipulating a DB2 database.

 P pull_splitF     b                                                            
 D wrkrowid        s                   like(xl2_lrecid) inz(*zeros)             
 D wrkdate         s                   like(xl2_lpymtdt) inz(*blanks)           
 D wrkpmt          s                   like(totpmt) inz(*zeros)                 
                                                                                
                                                                                
   dcl-s strSql   varchar(190);                                                 
   dcl-s x        int(3) inz(1);                                                
   dcl-s strId    varchar(10);                                                  
   dcl-s wrkInt   int(3) inz(*zeros);                                           
   dcl-s lowerVal varchar(10);                                                  
   dcl-s upperVal varchar(10);                                                  
   dcl-s wrkLen   int(3);                                                       
   dcl-s ejul     like(matdt) inz(*zeros);                                      
   dcl-s xjul     like(matdt) inz(*zeros);                                      
   dcl-s diffDays int(10) inz(0);                                               
   dcl-s EndRows  ind inz(*off);                                                
   dcl-s useSaved ind inz(*off);                                                
   
   //Get size of rowid and beginning 2 numbers                                        
   wrkLen = %len(%trim(%char(e_hrowid)));                                             
   wrkInt = %dec(%subst(%char(e_hrowid):1:2):2:0);                                    
                                                                                      
   //Get upper and lower ranges for rowid's in sql statement                          
   lowerVal = %subst(%char(e_hrowid):1:2);                                            
   wrkInt += 1;                                                                       
   upperVal = %char(wrkInt);                                                          
   wrkInt -= 1;
   
   // wrkLen - 2 for the first 2 numbers that we'll be keeping
   for x to wrkLen - 2;                                                               
     lowerVal += '0';                                                                 
     upperVal += '0';                                                                 
   endfor;   
   
   strSql = 'Select * from ELARHSTX where ' +                                          
            'lpymtamt + lsbszamt + lprnamt + lintamt + latefeamt + ' +                 
            'lescamt = ' + %char(totpmt) +  ' ' +                                      
            'and lrecid > ' + lowerVal + ' and lrecid < ' +  upperVal +                
            ' Order By lrecid';                                                        
                                                                                       
   Exec SQL Prepare S1 from :strSql;                                                   
   Exec SQL Declare C1 Cursor FOR S1;                                                  
   Exec SQL open C1;                                                                   
   Dow EndRows = *off;                                                                 
     Exec SQL fetch next from C1 into :elarhstxDS;                                     
                                                                                       
     EndRows = sqlstate = NoMoreRows;                                                  
                                                                                       
     if EndRows;                                                                       
       iter;                                                                           
     Endif;                                                                            
                                                                                       
     xjul = %int(@getdate(xl_lpymtdt));
     ejul = %int(@getdate(e_heffdt));                                                 
                                                                                      
     diffDays = %abs(xjul - ejul);                                                    
     sdiffDays = %abs(sjul - ejul);                                                   
                                                                                      
     //If the split transaction is within 4 days of the history date,                 
     //and the payments match, and the id exists within the created range,            
     //then I'm assuming this is the correct split                                    
     If diffDays <= 4;                                                                
       EndRows = *on;                                                                 
       useSaved = *off;                                                               
       prnpmt = xl_lprnamt;                                                           
       intpmt = xl_lintamt;                                                           
       escpmt = xl_lescamt;                                                           
       ltcpmt = xl_latefeamt;                                                         
       wrkpmt = prnpmt + intpmt + escpmt + ltcpmt;                                    
                                                                                      
     //Some splits are the correct record for the transaction in history                   
     //but the dates can differ by over a month                                        
     //so if the split date is within 60 days of the history transaction date         
     //I save and use the closest dated transaction to the history transaction date    
     Elseif diffDays <= 60 and sdiffDays > diffDays;                                  
       useSaved = *on;                                                                 
       sjul = xjul;                                                                    
       sdiffDays = diffDays;                                                           
       sprnpmt = xl_lprnamt;                                                           
       sintpmt = xl_lintamt;                                                           
       sescpmt = xl_lescamt;                                                           
       sltcpmt = xl_latefeamt;                                                         
     endif;                                                                            
                                                                                       
   Enddo;                                                                              
                                                                                       
   Exec SQL Close C1; 
   
   if useSaved;                                                     
     useSaved = *off;                                               
     sjul = 0;                                                      
     prnpmt = sprnpmt;                                              
     intpmt = sintpmt;                                              
     escpmt = sescpmt;                                              
     ltcpmt = sltcpmt;                                              
     wrkpmt = sprnpmt + sintpmt + sescpmt + sltcpmt;                
   endif;                                                           
   
