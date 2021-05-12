## Select pra pegar os usuários e grants do Oracle
```sh
declare 
  iSql VARCHAR2( 4000 ) ;
begin
  for cur in (select username from dba_users where username not in ('SYS', 'SYSTEM') order by username ) loop
   begin
    select dbms_metadata.get_ddl( 'USER', cur.username) into iSql from dual ;
    sys.dbms_output.put_line( '=======> Username : ' || cur.username || chr(13) || chr(10) || iSql || chr(13) || chr(10)) ; 
   exception 
   when others then
    dbms_output.put_line( 'Não foi possivel gerar DDL para : ' || cur.username || chr(13) || chr(10)) ;  
   end ;   
   
   begin 
    select DBMS_METADATA.GET_GRANTED_DDL('ROLE_GRANT', cur.username) into iSql  from dual;
    dbms_output.put_line( 'Role Grants: ' || iSql || chr(13) || chr(10)) ; 
   exception 
   when others then
    dbms_output.put_line( ' ** Não há roles' || chr(13) || chr(10)) ;   
   end ;
   
   begin
    select DBMS_METADATA.GET_GRANTED_DDL('SYSTEM_GRANT', cur.username) into iSql from dual;
    dbms_output.put_line( 'System Grants: ' || iSql || chr(13) || chr(10)) ; 
   exception 
   when others then
    dbms_output.put_line( ' ** Não há System Grants '  || chr(13) || chr(10)) ;  
   end;
   
   begin
    select DBMS_METADATA.GET_GRANTED_DDL('OBJECT_GRANT', cur.username) into iSql from dual;
    dbms_output.put_line( 'Objects Grants: ' || iSql ) ; 
  exception 
   when others then
    dbms_output.put_line( ' ** Não há Object Grants '  || chr(13) || chr(10)) ;    
   end;
   
  end loop; 
  
end;
/
```
