# Working with the supported foreign data wrappers for Amazon RDS for PostgreSQL<a name="Appendix.PostgreSQL.CommonDBATasks.Extensions.foreign-data-wrappers"></a>

A foreign data wrapper \(FDW\) is a specific type of extension that provides access to external data\. For example, the `oracle_fdw` extension allows your RDS for PostgreSQL DB cluster to work with Oracle databases\. As another example, by using the PostgreSQL native `postgres_fdw` extension you can access data stored in PostgreSQL DB instances external to your RDS for PostgreSQL DB instance\.

Following, you can find information about several supported PostgreSQL foreign data wrappers\. 

**Topics**
+ [Using the log\_fdw extension to access the DB log using SQL](#CHAP_PostgreSQL.Extensions.log_fdw)
+ [Using the postgres\_fdw extension to access external data](#postgresql-commondbatasks-fdw)
+ [Working with MySQL databases by using the mysql\_fdw extension](#postgresql-mysql-fdw)
+ [Working with Oracle databases by using the oracle\_fdw extension](#postgresql-oracle-fdw)
+ [Working with SQL Server databases by using the tds\_fdw extension](#postgresql-tds-fdw)

## Using the log\_fdw extension to access the DB log using SQL<a name="CHAP_PostgreSQL.Extensions.log_fdw"></a>

RDS for PostgreSQL supports the `log_fdw` extension, which you can use to access your database engine log using a SQL interface\. The `log_fdw` extension provides two functions that make it easy to create foreign tables for database logs:
+ `list_postgres_log_files` – Lists the files in the database log directory and the file size in bytes\.
+ `create_foreign_table_for_log_file(table_name text, server_name text, log_file_name text)` – Builds a foreign table for the specified file in the current database\.

All functions created by `log_fdw` are owned by `rds_superuser`\. Members of the `rds_superuser` role can grant access to these functions to other database users\.

By default, the log files are generated by Amazon RDS in `stderr` \(standard error\) format, as specified in `log_destination` parameter\. There are only two options for this parameter, `stderr` and `csvlog` \(comma\-separated values, CSV\)\. If you add the `csvlog` option to the parameter, Amazon RDS generates both `stderr` and `csvlog` logs\. This can affect the storage capacity on your instance, so you need to be aware of the other parameters that affect log handling\. For more information, see [Setting the log destination \(`stderr`, `csvlog`\)](USER_LogAccess.Concepts.PostgreSQL.md#USER_LogAccess.Concepts.PostgreSQL.Log_Format)\. 

One benefit of generating `csvlog` logs is that the `log_fdw` extension lets you build foreign tables with the data neatly split into several columns\. To do this, your instance needs to be associated with a custom DB parameter group so that you can change the setting for `log_destination`\. For more information about how to do so, see [Working with parameters on your RDS for PostgreSQL DB instance](Appendix.PostgreSQL.CommonDBATasks.Parameters.md)\.

The following example assumes that the `log_destination` parameter includes `cvslog`\. 

**To use the log\_fdw extension**

1. Get the `log_fdw` extension\.

   ```
   postgres=> CREATE EXTENSION log_fdw;
   CREATE EXTENSION
   ```

1. Create the log server as a foreign data wrapper\.

   ```
   postgres=> CREATE SERVER log_server FOREIGN DATA WRAPPER log_fdw;
   CREATE SERVER
   ```

1. Select all from a list of log files\.

   ```
   postgres=> SELECT * FROM list_postgres_log_files() ORDER BY 1;
   ```

   A sample response is as follows\.

   ```
             file_name           | file_size_bytes
   ------------------------------+-----------------
    postgresql.log.2016-08-09-22.csv |            1111
    postgresql.log.2016-08-09-23.csv |            1172
    postgresql.log.2016-08-10-00.csv |            1744
    postgresql.log.2016-08-10-01.csv |            1102
   (4 rows)
   ```

1. Create a table with a single 'log\_entry' column for the selected file\.

   ```
   postgres=> SELECT create_foreign_table_for_log_file('my_postgres_error_log',
        'log_server', 'postgresql.log.2016-08-09-22.csv');
   ```

   The response provides no detail other than that the table now exists\.

   ```
   -----------------------------------
   (1 row)
   ```

1. Select a sample of the log file\. The following code retrieves the log time and error message description\.

   ```
   postgres=> SELECT log_time, message FROM my_postgres_error_log ORDER BY 1;
   ```

   A sample response is as follows\.

   ```
                log_time             |                                  message
   ----------------------------------+---------------------------------------------------------------------------
   Tue Aug 09 15:45:18.172 2016 PDT | ending log output to stderr
   Tue Aug 09 15:45:18.175 2016 PDT | database system was interrupted; last known up at 2016-08-09 22:43:34 UTC
   Tue Aug 09 15:45:18.223 2016 PDT | checkpoint record is at 0/90002E0
   Tue Aug 09 15:45:18.223 2016 PDT | redo record is at 0/90002A8; shutdown FALSE
   Tue Aug 09 15:45:18.223 2016 PDT | next transaction ID: 0/1879; next OID: 24578
   Tue Aug 09 15:45:18.223 2016 PDT | next MultiXactId: 1; next MultiXactOffset: 0
   Tue Aug 09 15:45:18.223 2016 PDT | oldest unfrozen transaction ID: 1822, in database 1
   (7 rows)
   ```

## Using the postgres\_fdw extension to access external data<a name="postgresql-commondbatasks-fdw"></a>

You can access data in a table on a remote database server with the [postgres\_fdw](https://www.postgresql.org/docs/current/static/postgres-fdw.html) extension\. If you set up a remote connection from your PostgreSQL DB instance, access is also available to your read replica\. 

**To use postgres\_fdw to access a remote database server**

1. Install the postgres\_fdw extension\.

   ```
   CREATE EXTENSION postgres_fdw;
   ```

1. Create a foreign data server using CREATE SERVER\.

   ```
   CREATE SERVER foreign_server
   FOREIGN DATA WRAPPER postgres_fdw
   OPTIONS (host 'xxx.xx.xxx.xx', port '5432', dbname 'foreign_db');
   ```

1. Create a user mapping to identify the role to be used on the remote server\.

   ```
   CREATE USER MAPPING FOR local_user
   SERVER foreign_server
   OPTIONS (user 'foreign_user', password 'password');
   ```

1. Create a table that maps to the table on the remote server\.

   ```
   CREATE FOREIGN TABLE foreign_table (
           id integer NOT NULL,
           data text)
   SERVER foreign_server
   OPTIONS (schema_name 'some_schema', table_name 'some_table');
   ```

## Working with MySQL databases by using the mysql\_fdw extension<a name="postgresql-mysql-fdw"></a>

To access a MySQL\-compatible database from your RDS for PostgreSQL DB instance, you can install and use the `mysql_fdw` extension\. This foreign data wrapper lets you work with RDS for MySQL, Aurora MySQL, MariaDB, and other MySQL\-compatible databases\. The connection from RDS for PostgreSQL to the MySQL database is encrypted on a best\-effort basis, depending on the client and server configurations\. However, you can enforce encryption if you like\. For more information, see [Using encryption in transit with the extension](#postgresql-mysql-fdw.encryption-in-transit)\. 

The `mysql_fdw` extension is supported on Amazon RDS for PostgreSQL version 14\.2, 13\.6, and higher releases\. It supports selects, inserts, updates, and deletes from an RDS for PostgreSQL DB to tables on a MySQL\-compatible database instance\. 

**Topics**
+ [Setting up your RDS for PostgreSQL DB to use the mysql\_fdw extension](#postgresql-mysql-fdw.setting-up)
+ [Example: Working with an RDS for MySQL database from RDS for PostgreSQL](#postgresql-mysql-fdw.using-mysql_fdw)
+ [Using encryption in transit with the extension](#postgresql-mysql-fdw.encryption-in-transit)

### Setting up your RDS for PostgreSQL DB to use the mysql\_fdw extension<a name="postgresql-mysql-fdw.setting-up"></a>

Setting up the `mysql_fdw` extension on your RDS for PostgreSQL DB instance involves loading the extension in your DB instance and then creating the connection point to the MySQL DB instance\. For that task, you need to have the following details about the MySQL DB instance:
+ Hostname or endpoint\. For an RDS for MySQL DB instance, you can find the endpoint by using the Console\. Choose the Connectivity & security tab and look in the "Endpoint and port" section\. 
+ Port number\. The default port number for MySQL is 3306\. 
+ Name of the database\. The DB identifier\. 

You also need to provide access on the security group or the access control list \(ACL\) for the MySQL port, 3306\. Both the RDS for PostgreSQL DB instance and the RDS for MySQL DB instance need access to port 3306\. If access isn't configured correctly, when you try to connect to MySQL\-compatible table you see an error message similar to the following:

```
ERROR: failed to connect to MySQL: Can't connect to MySQL server on 'hostname.aws-region.rds.amazonaws.com:3306' (110)
```

In the following procedure, you \(as the `rds_superuser` account\) create the foreign server\. You then grant access to the foreign server to specific users\. These users then create their own mappings to the appropriate MySQL user accounts to work with the MySQL DB instance\. 

**To use mysql\_fdw to access a MySQL database server**

1. Connect to your PostgreSQL DB instance using an account that has the `rds_superuser` role\. If you accepted the defaults when you created your RDS for PostgreSQL DB instance, the user name is `postgres`, and you can connect using the `psql` command line tool as follows:

   ```
   psql --host=your-DB-instance.aws-region.rds.amazonaws.com --port=5432 --username=postgres –-password
   ```

1. Install the `mysql_fdw` extension as follows:

   ```
   postgres=> CREATE EXTENSION mysql_fdw;
   CREATE EXTENSION
   ```

After the extension is installed on your RDS for PostgreSQL DB instance, you set up the foreign server that provides the connection to a MySQL database\.

**To create the foreign server**

Perform these tasks on the RDS for PostgreSQL DB instance\. The steps assume that you're connected as a user with `rds_superuser` privileges, such as `postgres`\. 

1. Create a foreign server in the RDS for PostgreSQL DB instance:

   ```
   postgres=> CREATE SERVER mysql-db FOREIGN DATA WRAPPER mysql_fdw OPTIONS (host 'db-name.111122223333.aws-region.rds.amazonaws.com', port '3306');
   CREATE SERVER
   ```

1. Grant the appropriate users access to the foreign server\. These should be non\-administrator users, that is, users without the `rds_superuser` role\.

   ```
   postgres=> GRANT USAGE ON FOREIGN SERVER mysql-db to user1;
   GRANT
   ```

PostgreSQL users create and manage their own connections to the MySQL database through the foreign server\.

### Example: Working with an RDS for MySQL database from RDS for PostgreSQL<a name="postgresql-mysql-fdw.using-mysql_fdw"></a>

Suppose that you have a simple table on an RDS for MySQL DB instance\. Your RDS for PostgreSQL users want to query \(`SELECT`\), `INSERT`, `UPDATE`, and `DELETE` items on that table\. Assume that the `mysql_fdw` extension was created on your RDS for PostgreSQL DB instance, as detailed in the preceding procedure\. After you connect to the RDS for PostgreSQL DB instance as a user that has `rds_superuser` privileges, you can proceed with the following steps\. 

1. On the RDS for PostgreSQL DB instance, create a foreign server: 

   ```
   test=> CREATE SERVER mysqldb FOREIGN DATA WRAPPER mysql_fdw OPTIONS (host 'your-DB.aws-region.rds.amazonaws.com', port '3306');
   CREATE SERVER
   ```

1. Grant usage to a user who doesn't have `rds_superuser` permissions, for example, `user1`:

   ```
   test=> GRANT USAGE ON FOREIGN SERVER mysqldb TO user1;
   GRANT
   ```

1. Connect as *user1*, and then create a mapping to the MySQL user: 

   ```
   test=> CREATE USER MAPPING FOR user1 SERVER mysqldb OPTIONS (username 'myuser', password 'mypassword');
   CREATE USER MAPPING
   ```

1. Create a foreign table linked to the MySQL table:

   ```
   test=> CREATE FOREIGN TABLE mytab (a int, b text) SERVER mysqldb OPTIONS (dbname 'test', table_name '');
   CREATE FOREIGN TABLE
   ```

1. Run a simple query against the foreign table:

   ```
   test=> SELECT * FROM mytab;
   a |   b
   ---+-------
   1 | apple
   (1 row)
   ```

1. You can add, change, and remove data from the MySQL table\. For example:

   ```
   test=> INSERT INTO mytab values (2, 'mango');
   INSERT 0 1
   ```

   Run the `SELECT` query again to see the results:

   ```
   test=> SELECT * FROM mytab ORDER BY 1;
    a |   b
   ---+-------
   1 | apple
   2 | mango
   (2 rows)
   ```

### Using encryption in transit with the extension<a name="postgresql-mysql-fdw.encryption-in-transit"></a>

The connection to MySQL from RDS for PostgreSQL uses encryption in transit \(TLS/SSL\) by default\. However, the connection falls back to non\-encrypted when the client and server configuration differ\. You can enforce encryption for all outgoing connections by specifying the `REQUIRE SSL` option on the RDS for MySQL user accounts\. This same approach also works for MariaDB and Aurora MySQL user accounts\. 

For MySQL user accounts configured to `REQUIRE SSL`, the connection attempt fails if a secure connection can't be established\.

To enforce encryption for existing MySQL database user accounts, you can use the `ALTER USER` command\. The syntax varies, depending on the MySQL version, as shown in the following table\. For more information, see [ALTER USER](https://dev.mysql.com/doc/refman/8.0/en/alter-user.html) in *MySQL Reference Manual*\.


| MySQL 5\.7, MySQL 8\.0 | MySQL 5\.6 | 
| --- | --- | 
|  `ALTER USER 'user'@'%' REQUIRE SSL;`  |  `GRANT USAGE ON *.* to 'user'@'%' REQUIRE SSL;`  | 

For more information about the `mysql_fdw` extension, see the [mysql\_fdw](https://github.com/EnterpriseDB/mysql_fdw) documentation\. 

## Working with Oracle databases by using the oracle\_fdw extension<a name="postgresql-oracle-fdw"></a>

To access an Oracle database from your RDS for PostgreSQL DB instance you can install and use the `oracle_fdw` extension\. This extension is a foreign data wrapper for Oracle databases\. To learn more about this extension, see the [oracle\_fdw](https://github.com/laurenz/oracle_fdw) documentation\.

The `oracle_fdw` extension is supported on RDS for PostgreSQL 12\.7, 13\.3, and higher versions\.

**Topics**
+ [Turning on the oracle\_fdw extension](#postgresql-oracle-fdw.enabling)
+ [Example: Using a foreign server linked to an Amazon RDS for Oracle database](#postgresql-oracle-fdw.example)
+ [Working with encryption in transit](#postgresql-oracle-fdw.encryption)
+ [Understanding the pg\_user\_mappings view and permissions](#postgresql-oracle-fdw.permissions)

### Turning on the oracle\_fdw extension<a name="postgresql-oracle-fdw.enabling"></a>

To use the oracle\_fdw extension, perform the following procedure\. 

**To turn on the oracle\_fdw extension**
+ Run the following command using an account that has `rds_superuser` permissions\.

  ```
  CREATE EXTENSION oracle_fdw;
  ```

### Example: Using a foreign server linked to an Amazon RDS for Oracle database<a name="postgresql-oracle-fdw.example"></a>

The following example shows the use of a foreign server linked to an Amazon RDS for Oracle database\.

**To create a foreign server linked to an RDS for Oracle database**

1. Note the following on the RDS for Oracle DB instance:
   + Endpoint
   + Port
   + Database name

1. Create a foreign server\.

   ```
   test=> CREATE SERVER oradb FOREIGN DATA WRAPPER oracle_fdw OPTIONS (dbserver '//endpoint:port/DB_name');
   CREATE SERVER
   ```

1. Grant usage to a user who doesn't have `rds_superuser` privileges, for example `user1`\.

   ```
   test=> GRANT USAGE ON FOREIGN SERVER oradb TO user1;
   GRANT
   ```

1. Connect as `user1`, and create a mapping to an Oracle user\.

   ```
   test=> CREATE USER MAPPING FOR user1 SERVER oradb OPTIONS (user 'oracleuser', password 'mypassword');
   CREATE USER MAPPING
   ```

1. Create a foreign table linked to an Oracle table\.

   ```
   test=> CREATE FOREIGN TABLE mytab (a int) SERVER oradb OPTIONS (table 'MYTABLE');
   CREATE FOREIGN TABLE
   ```

1. Query the foreign table\.

   ```
   test=>  SELECT * FROM mytab;
   a
   ---
   1
   (1 row)
   ```

If the query reports the following error, check your security group and access control list \(ACL\) to make sure that both instances can communicate\.

```
ERROR: connection for foreign table "mytab" cannot be established
DETAIL: ORA-12170: TNS:Connect timeout occurred
```

### Working with encryption in transit<a name="postgresql-oracle-fdw.encryption"></a>

PostgreSQL\-to\-Oracle encryption in transit is based on a combination of client and server configuration parameters\. For an example using Oracle 21c, see [About the Values for Negotiating Encryption and Integrity](https://docs.oracle.com/en/database/oracle/oracle-database/21/dbseg/configuring-network-data-encryption-and-integrity.html#GUID-3A2AF4AA-AE3E-446B-8F64-31C48F27A2B5) in the Oracle documentation\. The client used for oracle\_fdw on Amazon RDS is configured with `ACCEPTED`, meaning that the encryption depends on the Oracle database server configuration\.

If your database is on RDS for Oracle, see [Oracle native network encryption](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.Oracle.Options.NetworkEncryption.html) to configure the encryption\.

### Understanding the pg\_user\_mappings view and permissions<a name="postgresql-oracle-fdw.permissions"></a>

The PostgreSQL catalog `pg_user_mapping` stores the mapping from an RDS for PostgreSQL user to the user on a foreign data \(remote\) server\. Access to the catalog is restricted, but you use the `pg_user_mappings` view to see the mappings\. In the following, you can find an example that shows how permissions apply with an example Oracle database, but this information applies more generally to any foreign data wrapper\.

In the following output, you can find roles and permissions mapped to three different example users\. Users `rdssu1` and `rdssu2` are members of the `rds_superuser` role, and `user1` isn't\. The example uses the `psql` metacommand `\du` to list existing roles\.

```
test=>  \du
                                                               List of roles
    Role name    |                         Attributes                         |                          Member of
-----------------+------------------------------------------------------------+-------------------------------------------------------------
 rdssu1          |                                                            | {rds_superuser}
 rdssu2          |                                                            | {rds_superuser}
 user1           |                                                            | {}
```

All users, including users that have `rds_superuser` privileges, are allowed to view their own user mappings \(`umoptions`\) in the `pg_user_mappings` table\. As shown in the following example, when `rdssu1` tries to obtain all user mappings, an error is raised even though `rdssu1``rds_superuser` privileges:

```
test=> SELECT * FROM pg_user_mapping;
ERROR: permission denied for table pg_user_mapping
```

Following are some examples\.

```
test=> SET SESSION AUTHORIZATION rdssu1;
SET
test=> SELECT * FROM pg_user_mappings;
 umid  | srvid | srvname | umuser | usename    |            umoptions
-------+-------+---------+--------+------------+----------------------------------
 16414 | 16411 | oradb   |  16412 | user1      |
 16423 | 16411 | oradb   |  16421 | rdssu1     | {user=oracleuser,password=mypwd}
 16424 | 16411 | oradb   |  16422 | rdssu2     |
 (3 rows)

test=> SET SESSION AUTHORIZATION rdssu2;
SET
test=> SELECT * FROM pg_user_mappings;
 umid  | srvid | srvname | umuser | usename    |            umoptions
-------+-------+---------+--------+------------+----------------------------------
 16414 | 16411 | oradb   |  16412 | user1      |
 16423 | 16411 | oradb   |  16421 | rdssu1     |
 16424 | 16411 | oradb   |  16422 | rdssu2     | {user=oracleuser,password=mypwd}
 (3 rows)

test=> SET SESSION AUTHORIZATION user1;
SET
test=> SELECT * FROM pg_user_mappings;
 umid  | srvid | srvname | umuser | usename    |           umoptions
-------+-------+---------+--------+------------+--------------------------------
 16414 | 16411 | oradb   |  16412 | user1      | {user=oracleuser,password=mypwd}
 16423 | 16411 | oradb   |  16421 | rdssu1     |
 16424 | 16411 | oradb   |  16422 | rdssu2     |
 (3 rows)
```

Because of implementation differences between `information_schema._pg_user_mappings` and `pg_catalog.pg_user_mappings`, a manually created `rds_superuser` requires additional permissions to view passwords in `pg_catalog.pg_user_mappings`\.

No additional permissions are required for an `rds_superuser` to view passwords in `information_schema._pg_user_mappings`\.

Users who don't have the `rds_superuser` role can view passwords in `pg_user_mappings` only under the following conditions:
+ The current user is the user being mapped and owns the server or holds the `USAGE` privilege on it\.
+ The current user is the server owner and the mapping is for `PUBLIC`\.

## Working with SQL Server databases by using the tds\_fdw extension<a name="postgresql-tds-fdw"></a>

You can use the PostgreSQL `tds_fdw` extension to access databases that support the tabular data stream \(TDS\) protocol, such as Sybase and Microsoft SQL Server databases\. This foreign data wrapper lets you connect from your RDS for PostgreSQL DB instance to databases that use the TDS protocol, including Amazon RDS for Microsoft SQL Server\. For more information, see [tds\-fdw/tds\_fdw](https://github.com/tds-fdw/tds_fdw) documentation on GitHub\. 

The `tds_fdw` extension is supported on Amazon RDS for PostgreSQL version 14\.2, 13\.6, and higher releases\. 

### Setting up your RDS for PostgreSQL DB to use the tds\_fdw extension<a name="postgresql-tds-fdw-setting-up"></a>

In the following procedures, you can find an example of setting up and using the `tds_fdw` with an RDS for PostgreSQL DB instance\. Before you can connect to a SQL Server database using `tds_fdw`, you need to get the following details for the instance:
+ Hostname or endpoint\. For an RDS for SQL Server DB instance, you can find the endpoint by using the Console\. Choose the Connectivity & security tab and look in the "Endpoint and port" section\. 
+ Port number\. The default port number for Microsoft SQL Server is 1433\. 
+ Name of the database\. The DB identifier\. 

You also need to provide access on the security group or the access control list \(ACL\) for the SQL Server port, 1433\. Both the RDS for PostgreSQL DB instance and the RDS for SQL Server DB instance need access to port 1433\. If access isn't configured correctly, when you try to query the Microsoft SQL Server you see the following error message:

```
ERROR: DB-Library error: DB #: 20009, DB Msg: Unable to connect:
Adaptive Server is unavailable or does not exist (mssql2019.aws-region.rds.amazonaws.com), OS #: 0, OS Msg: Success, Level: 9
```

**To use tds\_fdw to connect to a SQL Server database**

1. Connect to your PostgreSQL DB instance using an account that has the `rds_superuser` role:

   ```
   psql --host=your-DB-instance.aws-region.rds.amazonaws.com --port=5432 --username=test –-password
   ```

1. Install the `tds_fdw` extension:

   ```
   test=> CREATE EXTENSION tds_fdw;
   CREATE EXTENSION
   ```

After the extension is installed on your RDS for PostgreSQL DB instance, you set up the foreign server\.

**To create the foreign server**

Perform these tasks on the RDS for PostgreSQL DB instance using an account that has `rds_superuser` privileges\. 

1. Create a foreign server in the RDS for PostgreSQL DB instance:

   ```
   test=> CREATE SERVER sqlserverdb FOREIGN DATA WRAPPER tds_fdw OPTIONS (servername 'mssql2019.aws-region.rds.amazonaws.com', port '1433', database 'tds_fdw_testing');
   CREATE SERVER
   ```

1. Grant permissions to a user who doesn't have `rds_superuser` role privileges, for example, `user1`:

   ```
   test=> GRANT USAGE ON FOREIGN SERVER sqlserverdb TO user1;
   ```

1. Connect as user1 and create a mapping to a SQL Server user:

   ```
   test=> CREATE USER MAPPING FOR user1 SERVER sqlserverdb OPTIONS (username 'sqlserveruser', password 'password');
   CREATE USER MAPPING
   ```

1. Create a foreign table linked to a SQL Server table:

   ```
   test=> CREATE FOREIGN TABLE mytab (a int) SERVER sqlserverdb OPTIONS (table 'MYTABLE');
   CREATE FOREIGN TABLE
   ```

1. Query the foreign table:

   ```
   test=> SELECT * FROM mytab;
    a
   ---
    1
   (1 row)
   ```

#### Using encryption in transit for the connection<a name="postgresql-tds-fdw-ssl-tls-encryption"></a>

The connection from RDS for PostgreSQL to SQL Server uses encryption in transit \(TLS/SSL\) depending on the SQL Server database configuration\. If the SQL Server isn't configured for encryption, the RDS for PostgreSQL client making the request to the SQL Server database falls back to unencrypted\.

You can enforce encryption for the connection to RDS for SQL Server DB instances by setting the `rds.force_ssl` parameter\. To learn how, see [Forcing connections to your DB instance to use SSL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.Concepts.General.SSL.Using.html#SQLServer.Concepts.General.SSL.Forcing)\. For more information about SSL/TLS configuration for RDS for SQL Server, see [Using SSL with a Microsoft SQL Server DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.Concepts.General.SSL.Using.html)\. 