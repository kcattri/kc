**### Source : 19c non-cdb Target :  19c PDB (single)**
**On Source**
1. Check if RAT is enabled in database

```sql
select value from v$option where parameter = 'Real Application Testing';  --TRUE
select object_name, object_type, status from dba_objects where object_name like '%WORKLOAD_CAPTURE'; --3 VALID
```

2. sqlplus system 
```sqlplus
create or replace directory dbworkload as '/ora_bin/rat2';
```
3. Start Load capturing
```sqlplus
exec dbms_workload_capture.start_capture(name=>'W02',dir=>'DBWORKLOAD');
```
4. monitor the progress
```sqplus 
Select id, name, status from dba_workload_captures;
```
5.  Ask users to run the load
6. Once load is done. stop capturing load 
```sqlplus 
exec dbms_workload_capture.finish_capture();
```
7. export AWR also 
```sqlplus
BEGIN 
DBMS_WORKLOAD_CAPTURE.EXPORT_AWR (capture_id =>1);
END;
/
```

**On Target**
1. Connect sqlplus system --connect to cdb
2. Create Directory
```sqlplus
create or replace directory dbworkload as '/ora_bin/rat2';
```
3. Create restore point to redo things if not satisfied
```sqlplus
CREATE RESTORE POINT RAT GUARANTEE FLASHBACK DATABASE; --If needed for rollback
```
4. configure replay
```sqlplus
exec dbms_workload_replay.process_capture('DBWORKLOAD');
```
5.  Initialize  replay
```sqlpus
exec dbms_workload_replay.initialize_replay('PDB_REPLAY','DBWORKLOAD');
```
6. review mapping 
```sqlplus
select conn_id,capture_conn,replay_conn  from dba_workload_connection_map;
```
7.  remap connection to pdb
```sqlplus
exec DBMS_WORKLOAD_REPLAY.REMAP_CONNECTION (connection_id => 2,replay_connection =>'(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=rac-kc-scan)(PORT=1521))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=test.com)))');
```
8. Again verify mapping
```sqlplus
select replay_id,conn_id,capture_conn,replay_conn from dba_workload_connection_map;
```
9.  Prepare  replay
```sqlplus
exec dbms_workload_replay.prepare_replay(synchronization=>FALSE); --use true if want to commit serialize, source db should be reachable for true
```
10. Deploy and start wrc
```sqlplus
wrc system/oracle12@trat mode=replay replaydir=/ora_bin/rat2
```
11. Start replay
```sqlplus
exec dbms_workload_replay.start_replay;
```
12. Incase need to cancel replay
 ```sqlplus
exec dbms_workload_replay.cancel_replay(); 
```
13. monitor replay
```sqlplus
select id, name, status from dba_workload_replays;  --once finished status will show completed
```
14.  Fetch RAT awr
```sqlplus
set echo off head off feedback off linesize 200 pagesize 1000
	set long 1000000 longchunksize 10000000
	VARIABLE rep_id number;
	BEGIN
   	SELECT max(id) INTO :rep_id FROM dba_workload_replays;
	END;
	/

spool /ora_bin/rat2/replay_report.html;
select dbms_workload_replay.report( :rep_id, 'HTML') from dual;
spool off
```
