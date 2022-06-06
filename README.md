# Oracle_Query
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
