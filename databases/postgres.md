# Useful Queries


Do stuff to lots of tables?
```sql
SELECT
'ANALYZE ' || schemaname || '.' || relname || ';',
schemaname, relname, last_analyze,
pg_total_relation_size(schemaname || '.' || relname)
FROM pg_stat_all_tables
order by 5 desc;
```

Get table sizes
```sql
SELECT
    table_name,
    pg_size_pretty(table_size) AS table_size,
    pg_size_pretty(indexes_size) AS indexes_size,
    pg_size_pretty(total_size) AS total_size
FROM (
    SELECT
        table_name,
        pg_table_size(table_name) AS table_size,
        pg_indexes_size(table_name) AS indexes_size,
        pg_total_relation_size(table_name) AS total_size
    FROM (
        SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
        FROM information_schema.tables
    ) AS all_tables
    ORDER BY total_size DESC
) AS pretty_sizes;
```

Permissions:
- https://stackoverflow.com/questions/22315030/retrieving-all-object-privileges-for-specific-role
- https://dba.stackexchange.com/questions/4286/list-the-database-privileges-using-psql

```sql
SELECT table_catalog, table_schema, grantee, string_agg(privilege_type, ',')
FROM 
(
-- get schema level write permisisons only
select distinct  grantee, table_catalog, table_schema, privilege_type
from information_schema.role_table_grants
where privilege_type  in ('DELETE', 'UPDATE', 'INSERT', 'TRUNCATE')
) schema_grants
-- remote non-produciton schemas
where table_schema not in ('public')
group by grantee, table_catalog, table_schema
order by 1, 2, 3
;
```

###User Roles:
```sql
with roles as
(
 SELECT 
      r.rolname, 
      r.rolsuper, 
      r.rolinherit,
      r.rolcreaterole,
      r.rolcreatedb,
      r.rolcanlogin,
      r.rolconnlimit, r.rolvaliduntil,
-- Find all roles that "users" have access to
  ARRAY(SELECT b.rolname
        FROM pg_catalog.pg_auth_members m
        JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
        WHERE m.member = r.oid) as memberof
, r.rolreplication
, r.rolbypassrls
FROM pg_catalog.pg_roles r
ORDER BY 1
)
select * from roles
-- Only include "users"
where rolcanlogin is true
and array_to_string(memberof, ', ') like '%bids_dba%'
```

### Object Owners
```sql
select nsp.nspname as object_schema,
       cls.relname as object_name, 
       rol.rolname as owner, 
       case cls.relkind
         when 'r' then 'TABLE'
         when 'm' then 'MATERIALIZED_VIEW'
         when 'i' then 'INDEX'
         when 'S' then 'SEQUENCE'
         when 'v' then 'VIEW'
         when 'c' then 'TYPE'
         else cls.relkind::text
       end as object_type
from pg_class cls
  join pg_roles rol on rol.oid = cls.relowner
  join pg_namespace nsp on nsp.oid = cls.relnamespace
where nsp.nspname not in ('information_schema', 'pg_catalog')
  and nsp.nspname not like 'pg_toast%'
  and rol.rolname = "current_user"()  --- remove this if you want to see all objects
order by nsp.nspname, cls.relname;



set role bids_prod_admin;
create user new_user with password 'new_password';
grant bids_developer to new_user;
--bids_qa, bids_developer, bids_analyst
```


```sql
--blocked queries
select pid, 
       usename, 
       pg_blocking_pids(pid) as blocked_by, 
       query as blocked_query
from pg_stat_activity
where cardinality(pg_blocking_pids(pid)) > 0;
```


```sql

```




