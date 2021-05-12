## Select pra pegar os usuários e grants do Oracle
```sh
--##################################################################################################
                                                            #
--# Script para execucao em Oracle Database 11g SINGLE                                             #
--# Versao - v 2.1                                                                                 #
--#                                                                                                #
--# Instrucao de uso:                                                                              #
--# 1) Copiar o arquivo para um local no servidor com permissão de escrita no Sistema Operacional. #
--# 2) Executar o script na instancia Oracle com usuário de permissão SYSDBA.                      #
--# 3) enviar o arquivo gerado .lst para o time ORACLE requisitante.                                #
--##################################################################################################

set line 250
set long 999999
set timing off
set serveroutput on
set pagesize 0
set head on
set trimspool on
set feedback off
SET LINESIZE 500
SET PAGES 4000
SET VERIFY OFF
SET DEFINE ON
ALTER SESSION SET NLS_DATE_FORMAT='DD/MM/YYYY HH24:MI:SS';
ALTER SESSION SET NLS_TIMESTAMP_FORMAT='DD/MM/YYYY HH24:MI:SS';
ALTER SESSION SET NLS_TIMESTAMP_TZ_FORMAT='DD/MM/YYYY HH24:MI:SS TZR';

SET SERVEROUTPUT ON SIZE 1000000

DEFINE LOGNAME=DATE
COLUMN CLOGNAME NEW_VALUE LOGNAME
SELECT UPPER(HOST_NAME)||'_'||UPPER(INSTANCE_NAME)||'_'||TO_CHAR(SYSDATE,'YYYYMMDD')||'.lst' CLOGNAME FROM V$INSTANCE;
SPOOL '&LOGNAME'

SET DEFINE OFF
SET TRIMSPOOL ON

prompt 
prompt ===========================================================
prompt INSTANCE
prompt ===========================================================
prompt 
COL host_name FORMAT A30
COL version   FORMAT A10

SELECT instance_name,
       host_name,
       version,
       startup_time,
       status,
       instance_role
FROM v$instance;

prompt 
prompt ===========================================================
prompt DATABASE
prompt ===========================================================
prompt 
COL name          FORMAT A20
COL platform_name FORMAT A40
COL force_logging FORMAT A13

SELECT name,
       created,
       log_mode,
       open_mode,
       database_role,
       force_logging,
       platform_name
FROM v$database;

prompt 
prompt ===========================================================
prompt REGISTRY
prompt ===========================================================
prompt 
COL action_time   FORMAT A20
COL action        FORMAT A20
COL namespace     FORMAT A20
COL version       FORMAT A10
COL comments      FORMAT A30
COL bundle_series FORMAT A13

SELECT action_time,
       action,
       namespace,
       version,
       id,
       comments,
       bundle_series
FROM   sys.registry$history
ORDER BY action_time;

prompt 
prompt ===========================================================
prompt SGA
prompt ===========================================================
prompt 
COL value FORMAT 999999999999990

SELECT name,
       value
FROM   v$sga;

prompt 
prompt ===========================================================
prompt PROPERTIES
prompt ===========================================================
prompt 
COL property_name  FORMAT A40
COL property_value FORMAT A50

SELECT property_name,
       property_value
FROM   database_properties
ORDER BY property_name;

prompt 
prompt ===========================================================
prompt OPTIONS
prompt ===========================================================
prompt 
COL parameter FORMAT A50
COL value     FORMAT A10

SELECT *
FROM   v$option
ORDER BY parameter;

prompt 
prompt ===========================================================
prompt PARAMETERS
prompt ===========================================================
prompt 
COL name                  FORMAT A40
COL value                 FORMAT A60
COL isses_modifiable      FORMAT A16
COL issys_modifiable      FORMAT A16
COL isinstance_modifiable FORMAT A21

SELECT name,
       value,
       isses_modifiable,
       issys_modifiable,
       isinstance_modifiable
FROM   v$parameter
WHERE isdefault = 'FALSE'
ORDER BY name;

prompt 
prompt ===========================================================
prompt CONTROLFILES
prompt ===========================================================
prompt 
COL name FORMAT A80

SELECT name,
       status
FROM   v$controlfile
ORDER BY name;

prompt 
prompt ===========================================================
prompt REDO LOGFILES
prompt ===========================================================
prompt 
COL member FORMAT A80

SELECT group#,
       l.thread#,
       lf.member,
       l.bytes/1024/1024 size_mb,
       lf.type,
       nvl(lf.status,'IN USE') status,
       l.status group_status
FROM   v$logfile lf JOIN v$log l USING (group#)
ORDER BY group#, lf.member;

prompt 
prompt ===========================================================
prompt REDO LOGFILE SWITCHES PER HOUR (LAST 14 DAYS)
prompt ===========================================================
prompt 

SELECT *
FROM   (WITH switch_days AS (SELECT TRUNC(SYSDATE) - (level - 1) AS switch_day
                             FROM   dual
                             CONNECT BY level <= 14)
        SELECT TO_CHAR(nvl(TRUNC(first_time),switch_day),'DD/MM/YYYY') switch_day,
               TO_CHAR(first_time,'hh24') switch_hour
        FROM   switch_days LEFT OUTER JOIN v$loghist ON switch_day = TRUNC(first_time))
PIVOT  (COUNT(switch_hour) FOR (switch_hour) IN ('00' AS "00", '01' AS "01", '02' AS "02",
                                                 '03' AS "03", '04' AS "04", '05' AS "05",
                                                 '06' AS "06", '07' AS "07", '08' AS "08",
                                                 '09' AS "09", '10' AS "10", '11' AS "11",
                                                 '12' AS "12", '13' AS "13", '14' AS "14",
                                                 '15' AS "15", '16' AS "16", '17' AS "17",
                                                 '18' AS "18", '19' AS "19", '20' AS "20",
                                                 '21' AS "21", '22' AS "22", '23' AS "23"))
ORDER BY switch_day DESC;

prompt 
prompt ===========================================================
prompt TABLESPACES
prompt ===========================================================
prompt 
COL extent_management        FORMAT A17
COL allocation_type          FORMAT A15
COL segment_space_management FORMAT A24
COL bigfile                  FORMAT A7

SELECT tablespace_name,
       status,
       contents,
       logging,
       extent_management,
       allocation_type,
       segment_space_management,
       bigfile
FROM   dba_tablespaces
ORDER BY tablespace_name;

prompt 
prompt ===========================================================
prompt DATAFILES
prompt ===========================================================
prompt 
COL file_name      FORMAT A70
COL autoextensible FORMAT A15

SELECT file_id,
       file_name,
       ROUND(bytes/1024/1024) AS size_mb,
       ROUND(maxbytes/1024/1024) AS max_size_mb,
       autoextensible,
       increment_by,
       status
FROM   dba_data_files
ORDER BY file_name;

prompt 
prompt ===========================================================
prompt DATAFILES FREE SPACE
prompt ===========================================================
prompt 
COL used_pct FORMAT 900.00

SELECT df.tablespace_name,
       df.file_name,
       df.size_mb,
       f.free_mb,
       df.max_size_mb,
       f.free_mb + (df.max_size_mb - df.size_mb) AS max_free_mb,
       ROUND((df.max_size_mb-(f.free_mb + (df.max_size_mb - df.size_mb)))/max_size_mb*100,2) AS used_pct
FROM   (SELECT file_id,
               file_name,
               tablespace_name,
               TRUNC(bytes/1024/1024) AS size_mb,
               TRUNC(GREATEST(bytes,maxbytes)/1024/1024) AS max_size_mb
        FROM   dba_data_files) df,
       (SELECT TRUNC(SUM(bytes)/1024/1024) AS free_mb,
               file_id
        FROM dba_free_space
        GROUP BY file_id) f
WHERE  df.file_id = f.file_id (+)
ORDER BY df.tablespace_name,
         df.file_name;

prompt 
prompt ===========================================================
prompt INVALID OBJECTS
prompt ===========================================================
prompt 
COL object_name FORMAT A30

SELECT owner,
       object_type,
       COUNT(*) qty
FROM   dba_objects
WHERE  status = 'INVALID'
GROUP BY owner, object_type
ORDER BY owner, object_type;

prompt 
prompt ===========================================================
prompt UNUSABLE INDEXES
prompt ===========================================================
prompt 
SELECT owner,
       index_name
FROM   dba_indexes
WHERE  status NOT IN ('VALID', 'N/A')
ORDER BY owner, index_name;

prompt 
prompt ===========================================================
prompt DEBUG COMPILED OBJECTS
prompt ===========================================================
prompt 
SELECT owner,
       COUNT(*) qty
FROM  sys.all_probe_objects
WHERE debuginfo = 'T'
GROUP BY owner
ORDER BY owner;

prompt 
prompt ===========================================================
prompt OS STATISTICS
prompt ===========================================================
prompt 
COL pval2 FORMAT A30

SELECT *
FROM sys.aux_stats$;

prompt 
prompt ===========================================================
prompt OBJECT STATISTICS PREFERENCES
prompt ===========================================================
prompt 
COL value FORMAT A40

SELECT 'AUTOSTATS_TARGET' PARAMETER, DBMS_STATS.GET_PARAM('AUTOSTATS_TARGET') VALUE FROM dual
UNION
SELECT 'CASCADE', DBMS_STATS.GET_PARAM('CASCADE') FROM dual
UNION
SELECT 'DEGREE', DBMS_STATS.GET_PARAM('DEGREE') FROM dual
UNION
SELECT 'ESTIMATE_PERCENT', DBMS_STATS.GET_PARAM('ESTIMATE_PERCENT') FROM dual
UNION
SELECT 'GRANULARITY', DBMS_STATS.GET_PARAM('GRANULARITY') FROM dual
UNION
SELECT 'INCREMENTAL', DBMS_STATS.GET_PARAM('INCREMENTAL') FROM dual
UNION
SELECT 'METHOD_OPT', DBMS_STATS.GET_PARAM('METHOD_OPT') FROM dual
UNION
SELECT 'NO_INVALIDATE', DBMS_STATS.GET_PARAM('NO_INVALIDATE') FROM dual
UNION
SELECT 'PUBLISH', DBMS_STATS.GET_PARAM('PUBLISH') FROM dual
UNION
SELECT 'STALE_PERCENT', DBMS_STATS.GET_PARAM('STALE_PERCENT') FROM dual;

prompt 
prompt ===========================================================
prompt TABLE STATISTICS
prompt ===========================================================
prompt 
COL last_analyzed FORMAT A13

SELECT owner,
       TO_CHAR(last_analyzed, 'DD/MM/YYYY') last_analyzed,
       COUNT(*) qty
FROM (SELECT distinct
             owner,
             table_name,
             TRUNC(a.last_analyzed) last_analyzed
      FROM   dba_tab_statistics a JOIN dba_tables b USING (owner, table_name)
      WHERE  b.temporary = 'N'
      AND    stattype_locked IS NULL
      AND    index_type NOT IN ('LOB')
      ORDER BY TRUNC(a.last_analyzed) DESC NULLS FIRST)
GROUP BY owner, last_analyzed
ORDER BY owner, last_analyzed DESC NULLS FIRST;

prompt 
prompt ===========================================================
prompt TABLE LOCKED STATISTICS
prompt ===========================================================
prompt 

SELECT owner,
       table_name,
       stattype_locked
FROM   dba_tab_statistics
WHERE  stattype_locked IS NOT NULL
ORDER BY owner, table_name;

prompt 
prompt ===========================================================
prompt INDEXES STATISTICS
prompt ===========================================================
prompt 
SELECT owner,
       TO_CHAR(last_analyzed, 'DD/MM/YYYY') last_analyzed,
       COUNT(*) qty
FROM (SELECT distinct
             owner,
             index_name,
             TRUNC(a.last_analyzed) last_analyzed
      FROM   dba_ind_statistics a JOIN dba_indexes b USING (owner, index_name)
      WHERE  b.temporary = 'N'
      AND    stattype_locked IS NULL
      ORDER BY TRUNC(a.last_analyzed) DESC NULLS FIRST)
GROUP BY owner, last_analyzed
ORDER BY owner, last_analyzed DESC NULLS FIRST;

prompt 
prompt ===========================================================
prompt INDEX LOCKED STATISTICS
prompt ===========================================================
prompt 

SELECT owner,
       table_name,
       stattype_locked
FROM   dba_ind_statistics
WHERE  stattype_locked IS NOT NULL
ORDER BY owner, index_name;

prompt 
prompt ===========================================================
prompt FOREIGN KEY COLUMNS UNINDEXED
prompt ===========================================================
prompt 
COL column_name FORMAT A30

SELECT t.owner,
       t.table_name,
       c.constraint_name,
       c.table_name table2,
       acc.column_name
FROM   dba_constraints t,
       dba_constraints c,
       dba_cons_columns acc
WHERE  c.r_constraint_name = t.constraint_name
AND    c.table_name        = acc.table_name
AND    c.constraint_name   = acc.constraint_name
AND    NOT EXISTS (SELECT '1' 
                   FROM  dba_ind_columns aid
                   WHERE aid.table_name  = acc.table_name
                   AND   aid.column_name = acc.column_name)
ORDER BY t.owner, c.table_name;

prompt 
prompt ===========================================================
prompt MTTR ADVISOR
prompt ===========================================================
prompt 
SELECT mttr_target_for_estimate,
       estd_cache_write_factor,
       estd_total_write_factor,
       estd_total_io_factor
FROM   v$mttr_target_advice
ORDER BY mttr_target_for_estimate;

prompt 
prompt ===========================================================
prompt PGA ADVISOR
prompt ===========================================================
prompt 
COL pga_target_for_estimate FORMAT 999999999999990

SELECT pga_target_for_estimate,
       pga_target_factor,
       estd_pga_cache_hit_percentage
FROM v$pga_target_advice
ORDER BY pga_target_factor;

prompt 
prompt ===========================================================
prompt SGA ADVISOR
prompt ===========================================================
prompt 
COL estd_physical_reads FORMAT 999999999999990

SELECT sga_size,
       sga_size_factor,
       estd_db_time,
       estd_db_time_factor,
       estd_physical_reads
FROM v$sga_target_advice
ORDER BY sga_size_factor;

prompt 
prompt ===========================================================
prompt BUFFER CACHE ADVISOR
prompt ===========================================================
prompt 
COL name          FORMAT A50
COL advice_status FORMAT A13

SELECT name,
       block_size,
       advice_status,
       size_for_estimate,
       size_factor,
       estd_physical_read_factor
FROM v$db_cache_advice
ORDER BY name, size_factor;

prompt 
prompt ===========================================================
prompt BUFFER CACHE HIT RATIO%
prompt ===========================================================
prompt 
COL consistent_gets FORMAT 999999999999990
COL db_block_gets   FORMAT 999999999999990
COL physical_reads  FORMAT 999999999999990
COL hit_ratio       FORMAT 900.00

SELECT SUM( DECODE( a.name, 'consistent gets', a.value, 0 ) ) AS consistent_gets,
       SUM( DECODE( a.name, 'db block gets', a.value, 0 ) ) AS db_block_gets,
       SUM( DECODE( a.name, 'physical reads', a.value, 0 ) ) AS physical_reads,
       ROUND( ( ( SUM( DECODE( a.name, 'consistent gets', a.value, 0 ) ) +
                  SUM( DECODE( a.name, 'db block gets', a.value, 0 ) ) -
                  SUM( DECODE( a.name, 'physical reads', a.value, 0 ) ) ) /
              ( SUM( DECODE( a.name, 'consistent gets', a.value, 0 ) ) +
                SUM( DECODE( a.name, 'db block gets', a.value, 0 ) ) ) ) * 100, 2) AS hit_ratio
FROM   v$sysstat a;

prompt 
prompt ===========================================================
prompt SHARED POOL ADVISOR
prompt ===========================================================
prompt 
SELECT shared_pool_size_for_estimate,
       shared_pool_size_factor,
       estd_lc_size,
       estd_lc_time_saved_factor
FROM v$shared_pool_advice
ORDER BY shared_pool_size_factor;

prompt 
prompt ===========================================================
prompt LIBRARY CACHE HIT RATIO%
prompt ===========================================================
prompt 
COL get_ratio FORMAT 900.00
COL pin_ratio FORMAT 900.00

SELECT a.namespace,
       a.gets,
       a.gethits,
       ROUND( a.gethitratio * 100, 2 ) AS get_ratio,
       a.pins,
       a.pinhits,
       ROUND( a.pinhitratio * 100, 2 ) AS pin_ratio,
       a.reloads,
       a.invalidations
FROM   v$librarycache a
ORDER BY a.namespace;

prompt 
prompt ===========================================================
prompt DICTIONARY CACHE HIT RATIO%
prompt ===========================================================
prompt 
COL miss_hit_ratio FORMAT 900.00

SELECT parameter,
       gets,
       getmisses,
       modifications,
       flushes,
       ROUND( ( getmisses / DECODE( gets, 0, 1, gets ) ) * 100, 2 ) AS miss_hit_ratio
FROM v$rowcache
ORDER BY miss_hit_ratio DESC, parameter;

prompt 
prompt ===========================================================
prompt LATCH HIT RATIO%
prompt ===========================================================
prompt 
COL gets            FORMAT 999999999999990
COL misses          FORMAT 999999999999990
COL latch_hit_ratio FORMAT 900.00

SELECT l.name,
       l.gets,
       l.misses,
       ROUND( ( ( 1 - ( l.misses / l.gets ) ) * 100 ), 2 ) AS latch_hit_ratio
FROM   v$latch l
WHERE  l.gets != 0
UNION
SELECT l.name,
       l.gets,
       l.misses,
       100 AS latch_hit_ratio
FROM   v$latch l
WHERE  l.gets = 0
ORDER BY latch_hit_ratio, name;

prompt 
prompt ===========================================================
prompt SYSTEM EVENTS
prompt ===========================================================
prompt 
COL event      FORMAT A60
COL wait_class FORMAT A15
COL pct        FORMAT 900.00
COL pct_fg     FORMAT 900.00

SELECT *
FROM (SELECT event,
             total_waits,
             total_timeouts,
             time_waited / 100 time_waited_s,
             average_wait / 100 average_wait_s,
             ROUND( time_waited / total_waits.total_wait_time * 100, 2) AS pct,
             total_waits_fg,
             total_timeouts_fg,
             time_waited_fg / 100 time_waited_fg_s,
             average_wait_fg / 100 average_wait_fg_s,
             ROUND( time_waited_fg / total_waits.total_wait_time_fg * 100, 2) AS pct_fg,
             wait_class
      FROM   v$system_event,
             (SELECT SUM(time_waited) total_wait_time,
                     SUM(time_waited_fg) total_wait_time_fg
              FROM v$system_event
              WHERE wait_class NOT IN ('Idle')) total_waits
      WHERE  wait_class NOT IN ('Idle')
      ORDER BY time_waited DESC)
WHERE rownum <= 10;

prompt 
prompt ===========================================================
prompt ASM DISKGROUPS
prompt ===========================================================
prompt 
COL name                   FORMAT A20
COL compatibility          FORMAT A22
COL database_compatibility FORMAT A22

SELECT group_number,
       name,
       sector_size,
       block_size,
       allocation_unit_size au_size,
       type,
       total_mb,
       free_mb,
       usable_file_mb,
       compatibility,
       database_compatibility
FROM v$asm_diskgroup
ORDER BY group_number;

prompt 
prompt ===========================================================
prompt ASM DISKS
prompt ===========================================================
prompt 
COL mode_status   FORMAT A11
COL redundancy    FORMAT A10
COL path          FORMAT A30
COL bytes_read    FORMAT 999999999999990
COL bytes_written FORMAT 999999999999990

SELECT group_number,
       dg.name,
       dk.disk_number,
       dk.name,
       dk.mode_status,
       dk.state,
       dk.redundancy,
       dk.total_mb,
       dk.free_mb,
       dk.failgroup,
       dk.path,
       dk.reads,
       dk.writes,
       dk.read_errs,
       dk.write_errs,
       dk.read_time,
       dk.write_time,
       dk.bytes_read,
       dk.bytes_written
FROM v$asm_disk dk JOIN v$asm_diskgroup dg USING (group_number)
ORDER BY group_number, dk.disk_number;



prompt 
prompt ===========================================================
prompt VERSAO DO BANCO DE DADOS
prompt ===========================================================
prompt 

select * from v$version;

prompt 
prompt ===========================================================
prompt INSTANCIA DE BANCO DE DADOS
prompt ===========================================================
prompt 
set pagesize 10000
select a.instance_name as "INSTANCIA", 
       a.host_name as "SERVIDOR", 
       a.status as "STATUS", a.archiver as "MODO ARQUIVAMENTO"
from v$instance a;

prompt 
prompt ===========================================================
prompt BANCO DE DADOS
prompt ===========================================================
prompt 
set pagesize 10000
column platform_name format a30
column force_logging format a15
select a.dbid, a.name, a.log_mode, a.platform_name, a.flashback_on, a.db_unique_name
  from v$database a;

select a.protection_mode, protection_level, force_logging
  from v$database a;

prompt 
prompt ===========================================================
prompt PARAMETROS DO BANCO DE DADOS
prompt ===========================================================
prompt 
column Parametro format a35
column valor     format a45
select name Parametro, type tipo , value valor 
 from v$parameter
order by name;

prompt 
prompt ===========================================================
prompt MEMORIA UTILIZADA
prompt ===========================================================
prompt 
set numwidth 12
select * from V$SGA_TARGET_ADVICE order by 1;
select * from V$MEMORY_TARGET_ADVICE;

select 
   component, 
   oper_type, 
   oper_mode, 
   initial_size/1024/1024 "Initial", 
   TARGET_SIZE/1024/1024  "Target", 
   FINAL_SIZE/1024/1024   "Final", 
   status 
from 
   v$sga_resize_ops;

select 
   component, 
   current_size/1024/1024 "CURRENT_SIZE", 
   min_size/1024/1024 "MIN_SIZE",
   user_specified_size/1024/1024 "USER_SPECIFIED_SIZE", 
   last_oper_type "TYPE" 
from 
   v$sga_dynamic_components;
 
select * from V$MEMORY_TARGET_ADVICE;
select * from V$PGA_TARGET_ADVICE ;


prompt 
prompt ===========================================================
prompt PROCESSOS
prompt ===========================================================
prompt 
set numwidth 12
select * from V$RESOURCE_LIMIT order by 1;


prompt 
prompt ===========================================================
prompt FEATURES UTILIZADAS
prompt ===========================================================
prompt 
select	name feature
,	detected_usages
from	dba_feature_usage_statistics
where 	detected_usages > 0;


prompt 
prompt ===========================================================
prompt VOLUMETRIA FISICA 
prompt ===========================================================
prompt 
set pagesize 10000
select sum(bytes)/1024/1024 as "Tamanho (MB)" from dba_data_files;

prompt 
prompt ===========================================================
prompt VOLUMETRIA LOGICA 
prompt ===========================================================
prompt 
select sum(bytes)/1024/1024 as "Tamanho (MB)" from dba_segments;

prompt 
prompt ===========================================================
prompt RESUMO DE OCUPACAO DE ESPACO POR ESQUEMA
prompt ===========================================================
prompt 
select owner as "ESQUEMA", segment_type as "TIPO DE OBJETO", sum(bytes)/1024/1024 as "TAMANHO (MB)"
from dba_segments
group by owner, segment_type
order by owner;

prompt 
prompt ===========================================================
prompt TABLESPACES
prompt ===========================================================
prompt 
select tablespace_name as "TABLESPACE", block_size as "BLOCO DE DADOS", status as "STATUS", logging as "LOGGING",
extent_management as "GER.EXTENSAO", allocation_type as "TIPO DE ALOCACAO", segment_space_management as "GER.SEGMENTO",
retention as "RETENCAO"
from dba_tablespaces;

prompt 
prompt ===========================================================
prompt RESUMO POR TABLESPACES
prompt ===========================================================
prompt 
select decode(grouping(tablespace_name),0,null,1,'TOTAL (MB) =') Total,
tablespace_name as "TABLESPACE", segment_type as "TIPO DE OBJETO", sum(bytes)/1024/1024 as "TAMANHO (MB)"
from dba_segments
group by rollup(tablespace_name, segment_type)
order by tablespace_name;

prompt 
prompt ===========================================================
prompt COMANDOS DE CRIAÇAO TABLESPACES
prompt ===========================================================
prompt 

declare 
  iSql VARCHAR2( 1200 ) ;
begin
  for cur in (select tablespace_name from dba_tablespaces) loop
  
    select dbms_metadata.get_ddl( 'TABLESPACE', cur.tablespace_name) into iSql from dual ;
  
    dbms_output.put_line( 'Tablespace : ' || cur.tablespace_name || chr(13) || chr(10) || iSql ) ;
  
  end loop;
end;
/


prompt ============================================================
prompt ARQUIVOS DO BANCO DE DADOS
prompt ============================================================

select name from v$datafile
union
select name from v$tempfile
union
select member from v$logfile
union
select name from v$controlfile
union
select filename from v$block_change_tracking
union
select name from v$flashback_database_logfile;
/

prompt 
prompt ===========================================================
prompt REDO LOGS
prompt ===========================================================
prompt 
col member	format a60
col "Size MB"	format 9,999,999
select	lf.group#, lf.member, ceil(lg.bytes / 1024 / 1024) MBytes
 from	v$logfile lf ,	v$log lg
where	lg.group# = lf.group#
order	by 1;


prompt 
prompt ===========================================================
prompt UNDO SEGMENTS
prompt ===========================================================
prompt 
select	segment_name,	status, tablespace_name
from	dba_rollback_segs;

prompt 
prompt ===========================================================
prompt NLS DATABASE PARAMETERS 
prompt ===========================================================
prompt 
select * from nls_database_parameters ;


prompt 
prompt ===========================================================
prompt ASM DISKGROUPS
prompt ===========================================================
prompt 
set lines 100
col name	format a10
col path 	format a30
select	name
,	group_number
,	disk_number
,	mount_status
,	state
,	path
from 	v$asm_disk
order 	by group_number;

prompt 
prompt ===========================================================
prompt PROFILES
prompt ===========================================================
prompt 
select * from dba_profiles ;


prompt 
prompt ===========================================================
prompt JOBS (DBA_JOBS)
prompt ===========================================================
prompt 
set lines 100 pages 999
col	schema_user format a15
col	fails format 999
select	job
,	schema_user
,	to_char(last_date, 'hh24:mi dd/mm/yy') last_run
,	to_char(next_date, 'hh24:mi dd/mm/yy') next_run
,	failures fails
,	broken
,	substr(what, 1, 45) what
from	dba_jobs
order by 4;


prompt 
prompt ===========================================================
prompt JOBS (DBA_SCHEDULER_JOBS)
prompt ===========================================================
prompt 
select owner, count(1) qtde 
 from dba_scheduler_jobs
group by owner;


prompt 
prompt ===========================================================
prompt DIRETORIOS
prompt ===========================================================
prompt 
col directory_name format a30
col directory_path format a60
select * from dba_directories;


prompt 
prompt ===========================================================
prompt TIPOS DE OBJETOS
prompt ===========================================================
prompt 
select owner, object_type, count(1) qtde
  from dba_objects
  where owner not in ('SYS', 'SYSTEM')
group by owner, object_type
order by owner, 3 desc;


prompt 
prompt ===========================================================
prompt OBJETOS INVALIDOS
prompt ===========================================================
prompt 
set lines 200 pages 999
col "obj" format a40
select owner || '.' || object_name "obj",  
object_type
from dba_objects
where status = 'INVALID';


prompt 
prompt ===========================================================
prompt ROLES
prompt ===========================================================
prompt
select * from dba_roles ;

prompt 
prompt ===========================================================
prompt USUÁRIOS
prompt ===========================================================
prompt 

set pages 999 lines 100
col username	format a20
col status	format a8
col tablespace	format a20
col temp_ts	format a20
select	username
,	account_status status
,	created
,	default_tablespace
,	temporary_tablespace temp_ts
from	dba_users
order	by username
/ 
prompt 
prompt ===========================================================
prompt COMANDOS DE CRIAÇÃO / GRANTS DE USUÁRIOS
prompt ===========================================================
prompt

set line 250

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
    dbms_output.put_line( ' **** Não há roles' || chr(13) || chr(10)) ;   
   end ;
   
   begin
    select DBMS_METADATA.GET_GRANTED_DDL('SYSTEM_GRANT', cur.username) into iSql from dual;
    dbms_output.put_line( 'System Grants: ' || iSql || chr(13) || chr(10)) ; 
   exception 
   when others then
    dbms_output.put_line( ' **** Não há System Grants '  || chr(13) || chr(10)) ;  
   end;
   
   begin
    select DBMS_METADATA.GET_GRANTED_DDL('OBJECT_GRANT', cur.username) into iSql from dual;
    dbms_output.put_line( 'Objects Grants: ' || iSql ) ; 
  exception 
   when others then
    dbms_output.put_line( ' **** Não há Object Grants '  || chr(13) || chr(10)) ;    
   end;
   
  end loop;
  
end;
/


spool off

exit

```
