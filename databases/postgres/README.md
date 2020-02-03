# Common DBA Tasks
[Common DBA AWS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.html#Appendix.PostgreSQL.CommonDBATasks.Autovacuum)



# Autovacuum
[Tunning Autovacuum in RDS](https://aws.amazon.com/blogs/database/a-case-study-of-tuning-autovacuum-in-amazon-rds-for-postgresql/)
- autovacuum
	- find last autovacuum % dead_tuples in pg_stat_all_tables
	- set to run when % dead tuples - defaults = 10% (.1) - see below aws parameter group
- blocking autovacuum
	- pg_stat_activity
	- SELECT * FROM pg_terminate_backend('pid') - use on blocking autovacuum
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
    - test
    

# Tuning
- work_mem 
    - **work_mem is PER transaction = multiples per query**
    - default = 4mb
    - [Increasing work_mem and shared_buffers](https://dba.stackexchange.com/questions/27893/increasing-work-mem-and-shared-buffers-on-postgres-9-2-significantly-slows-down)
    - each session = max_sessions can baloon used memory as well!!
    - OLTP / OLAP <-> smaller / larger
    - global or session level setting
        - SET work_mem = '1 GB';
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