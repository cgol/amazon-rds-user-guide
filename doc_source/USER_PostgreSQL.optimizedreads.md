# Improving query performance for RDS for PostgreSQL with Amazon RDS Optimized Reads<a name="USER_PostgreSQL.optimizedreads"></a>

You can achieve faster query processing for RDS for PostgreSQL with Amazon RDS Optimized Reads\. An RDS for PostgreSQL DB instance that uses RDS Optimized Reads can achieve up to 50% faster query processing compared to a DB instance that doesn't use it\.

**Topics**
+ [Overview of RDS Optimized Reads in PostgreSQL](#USER_PostgreSQL.optimizedreads-overview)
+ [Use cases for RDS Optimized Reads](#USER_PostgreSQL.optimizedreads-use-cases)
+ [Best practices for RDS Optimized Reads](#USER_PostgreSQL.optimizedreads-best-practices)
+ [Using RDS Optimized Reads](#USER_PostgreSQL.optimizedreads-using)
+ [Monitoring DB instances that use RDS Optimized Reads](#USER_PostgreSQL.optimizedreads-monitoring)
+ [Limitations for RDS Optimized Reads in PostgreSQL](#USER_PostgreSQL.optimizedreads-limitations)

## Overview of RDS Optimized Reads in PostgreSQL<a name="USER_PostgreSQL.optimizedreads-overview"></a>

Optimized Reads is available by default on RDS for PostgreSQL versions 15\.2 and higher, 14\.7 and higher, and 13\.10 and higher\. 

When you use an RDS for PostgreSQL DB instance that has RDS Optimized Reads turned on, your DB instance achieves up to 50% faster query performance using the local Non\-Volatile Memory Express \(NVMe\) based solid state drive \(SSD\) block\-level storage\. You can achieve faster query processing by placing the temporary tables that are generated by PostgreSQL on the local storage, which reduces the traffic to Elastic Block Storage \(EBS\) over the network\.

In PostgreSQL, temporary objects are assigned to a temporary namespace that drops automatically at the end of the session\. The temporary namespace while dropping removes any objects that are session\-dependent, including schema\-qualified objects, such as tables, functions, operators, or even extensions\.

In RDS for PostgreSQL, the `temp_tablespaces` parameter is configured for this temporary work area where the temporary objects are stored\.

The following queries return the name of the tablespace and its location\.

```
postgres=> show temp_tablespaces;
temp_tablespaces
---------------------
rds_temp_tablespace
(1 row)
```

```
postgres=> SELECT spcname,pg_tablespace_location(oid) FROM pg_tablespace WHERE pg_tablespace_location(oid) LIKE '%local%';
spcname              |  pg_tablespace_location
---------------------+--------------------------------------
rds_temp_tablespace  | /rdslocalstorage/rds_temp_tablespace
(1 row)
```

You can always switch back to Amazon EBS storage by modifying this parameter in the `Parameter group` using the AWS Management Console to point to any tablespace other than `rds_temp_tablespace`\. For more information, see [ Modifying parameters in a DB parameter group](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithDBInstanceParamGroups.html#USER_WorkingWithParamGroups.Modifying)\. You can also use the SET command to modify the value of the `temp_tablespaces` parameter to `pg_default` at the session level using SET command\. Modifying the parameter redirects the temporary work area to Amazon EBS\. Switching back to Amazon EBS helps when the local storage for your RDS instance isn't sufficient to perform a specific SQL operation\.

```
postgres=> SET temp_tablespaces TO 'pg_default';
SET
```

```
postgres=> show temp_tablespaces;
            
 temp_tablespaces
------------------
 pg_default
```

## Use cases for RDS Optimized Reads<a name="USER_PostgreSQL.optimizedreads-use-cases"></a>

The following are some use cases that can benefit from Optimized Reads:
+ Analytical queries that include Common Table Expressions \(CTEs\), derived tables, and grouping operations\.
+ Read replicas that handle the unoptimized queries for an application\.
+ On\-demand or dynamic reporting queries with complex operations such as GROUP BY and ORDER BY that can't always use appropriate indexes\.
+ Other workloads that use internal temporary tables\.

## Best practices for RDS Optimized Reads<a name="USER_PostgreSQL.optimizedreads-best-practices"></a>

Use the following best practices for RDS Optimized Reads:
+ Add retry logic for read\-only queries in case they fail because the instance store is full during the execution\.
+ Monitor the storage space available on the instance store with the CloudWatch metric `FreeLocalStorage`\. If the instance store is reaching its limit because of the workload on the DB instance, modify the DB instance to use a larger DB instance class\.

## Using RDS Optimized Reads<a name="USER_PostgreSQL.optimizedreads-using"></a>

When you provision an RDS for PostgreSQL DB instance with one of the following DB instance classes in a Single\-AZ DB deployment or Multi\-AZ deployment, the DB instance automatically uses RDS Optimized Reads:
+ db\.m6gd
+ db\.r6gd
+ db\.m5d
+ db\.r5d

For more information about Multi\-AZ deployment, see [ Configuring and managing a Multi\-AZ deployment](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)\.

To turn on RDS Optimized Reads, do one of the following:
+ Create an RDS for PostgreSQL DB instance using one of these DB instance classes\. For more information, see [Creating an Amazon RDS DB instance](USER_CreateDBInstance.md)\.
+ Modify an existing RDS for PostgreSQL DB instance to use one of these DB instance classes\. For more information, see [Modifying an Amazon RDS DB instance](Overview.DBInstance.Modifying.md)\.

RDS Optimized Reads is available in all AWS Regions where one or more of these DB instance classes are supported\. For more information, see [DB instance classes](Concepts.DBInstanceClass.md)\.

To switch back to a non\-optimized reads RDS instance, modify the DB instance class of your RDS instance to the similar instance class that only supports EBS storage for your database workloads\. For example, if the current DB instance class is db\.r6gd\.4xlarge, choose db\.r6g\.4xlarge to switch back\. For more information, see [Modifying an Amazon RDS DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.DBInstance.Modifying.html)\.

## Monitoring DB instances that use RDS Optimized Reads<a name="USER_PostgreSQL.optimizedreads-monitoring"></a>

You can monitor DB instances that use RDS Optimized Reads using the following CloudWatch metrics:
+ `FreeLocalStorage`
+ `ReadIOPSLocalStorage`
+ `ReadLatencyLocalStorage`
+ `ReadThroughputLocalStorage`
+ `WriteIOPSLocalStorage`
+ `WriteLatencyLocalStorage`
+ `WriteThroughputLocalStorage`

These metrics provide data about available instance store storage, IOPS, and throughput\. For more information about these metrics, see [Amazon CloudWatch instance\-level metrics for Amazon RDS](rds-metrics.md#rds-cw-metrics-instance)\.

To monitor current usage of your local storage, log in to your database using the following query:

```
SELECT
    spcname AS "Name",
    pg_catalog.pg_size_pretty(pg_catalog.pg_tablespace_size(oid)) AS "size"
FROM
    pg_catalog.pg_tablespace
WHERE
    spcname IN ('rds_temp_tablespace');
```

For more information about the temporary files and their usage, see [Managing temporary files with PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.ManagingTempFiles.html)\.

## Limitations for RDS Optimized Reads in PostgreSQL<a name="USER_PostgreSQL.optimizedreads-limitations"></a>

The following limitations apply to RDS Optimized Reads in PostgreSQL:
+ Transactions can fail when the instance store is full\.
+ If a source instance doesn't have local storage, then read replica with local storage is unsupported\.
+ Point\-in\-time restore from a local storage instance to non\-local storage instance is unsupported\.
+ RDS Optimized Reads in PostgreSQL is available in all Regions except the following:
  + China \(Beijing\) Region
  + China \(Ningxia\)