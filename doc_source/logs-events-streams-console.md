# Viewing logs, events, and streams in the Amazon RDS console<a name="logs-events-streams-console"></a>

Amazon RDS integrates with AWS services to show information about logs, events, and database activity streams in the RDS console\.

The **Logs & events** tab for your RDS DB instance shows the following information:
+ **Amazon CloudWatch alarms** – Shows any metric alarms that you have configured for the DB instance\. If you haven't configured alarms, you can create them in the RDS console\. For more information, see [Monitoring Amazon RDS metrics with Amazon CloudWatch](monitoring-cloudwatch.md)\.
+ **Recent events** – Shows a summary of events \(environment changes\) for your RDS DB instance \. For more information, see [Viewing Amazon RDS events](USER_ListEvents.md)\.
+ **Logs** – Shows database log files generated by a DB instance\. For more information, see [Monitoring Amazon RDS log files](USER_LogAccess.md)\.

The **Configuration** tab displays information about database activity streams\.

**To view logs, events, and streams for your DB instance in the RDS console**

1. Sign in to the AWS Management Console and open the Amazon RDS console at [https://console\.aws\.amazon\.com/rds/](https://console.aws.amazon.com/rds/)\.

1. In the navigation pane, choose **Databases**\.

1. Choose the name of the DB instance that you want to monitor\.

   The database page appears\. The following example shows an Oracle database named `orclb`\.  
![\[Database page with monitoring tab shown\]](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/images/oracle-with-monitoring-tab.png)

1. Choose **Logs & events**\.

   The Logs & events section appears\.  
![\[Database page with Logs & events tab shown\]](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/images/oracle-logs-and-events-subpage.png)

1. Choose **Configuration**\.

   The following example shows the status of the database activity streams for your DB instance\.  
![\[Enhanced Monitoring\]](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/images/oracle-das.png)