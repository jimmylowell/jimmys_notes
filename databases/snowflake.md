#Snowflake

##USERS / GROUPS

```sql
-- Get list of all privileges users
-- Must be run all at once for result_scan() function
USE ROLE accountadmin;
SHOW GRANTS OF ROLE accountadmin;
SHOW GRANTS OF ROLE sysadmin;
SHOW GRANTS OF ROLE securityadmin;
WITH sox_roles AS
(
SELECT * FROM table(result_scan(last_query_id(-1)))
UNION ALL
SELECT * FROM table(result_scan(last_query_id(-2)))
UNION ALL
SELECT * FROM table(result_scan(last_query_id(-3)))
)
SELECT *, current_timestamp FROM sox_roles;
```