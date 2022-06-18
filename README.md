# Oracle_Query
==== https://dbaclass.com/monitor-your-db/ ======
-----------------sq.sql-----------------
set echo off
set serveroutput on size 50000
set verify off

set feedback off
accept inst_id prompt 'Enter instance id: '
accept SID prompt 'Enter sid: '
DECLARE
  v_sid number;
  vs_cnt number;
  s sys.gv_$session%ROWTYPE;
  p sys.gv_$process%ROWTYPE;
  cursor cur_c1 is select sid from   sys.gv_$process p,sys.gv_$session s  where p.addr  = s.paddr and   sid = &SID  and s.inst_id='&inst_id' and p.inst_id=s.inst_id;
BEGIN
    dbms_output.put_line('=====================================================================');
    select nvl(count(sid),0) into vs_cnt from sys.gv_$process p, sys.gv_$session s  where  p.addr  = s.paddr and   sid = &SID and s.inst_id='&inst_id' and p.inst_id=s.inst_id;
    open cur_c1;
    LOOP
      FETCH cur_c1 INTO v_sid;
        EXIT WHEN (cur_c1%NOTFOUND);
        select * into s from sys.gv_$session where sid  = v_sid and inst_id='&inst_id';
          select * into p from sys.gv_$process where addr = s.paddr and inst_id='&inst_id';
        dbms_output.put_line('INST_ID  : '|| s.inst_id);
        dbms_output.put_line('SID/Serial  : '|| s.sid||','||s.serial#);
          dbms_output.put_line('Foreground  : '|| 'PID: '||s.process||' - '||s.program);
          dbms_output.put_line('Shadow      : '|| 'PID: '||p.spid||' - '||p.program);
          dbms_output.put_line('Terminal    : '|| s.terminal || '/ ' || p.terminal);
          dbms_output.put_line('OS User     : '|| s.osuser||' on '||s.machine);
          dbms_output.put_line('Ora User    : '|| s.username);
        dbms_output.put_line('Details     : '|| s.action||' - '||s.module);
          dbms_output.put_line('Status Flags: '|| s.status||' '||s.server||' '||s.type);
          dbms_output.put_line('Tran Active : '|| nvl(s.taddr, 'NONE'));
          dbms_output.put_line('Login Time  : '|| to_char(s.logon_time, 'Dy HH24:MI:SS'));
          dbms_output.put_line('Last Call   : '|| to_char(sysdate-(s.last_call_et/60/60/24), 'Dy HH24:MI:SS') || ' - ' || to_char(s.last_call_et/60, '9990.0') || ' min');
          dbms_output.put_line('Lock/ Latch : '|| nvl(s.lockwait, 'NONE')||'/ '||nvl(p.latchwait, 'NONE'));
          dbms_output.put_line('Latch Spin  : '|| nvl(p.latchspin, 'NONE'));
          dbms_output.put_line('Current SQL statement:');
        for c1 in ( select * from sys.gv_$sqltext  where HASH_VALUE = s.sql_hash_value and inst_id='&inst_id' order by piece)
        loop
            dbms_output.put_line(chr(9)||c1.sql_text);
          end loop;
        dbms_output.put_line('Previous SQL statement:');
          for c1 in ( select * from sys.gv_$sqltext  where HASH_VALUE = s.prev_hash_value and inst_id='&inst_id' order by piece)
        loop
            dbms_output.put_line(chr(9)||c1.sql_text);
          end loop;
        dbms_output.put_line('Session Waits:');
          for c1 in ( select * from sys.gv_$session_wait where sid = s.sid and inst_id='&inst_id')
        loop
        dbms_output.put_line(chr(9)||c1.state||': '||c1.event);
          end loop;
  dbms_output.put_line('Connect Info:');
  for c1 in ( select * from sys.gv_$session_connect_info where sid = s.sid and inst_id='&inst_id') loop
    dbms_output.put_line(chr(9)||': '||c1.network_service_banner);
  end loop;
          dbms_output.put_line('Locks:');
          for c1 in ( select  /*+ RULE */ decode(l.type,
          -- Long locks
                      'TM', 'DML/DATA ENQ',   'TX', 'TRANSAC ENQ',
                      'UL', 'PLS USR LOCK',
          -- Short locks
                      'BL', 'BUF HASH TBL',  'CF', 'CONTROL FILE',
                      'CI', 'CROSS INST F',  'DF', 'DATA FILE   ',
                      'CU', 'CURSOR BIND ',
                      'DL', 'DIRECT LOAD ',  'DM', 'MOUNT/STRTUP',
                      'DR', 'RECO LOCK   ',  'DX', 'DISTRIB TRAN',
                      'FS', 'FILE SET    ',  'IN', 'INSTANCE NUM',
                      'FI', 'SGA OPN FILE',
                      'IR', 'INSTCE RECVR',  'IS', 'GET STATE   ',
                      'IV', 'LIBCACHE INV',  'KK', 'LOG SW KICK ',
                      'LS', 'LOG SWITCH  ',
                      'MM', 'MOUNT DEF   ',  'MR', 'MEDIA RECVRY',
                      'PF', 'PWFILE ENQ  ',  'PR', 'PROCESS STRT',
                      'RT', 'REDO THREAD ',  'SC', 'SCN ENQ     ',
                      'RW', 'ROW WAIT    ',
                      'SM', 'SMON LOCK   ',  'SN', 'SEQNO INSTCE',
                      'SQ', 'SEQNO ENQ   ',  'ST', 'SPACE TRANSC',
                      'SV', 'SEQNO VALUE ',  'TA', 'GENERIC ENQ ',
                      'TD', 'DLL ENQ     ',  'TE', 'EXTEND SEG  ',
                      'TS', 'TEMP SEGMENT',  'TT', 'TEMP TABLE  ',
                      'UN', 'USER NAME   ',  'WL', 'WRITE REDO  ',
                      'TYPE='||l.type) type,
                         decode(l.lmode, 0, 'NONE', 1, 'NULL', 2, 'RS', 3, 'RX',
                       4, 'S',    5, 'RSX',  6, 'X',
                       to_char(l.lmode) ) lmode,
                          decode(l.request, 0, 'NONE', 1, 'NULL', 2, 'RS', 3, 'RX',
                         4, 'S', 5, 'RSX', 6, 'X',
                         to_char(l.request) ) lrequest,
                           decode(l.type, 'MR', o.name,
                      'TD', o.name,
                      'TM', o.name,
                      'RW', 'FILE#='||substr(l.id1,1,3)||
                            ' BLOCK#='||substr(l.id1,4,5)||' ROW='||l.id2,
                      'TX', 'RS+SLOT#'||l.id1||' WRP#'||l.id2,
                      'WL', 'REDO LOG FILE#='||l.id1,
                      'RT', 'THREAD='||l.id1,
                      'TS', decode(l.id2, 0, 'ENQUEUE', 'NEW BLOCK ALLOCATION'),
                      'ID1='||l.id1||' ID2='||l.id2) objname
                       from  sys.gv_$lock l, sys.obj$ o
                       where sid   = s.sid
                                and inst_id='&inst_id'
                         and l.id1 = o.obj#(+) )
            loop
             dbms_output.put_line(chr(9)||c1.type||' H: '||c1.lmode||' R: '||c1.lrequest||' - '||c1.objname);
              end loop;
            dbms_output.put_line('=====================================================================');
    END LOOP;

      close cur_c1;
exception
    when no_data_found then
      dbms_output.put_line('Unable to find process id &&SID for the instance number '||'&inst_id'||' !!!');
      dbms_output.put_line('=====================================================================');
      return;
    when others then
      dbms_output.put_line(sqlerrm);
      return;
END;
/
undef SID
set heading on
set verify on
set feedback on
-------------------------- END -----------------
---------------wait.sql--------------
set lines 200
col event format a40
set pages 1000
set colsep |
set underline _
break on sql_id   skip page
select event,count(*) from gv$session_wait group by event order by 2
/

col sql format a40
col state format a12
col username format a15

select distinct s.inst_id,w.sid,s.username,s.sql_id,w.event,substr(w.state,1,12) state,
substr(q.sql_text,1,15)||'.....'||substr(q.sql_text,instr(q.sql_text,'FROM',1),20) sql,round(s.last_call_et/60) MINS_ACTIVE
from gv$session_wait w,gv$session s,v$sql q
where w.event like '%&event%'
and w.sid=s.sid
and s.SQL_HASH_VALUE=q.HASH_VALUE
and s.status='ACTIVE'
and s.inst_id=w.inst_id
and s.username is not null
and s.last_call_et>0
order by sql_id, username
/
---------------END------------------
-------------- aue.sql ----------------------
Set echo off
set pagesize 50
set lines 200
set verify off
set heading on
set feedback on
col SESS format a12
col status format a10
col program format a30
col terminal format a12
col "Machine Name" format a15
col "Machine Name" format a15
col "DB User" format a14
col "Logon Time" format a14
col "OS User" format a10


select inst_id, rpad(s.username,14,' ') as "DB User",
   to_char(logon_time,'hh24:mi Mon/dd') as "Logon Time",
   initcap(status) as "Status",s.sid||','||s.serial# SESS,
   s.event,
   rpad(upper(substr(s.program,instr(s.program,'\',-1)+1)),30,' ') as "Program",
   rpad(initcap(machine),15,' ') as "Machine Name" from gv$session s
   where upper(s.username) like upper('%&Username%') and s.status='ACTIVE'
   order by machine,s.program
/

-------------- END---------------
---------- sqh.sql (SQL HISTORY) ----------
set lines 250 pages 5000
col NOde for 99
col BEG_time for a15
col module for a43
col parsing_schema_name for a12 heading user
col instance_number for 99
--col SQL_PROFILE noprint
col SQL_PROFILE for a10
--break on Beg_time skip page
set colsep |
set underline _
col snap_id noprint
select ss.snap_id, ss.instance_number node, PARSING_SCHEMA_NAME, to_char(begin_interval_time,'dd-MON-yy hh24:mi') BEG_time, sql_id, plan_hash_value,module,SQL_PROFILE,
nvl(executions_delta,0) execs,
(elapsed_time_delta/decode(nvl(executions_delta,0),0,1,executions_delta))/1000000 avg_etime_s,
(buffer_gets_delta/decode(nvl(buffer_gets_delta,0),0,1,executions_delta)) avg_lio
from DBA_HIST_SQLSTAT S, DBA_HIST_SNAPSHOT SS
where
sql_id = nvl('&sql_id','bdqf2qwjvuupd') and ss.snap_id = S.snap_id
and ss.instance_number = S.instance_number and executions_delta > 0
and executions_delta>0
and begin_interval_time > trunc(sysdate-&Days)
order by snap_id, PARSING_SCHEMA_NAME,ss.instance_number
/

----------END----------

----------sql.monitoring-----------
col key format 999999999999
col sql_exec_start for a25
col sql_text for a60 trunc
break on sql_id on sql_text
set colsep '|'
break on sql_id on plan_hash_value
col sql_exec_start for a20
select sid, sql_id, sql_exec_id, to_char(sql_exec_start,'DD-Mon-YY HH24:MI:SS') sql_exec_start, sql_plan_hash_value plan_hash_value,
elapsed_time/1000000 etime, buffer_gets, disk_reads
from v$sql_monitor
where sid like nvl('&sid',sid)
and sql_id like nvl('&sql_id',sql_id)
and sql_exec_id like nvl('&sql_exec_id',sql_exec_id)
order by sql_id, sql_exec_id
/
set colsep ' '

-------- END----------

--------- az.sql -------------
spool results01.txt
 set echo on feedback on time on timing on pagesize 100 linesize 80 numwidth 13
 show user
 alter session set nls_date_format = 'DD-MON-YYYY HH24:MI:SS';
 select * from v$version;
 select to_char(sysdate, 'DD-MON-YYYY HH24:MI:SS') as current_date from dual;
 column name format a30
 column value format a49
 select name, value from v$parameter where isdefault='FALSE' order by 1;
 column parameter format a30
 column value format a49
 select * from v$nls_parameters order by parameter;
 column name format a10
 select dbid, name,
 to_char(created, 'DD-MON-YYYY HH24:MI:SS') created,
 open_mode, log_mode,
 to_char(checkpoint_change#, '999999999999999') as checkpoint_change#,
 controlfile_type,
 to_char(controlfile_change#, '999999999999999') as controlfile_change#,
 to_char(controlfile_time, 'DD-MON-YYYY HH24:MI:SS') controlfile_time,
 to_char(resetlogs_change#, '999999999999999') as resetlogs_change#,
 to_char(resetlogs_time, 'DD-MON-YYYY HH24:MI:SS') resetlogs_time
 from v$database;
 select * from v$instance;
 archive log list;
 select * from v$thread order by thread#;
 select * from v$log order by first_change#;
 column member format a60
set lines 200
 select * from v$logfile;
 column name format a79
 select '#' || ts.name || '#' as tablespace_name, ts.ts#,
 '#' || df.name || '#' as filename, df.file#, df.status, df.enabled, df.creation_change#,
 to_char(df.creation_time, 'DD-MON-YYYY HH24:MI:SS') as creation_time,
 to_char(df.checkpoint_change#, '999999999999999') as checkpoint_change#,
 to_char(df.checkpoint_time, 'DD-MON-YYYY HH24:MI:SS') as checkpoint_time,
 to_char(df.offline_change#, '999999999999999') as offline_change#,
 to_char(df.online_change#, '999999999999999') as online_change#,
 to_char(df.online_time, 'DD-MON-YYYY HH24:MI:SS') as online_time,
 to_char(df.unrecoverable_change#, '999999999999999') as online_change#,
 to_char(df.unrecoverable_time, 'DD-MON-YYYY HH24:MI:SS') as online_time,
 to_char(df.bytes, '9,999,999,999,990') as bytes, block_size
 from v$datafile df, v$tablespace ts
 where ts.ts# = df.ts#
 and ( df.status <> 'ONLINE'
 or df.checkpoint_change# <> (select checkpoint_change# from v$database) );
 select '#' || ts.name || '#' as tablespace_name, ts.ts#,
 '#' || dh.name || '#' as filename, dh.file#, dh.status, dh.error, dh.
 fuzzy, dh.creation_change#,
 to_char(dh.creation_time, 'DD-MON-YYYY HH24:MI:SS') as creation_time,
 to_char(dh.checkpoint_change#, '999999999999999') as checkpoint_change#,
 to_char(dh.checkpoint_time, 'DD-MON-YYYY HH24:MI:SS') as checkpoint_time,
 to_char(dh.resetlogs_change#, '999999999999999') as resetlogs_change#,
 to_char(dh.resetlogs_time, 'DD-MON-YYYY HH24:MI:SS') as resetlogs_time,
 to_char(dh.bytes, '9,999,999,999,990') as bytes
 from v$datafile_header dh, v$tablespace ts
 where ts.ts# = dh.ts#
 and ( dh.status <> 'ONLINE'
 or dh.checkpoint_change# <> (select checkpoint_change# from v$database) );
 select * from v$tempfile;
 select HXFIL File_num,substr(HXFNM,1,60) file_name, FHTNM tablespace_name,
 FHTYP type, HXERR validity,
 FHSCN SCN, FHTIM SCN_Time, FHSTA status,
 FHTHR Thread, FHRBA_SEQ Sequence
 from X$KCVFH
 --where HXERR > 0
 order by HXERR, FHSTA, FHSCN, HXFIL;
 select FHTHR Thread, FHRBA_SEQ Sequence, count(1)
 from X$KCVFH
 group by FHTHR, FHRBA_SEQ
 order by FHTHR, FHRBA_SEQ;
 column error format a15
 select error, fuzzy, status, checkpoint_change#,
 to_char(checkpoint_time, 'DD-MON-YYYY HH24:MI:SS') as checkpoint_time,
 count(*)
 from v$datafile_header
 group by error, fuzzy, status, checkpoint_change#, checkpoint_time
 order by checkpoint_change#, checkpoint_time;
 select * from V$INSTANCE_RECOVERY;
 select * from v$recover_file order by change#;
 select * from dba_tablespaces where status <> 'ONLINE';
 SELECT * FROM database_properties order by property_name;
 select *
 from X$KCCLH, (select min(checkpoint_change#) df_min_scn,
 min(checkpoint_change#) df_max_scn
 from v$datafile_header
 where status='ONLINE') df
 where LHLOS in (select first_change# from v$log)
 or df.df_min_scn between LHLOS and LHNXS
 or df.df_max_scn between LHLOS and LHNXS;
 select * from v$backup where status <> 'NOT ACTIVE';
 select ADDR, XIDUSN, XIDSLOT, XIDSQN,
 UBAFIL, UBABLK, UBASQN,
 START_UBAFIL, START_UBABLK, START_UBASQN,
 USED_UBLK, STATUS
 from v$transaction;
 select * from v$archive_gap;
 select * from v$archive_dest_status where recovery_mode <> 'IDLE';
 column USED_GB format 999,990.999
 column USED% format 990.99
 column RECLAIM_GB format 999,990.999
 column RECLAIMABLE% format 990.99
 column LIMIT_GB format 999,990.999
 select frau.file_type as type,
 frau.percent_space_used/100 * rfd.space_limit /1024/1024/1024 "USED_GB",
 frau.percent_space_used "USED%",
 frau.percent_space_reclaimable "RECLAIMABLE%",
 frau.percent_space_reclaimable/100 * rfd.space_limit /1024/1024/1024 "RECLAIM_GB",
 frau.number_of_files "FILES#"
 from v$flash_recovery_area_usage frau,
 v$recovery_file_dest rfd
 order by file_type;
 select name,
 space_limit/1024/1024/1024 "LIMIT_GB",
 space_used/1024/1024/1024 "USED_GB",
 space_used/space_limit*100 "USED%",
 space_reclaimable/1024/1024/1024 "RECLAIM_GB",
 number_of_files "FILE#"
 from v$recovery_file_dest;
 select * from v$backup_corruption;
 select * from v$copy_corruption order by file#, block#;
 select * from v$database_block_corruption order by file#, block#;
 SELECT f.file#, f.name,
 e.tablespace_name, e.segment_type, e.owner, e.segment_name,
 c.file#, c.block#, c.blocks, c.corruption_change#, c.corruption_type
 FROM dba_extents e, V$database_block_corruption c, v$datafile f
 WHERE c.file# = f.file#
 and e.file_id = c.file#
 and c.block# between e.block_id AND e.block_id + e.blocks - 1;
 select * from v$database_incarnation;
 select * from v$rman_configuration;
 select s.recid as bs_key, p.recid as bp_key, p.status, p.tag, p.device_type,
 p.handle, p.media, p.completion_time, p.bytes
 from v$backup_piece p, v$backup_set s
 where p.set_stamp = s.set_stamp
 and s.controlfile_included='YES'
 order by p.completion_time;
 select * from v$block_change_tracking;
 select p.completion_time, 'File*' || f.file# || '*', s.recid as bs_key, p.recid as bp_key, p.status, p.tag, p.device_type,
 p.handle, p.media, f.absolute_fuzzy_change#, p.bytes,
 f.incremental_level, f. block_size, f.datafile_blocks, f.blocks_read, f.blocks,
 round((f.blocks_read/f.datafile_blocks) * 100,2) "%READ",
 round((f.blocks/f.datafile_blocks)*100,2) "%WRTN",
 f.used_change_tracking, f.creation_change#, f.incremental_change#, f.checkpoint_change#
 from v$backup_datafile f, v$backup_piece p, v$backup_set s
 where p.set_stamp = s.set_stamp
 and f.set_stamp = s.set_stamp
 and p.handle is not null
 --and f.file# = 1
 and f.file# > 0
 order by p.completion_time, f.file#;
 select 'File*' || file# || '*', completion_time, to_char(creation_change#), incremental_level, to_char(incremental_change#) inc#,
 to_char(checkpoint_change#) ckp#, datafile_blocks BLKS, block_size blksz,
 blocks_read READ, round((blocks_read/datafile_blocks) * 100,2) "%READ",
 blocks WRTN, round((blocks/datafile_blocks)*100,2) "%WRTN", used_change_tracking
 from v$backup_datafile
 where completion_time > sysdate-7
 and file# > 0
 order by file#, completion_time;
 COL rman_start_time FORM A20
 COL rman_end_time FORM A20
 COL time_taken_display FORM A10 HEAD "Time|Taken|HH:MM:SS"
 COL i_size_gig FORM 990.99 HEAD "Input|Gig"
 COL o_size_gig FORM 990.99 HEAD "Output|Gig"
 COL compression_ratio FORM 90.99 HEAD "Comp.|Ratio"
 COL status FORM A12
 COL input_type FORM A14
 COL INPUT_BYTES_PER_SEC_DISPLAY FORM A9 HEAD "Input|(Bytes/s)"
 COL OUTPUT_BYTES_PER_SEC_DISPLAY FORM A9 HEAD "Output|(Bytes/s)"
 --
 SELECT
 command_id
 ,TO_CHAR(start_time,'DD-MON-RRRR HH24:MI:SS') AS rman_start_time
 ,TO_CHAR(end_time,'DD-MON-RRRR HH24:MI:SS') AS rman_end_time
 ,time_taken_display
 ,input_bytes/1024/1024/1024 i_size_gig
 ,output_bytes/1024/1024/1024 o_size_gig
 ,compression_ratio
 ,status
 ,input_type
 ,input_bytes, output_bytes
 ,INPUT_BYTES_PER_SEC_DISPLAY
 ,OUTPUT_BYTES_PER_SEC_DISPLAY
 FROM v$rman_backup_job_details
 ORDER BY end_time;
 select * from v$filestat;
 column EBS_MB format 9,990.99
 column TOTAL_MB format 999,990.99
 select SID, SERIAL, FILENAME, EFFECTIVE_BYTES_PER_SECOND/1024/1024 as EBS_MB,
 OPEN_TIME, CLOSE_TIME, ELAPSED_TIME, TOTAL_BYTES/1024/1024 as TOTAL_MB,
 STATUS, MAXOPENFILES, buffer_size, buffer_count
 from v$backup_async_io
 where close_time >= sysdate-3
 order by close_time;
 select SID, SERIAL, FILENAME, EFFECTIVE_BYTES_PER_SECOND/1024/1024 as EBS_MB,
 OPEN_TIME, CLOSE_TIME, ELAPSED_TIME, TOTAL_BYTES/1024/1024 as TOTAL_MB,
 STATUS, MAXOPENFILES, buffer_size, buffer_count
 from v$backup_sync_io
 where close_time >= sysdate-3;
 select * from v$controlfile_record_section order by type;
 select to_char(rownum) || '. ' || output rman_output from v$rman_output;
 select * from v$rman_status where trunc(end_time) > trunc(sysdate)-3;
 select protection_mode, protection_level from v$database;
 select * from v$recovery_progress;
 SELECT
 a.username
 ,a.sid || ',' || a.serial# AS kill_string
 , b.spid AS OS_ID
 ,(CASE WHEN a.client_info IS NULL AND a.action IS NOT NULL THEN 'First Default'
 WHEN a.client_info IS NULL AND a.action IS NULL THEN 'Polling'
 ELSE a.client_info
 END) client_info
 ,a.action
 FROM v$session a
 ,v$process b
 WHERE a.program like '%rman%'
 AND a.paddr = b.addr;
 select s.client_info,
 sl.message,
 sl.sid, sl.serial#, p.spid,
 sl.start_time, sl.last_update_time,
 sl.time_remaining, sl.elapsed_seconds, sl.time_remaining + sl.elapsed_seconds as total_seconds,
 sl.time_remaining/60 "Remaining (Min)", sl.elapsed_seconds/60 "Elapsed (Min)",
 round(sl.sofar/sl.totalwork*100,2) || '%' "Completed"
 from v$session_longops sl, v$session s, v$process p
 where p.addr = s.paddr
 and sl.sid=s.sid
 and sl.serial#=s.serial#
 and opname LIKE 'RMAN%'
 and opname NOT LIKE '%aggregate%'
 and totalwork != 0
 and sofar <> totalwork;
 select AL.*,
 DF.min_checkpoint_change#, DF.min_checkpoint_time
 from v$archived_log AL,
 (select min(checkpoint_change#) min_checkpoint_change#,
 min(checkpoint_time) min_checkpoint_time
 from v$datafile_header
 where status='ONLINE') DF
 where DF.min_checkpoint_change# between AL.first_change# and AL.next_change#
 order by AL.first_change#;
 select thread#, min(first_time), max(next_time), min(first_change#), max(next_change#),
 min(sequence#) as min_seq, max(sequence#) as max_seq, count(1)
 from v$archived_log
 where status = 'A'
 group by thread#;
 select * from v$asm_diskgroup;
 select * from v$asm_disk;
 select * from v$flashback_database_stat order by begin_time desc;
 select * from v$flashback_database_log;
 select * from v$flashback_database_logfile order by first_time desc;
 select * from v$restore_point;
 select * from v$rollname;
 select * from v$undostat;
 select * from dba_rollback_segs;
 spool off

----------end ------------


-- SQL Profile/Baseline -----


--- creatre sql baseline from coursor cache ----
DECLARE

  l_plans_loaded  PLS_INTEGER;
BEGIN
  l_plans_loaded := DBMS_SPM.load_plans_from_cursor_cache(
    sql_id => '&sql_id');
END;
/

-- Create baseline with a particular hash value

DECLARE
  l_plans_loaded  PLS_INTEGER;
BEGIN
  l_plans_loaded := DBMS_SPM.load_plans_from_cursor_cache(
    sql_id => '&sql_id', plan_hash_value => '&plan_hash_value');
END;
/

----------drop a sql baseline -----

declare
drop_result pls_integer;
begin
drop_result := DBMS_SPM.DROP_SQL_PLAN_BASELINE(
plan_name => '&sql_plan_baseline_name');
dbms_output.put_line(drop_result);
end;
/

You can get the sql baseline from a sql_id from below command:

SELECT sql_handle, plan_name FROM dba_sql_plan_baselines WHERE signature IN
( SELECT exact_matching_signature FROM gv$sql WHERE sql_id='&SQL_ID');

--------------create baselines for all sqls of a schema---

DECLARE
nRet NUMBER;
BEGIN
nRet := dbms_spm.load_plans_from_cursor_cache(
attribute_name => 'PARSING_SCHEMA_NAME',
attribute_value => '&schema_name'
);
END;


---- drop sql_profile--

BEGIN
DBMS_SQLTUNE.drop_sql_profile (
name => '&sql_profile',
ignore => TRUE);
END;
/

You can get the respective sql_profile of a sql_id from below:

select distinct
p.name sql_profile_name,
s.sql_id
from
dba_sql_profiles p,
DBA_HIST_SQLSTAT s
where
p.name=s.sql_profile and s.sql_id='&sql_id';

------------------ Script for getting sql_profile created for a sql_id

select distinct
p.name sql_profile_name,
s.sql_id
from
dba_sql_profiles p,
DBA_HIST_SQLSTAT s
where
p.name=s.sql_profile and s.sql_id='&sql_id';

------- SQL Tunning Advisor ----

Create tuning task:

DECLARE
l_sql_tune_task_id VARCHAR2(100);
BEGIN
l_sql_tune_task_id := DBMS_SQLTUNE.create_tuning_task (
sql_id => '12xca9smf3hfy',
scope => DBMS_SQLTUNE.scope_comprehensive,
time_limit => 500,
task_name => '12xca9smf3hfy_tuning_task',
description => 'Tuning task1 for statement 12xca9smf3hfy');
DBMS_OUTPUT.put_line('l_sql_tune_task_id: ' || l_sql_tune_task_id);
END;
/

Execute tuning task:

EXEC DBMS_SQLTUNE.execute_tuning_task(task_name => '12xca9smf3hfy_tuning_task');

Get the tuning advisory report

set long 65536
set longchunksize 65536
set linesize 100
select dbms_sqltune.report_tuning_task('12xca9smf3hfy_tuning_task') from dual;

------ DISABLE/ENABLE SQL profile ---

EXEC DBMS_SQLTUNE.ALTER_SQL_PROFILE('&sql_profile_name','STATUS','DISABLED');

----Pass the sql_id to get the respective sql baseline-----

SELECT sql_handle, plan_name FROM dba_sql_plan_baselines WHERE
signature IN ( SELECT exact_matching_signature FROM gv$sql WHERE sql_id='&SQL_ID')

------------ To disable a baseline:----------

Begin
dbms_spm.alter_sql_plan_baseline(sql_handle =>'SQL_SQL_5818768f40d7be2a',
plan_name => 'SQL_PLAN_aaxsg8yktm4h100404251',
attribute_name=> 'enabled',
attribute_value=>'NO');
END;
/
Begin
dbms_spm.alter_sql_plan_baseline(sql_handle =>'SQL_SQL_5818768f40d7be2a',
plan_name => 'SQL_PLAN_aaxsg8yktm4h100404251',
attribute_name=> 'fixed',
attribute_value=>'NO');
END;
/

-- To enable again, just modify the attribute_value to YES,-----

================ Table sapce details with Autoextent on details =============

set feedback off
set pagesize 70;
set linesize 2000
set head on
COLUMN Tablespace format a25 heading 'Tablespace Name'
COLUMN autoextensible format a11 heading 'AutoExtend'
COLUMN files_in_tablespace format 999 heading 'Files'
COLUMN total_tablespace_space format 99999999 heading 'TotalSpace'
COLUMN total_used_space format 99999999 heading 'UsedSpace'
COLUMN total_tablespace_free_space format 99999999 heading 'FreeSpace'
COLUMN total_used_pct format 9999 heading '%Used'
COLUMN total_free_pct format 9999 heading '%Free'
COLUMN max_size_of_tablespace format 99999999 heading 'ExtendUpto'
COLUM total_auto_used_pct format 999.99 heading 'Max%Used'
COLUMN total_auto_free_pct format 999.99 heading 'Max%Free'
WITH tbs_auto AS
(SELECT DISTINCT tablespace_name, autoextensible
FROM dba_data_files
WHERE autoextensible = 'YES'),
files AS
(SELECT tablespace_name, COUNT (*) tbs_files,
SUM (BYTES/1024/1024) total_tbs_bytes
FROM dba_data_files
GROUP BY tablespace_name),
fragments AS
(SELECT tablespace_name, COUNT (*) tbs_fragments,
SUM (BYTES)/1024/1024 total_tbs_free_bytes,
MAX (BYTES)/1024/1024 max_free_chunk_bytes
FROM dba_free_space
GROUP BY tablespace_name),
AUTOEXTEND AS
(SELECT tablespace_name, SUM (size_to_grow) total_growth_tbs
FROM (SELECT tablespace_name, SUM (maxbytes)/1024/1024 size_to_grow
FROM dba_data_files
WHERE autoextensible = 'YES'
GROUP BY tablespace_name
UNION
SELECT tablespace_name, SUM (BYTES)/1024/1024 size_to_grow
FROM dba_data_files
WHERE autoextensible = 'NO'
GROUP BY tablespace_name)
GROUP BY tablespace_name)
SELECT c.instance_name,a.tablespace_name Tablespace,
CASE tbs_auto.autoextensible
WHEN 'YES'
THEN 'YES'
ELSE 'NO'
END AS autoextensible,
files.tbs_files files_in_tablespace,
files.total_tbs_bytes total_tablespace_space,
(files.total_tbs_bytes - fragments.total_tbs_free_bytes
) total_used_space,
fragments.total_tbs_free_bytes total_tablespace_free_space,
round(( ( (files.total_tbs_bytes - fragments.total_tbs_free_bytes)
/ files.total_tbs_bytes
)
* 100
)) total_used_pct,
round(((fragments.total_tbs_free_bytes / files.total_tbs_bytes) * 100
)) total_free_pct
FROM dba_tablespaces a,v$instance c , files, fragments, AUTOEXTEND, tbs_auto
WHERE a.tablespace_name = files.tablespace_name
AND a.tablespace_name = fragments.tablespace_name
AND a.tablespace_name = AUTOEXTEND.tablespace_name
AND a.tablespace_name = tbs_auto.tablespace_name(+)
order by total_free_pct;

================== end===================