# Common DBA Tasks
[Common DBA AWS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.html#Appendix.PostgreSQL.CommonDBATasks.Autovacuum)



# Autovacuum
[Tunning Autovacuum in RDS](https://aws.amazon.com/blogs/database/a-case-study-of-tuning-autovacuum-in-amazon-rds-for-postgresql/)
- autovacuum
	- find last autovacuum % dead_tuples in pg_stat_all_tables
	- set to run when % dead tuples - defaults = 10% (.1) - see below aws parameter group
- blocking autovacuum
	- pg_stat_activity
	- `SELECT * FROM pg_terminate_backend('pid')` - use on blocking autovacuum
		- table will stay at highest priority for autovacuum
		- will resume after query completes
- autovacuum_work_mem
- maintenance_work_mem 
	- (kB) Sets the maximum memory to be used for maintenance operations.
	- aws default = GREATEST({DBInstanceClassMemory/63963136*1024},65536)
	- increasing this will help if you see constant autovacuum jobs
- autovacuum_analyze_scale_factor
	- Number of tuple inserts, updates or deletes prior to analyze as a fraction of reltuples
	- default = .05
- autovacuum_vacuum_scale_factor
	- Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.
	- default = .1
- vacuumdb tool

## Vacuumdb
https://www.postgresql.org/docs/9.5/app-vacuumdb.html

Batch file for windows
```
@echo off
setlocal EnableDelayedExpansion

set "startTime=%time: =0%"
echo.
echo Start:    %startTime%
echo.

echo CMD:
@echo on
C:\PostgreSQL\pg10\bin\vacuumdb.exe ^
 -h ohostname ^
 -d database ^
 --analyze ^
 --jobs=20 ^
 --echo

@echo off

set "endTime=%time: =0%"

rem Get elapsed time:
set "end=!endTime:%time:~8,1%=%%100)*100+1!"  &  set "start=!startTime:%time:~8,1%=%%100)*100+1!"
set /A "elap=((((10!end:%time:~2,1%=%%100)*60+1!%%100)-((((10!start:%time:~2,1%=%%100)*60+1!%%100)"

rem Convert elapsed time to HH:MM:SS:CC format:
set /A "cc=elap%%100+100,elap/=100,ss=elap%%60+100,elap/=60,mm=elap%%60+100,hh=elap/60+100"


echo.
echo Start:    %startTime%
echo End:      %endTime%
echo Elapsed:  %hh:~1%%time:~2,1%%mm:~1%%time:~2,1%%ss:~1%%time:~8,1%%cc:~1%
pause
```    

# Tuning
- work_mem 
    - **work_mem is PER operation = multiples per query**
    - default = 4mb
    - [Increasing work_mem and shared_buffers](https://dba.stackexchange.com/questions/27893/increasing-work-mem-and-shared-buffers-on-postgres-9-2-significantly-slows-down)
    - each session = max_sessions can baloon used memory as well!!
    - OLTP / OLAP <-> smaller / larger
    - global or session level setting
        - `SET work_mem = '1 GB';`
        - aws parameter group - reconfigure instance
    - analyze query to determine if in memory or on-disk

- max_connections
	- Sets the maximum number of concurrent connections.
	- default = LEAST({DBInstanceClassMemory/9531392},5000)

- max_worker_processes
	- Sets the maximum number of concurrent worker processes.
	- aws default = 8
	

	
# Table Stats/Info
- [get all row counts](https://stackoverflow.com/questions/2596670/how-do-you-find-the-row-count-for-all-your-tables-in-postgres/5724737)
- depends on last vacuum b/c of dead tuples
- pg_stat_user_table
- pg_stat_all_tables 
    - tuples
    - autovacuum dates
    - analyze dates

- information_schema.
	- tables - table size (bytes)
	- triggers
	- sequences
	- routines (functions)
	
# AWS Parameter group options
- log_statement - default = 'all'
- rds.log_retention_period

## Aurora Postgres
Ran into PostgreSQL Bug with large volume transactions causing OOM error.
Speaking with AWS support they claim an arbitrary 100k record limit per transaction.

Only fix was to batch process in smaller pieces 
which we could not easily do so we ended up switching back to non-Aurora Postgres RDS.

# PG DUMP
https://severalnines.com/database-blog/backup-postgresql-using-pgdump-and-pgdumpall

# copy command
https://stackoverflow.com/questions/1517635/save-pl-pgsql-output-from-postgresql-to-a-csv-file



```bat
pg_dump --host bids-rt-ito-dev.cpcaerjz3tuh.us-east-1.rds.amazonaws.com --port 5432 --username jfassbender --format plain --verbose --file "stagingmisc.temp_trans_to_cj.sql" --table stagingmisc.temp_trans_to_cj ito_dev
pg_dump --host bids-rt-ito-sit.cjsn9udn7ywp.us-east-1.rds.amazonaws.com --port 5432 --username jfassbender --format plain --verbose --file "stagingmisc.temp_trans_to_cj.sql" --table stagingmisc.temp_trans_to_cj ito_sit
pg_dump --host bids-rt-ito-uat.cbmvcsl9lcuw.us-east-1.rds.amazonaws.com --port 5432 --username jfassbender --format plain --verbose --file "stagingmisc.temp_trans_to_cj.sql" --table stagingmisc.temp_trans_to_cj ito_uat





pg_dump --host bids-rt-prod.cpcaerjz3tuh.us-east-1.rds.amazonaws.com --port 5432 --username jfassbender --format plain --verbose --file "stagingmisc.temp_trans_to_cj.sql" --table stagingmisc.temp_trans_to_cj prod

-- change schema name

psql -U ccuser --host=credit-check-prod.cfronqumk9e3.us-east-1.rds.amazonaws.com --dbname=creditcheck < stagingmisc.temp_trans_to_cj.sql

Y2N1JDNyMjAxOCE=





psql -U username -d database -1 -f your_dump.sql

psql -U ccuser --host=itodev-credit-check-dev.cxxtrrcpiehh.us-east-1.rds.amazonaws.com --dbname=creditcheck < stagingmisc.temp_trans_to_cj.sql
psql -U ccuser --host=itosit-credit-check-sit.c8sgo7j7zbvf.us-east-1.rds.amazonaws.com --dbname=creditcheck < stagingmisc.temp_trans_to_cj.sql
psql -U ccuser --host=itouat-credit-check-uat.ce4ytnurobip.us-east-1.rds.amazonaws.com --dbname=creditcheck < stagingmisc.temp_trans_to_cj.sql


psql -U ccuser --host=itouat-credit-check-uat.ce4ytnurobip.us-east-1.rds.amazonaws.com --dbname=creditcheck < stagingmisc.temp_trans_to_cj.sql


copy
COPY (SELECT * FROM ...) TO '/tmp/filename.csv' (FORMAT csv);
psql -U ccuser --host=itouat-credit-check-uat.ce4ytnurobip.us-east-1.rds.amazonaws.com --dbname=creditcheck < stagingmisc.temp_trans_to_cj.sql

psql -U jfassbender --host=i


gis 


creditcheck

Y2N1JDNyMjAxOCE=


```
```bat
C:\Program Files\pgAdmin 4\v4\runtime\pg_dump.exe 
--file "C:\\Users\\JFASSB~1\\PTM-PR~1.BAC" 
--host "ptm-message-queue.cmr9iiqb2m5e.us-east-1.rds.amazonaws.com" 
--port "5432" --username "wow_sysdba" --no-password 
--verbose --format=c --blobs --create --section=pre-data 
--section=data --section=post-data "prod"```

