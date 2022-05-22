---
title: "Slashing AWS RDS costs for a database instance in an enterprise context"
date: 2022-05-22T12:28:22+02:00
authors: ["Geert Van Wauwe"]
draft: false
---

# A step by step guide on how to reduce costs for an RDS instance
Some obvious and some lesser-known cost savings for AWS RDS and the order in which to apply them are discussed.
Some cost savings come with deliberate tradeoffs and depend on the context in which the RDS instance operates.
These careful considerations are applied on a real life use case; A MySQL RDS instance.

Finally, an overview of all the cost savings is presented for the MySQL RDS instance.

# What is AWS RDS

AWS RDS is AWS's relational database service.
It supports proprietary and open source database engines.
Amazon Aurora, MySQL, MariaDB, Oracle, SQL Server, and PostgreSQL are the current options to choose from.
This service should be considered if you want to have a relational database without wanting to deal with maintenance, durability, disaster recovery and high availability.

## Real life use case RDS Instance

A MySQL RDS instance in a production environment is used as an example of what savings can be applied.
The instance is used 6 days a week and experiences its highest load during extended business hours.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_Typical_Daily_CPUUtilization.png)
Housekeeping is performed every night which explains the small peak around 1:00AM.

# Cost Savings

Using AWS RDS can become costly rather quickly. 
Some options in the RDS configuration should be a deliberate choice tailored to the context in which the instance operates.
An oversight during setup or a failure to reevaluate a configuration can make your AWS cost exponentially higher than it should be.

## AWS RDS

The best way to reduce AWS RDS costs is by not using AWS RDS at all!
EC2 is perfectly suited to host a relational database and will cost you approximately 50% less.
If you do not care about high availability, disaster recovery, security or performance then this is definitely the way to go.
Most businesses however do care, so I would not consider this a viable option anymore in 2022 except for maybe on a hobby project.

## Database instance class, type and size

When starting with AWS RDS consider which instance class you want to use.  
Consider the burstable performance class (the t-instance types), when you have a non-constant workload over 24 hours,
such as when the database is almost exclusively used during the day.  
Consider the memory optimized class (R,x and z instance types) when you have a read heavy database workload.  
Consider the general purpose class for other workloads.

## Database instance type

Choosing the right database instance type and size is paramount in reducing AWS RDS costs.
If you are not quite sure how much Memory or CPU the instance will need, it is best to over-provision at the start with a rather large instance type.  
Once a utilization baseline has been established, you can consider scaling down the instance type vertically.

### Scaling down instance type

To scale down the instance type confidently, check both `CPUUtilization` and `FreeableMemory` metrics.
Scale down when the maximum of the `CPUUtilization` metric is below 50% for more than 99.9% of the time and when the currently used memory (instance type memory minus freeable memory) is less than 95% of the memory of the targeted scaled down database instance type.

The `FreeableMemory` metric is not always a good performance predictor.
Consider using enhanced monitoring's Free Memory metric when the `FreeableMemory` drops below 15% of the configured memory of the RDS instance or when the `Swap Usage` metric increases regularly.


Reducing the VCPUs by a factor of 2, roughly increases the CPUUtilization percentage by a factor of 2. 
This heuristic can help you estimate how much you can downscale your database instance and when you would cross the 60% CPU Utilization limit more than a couple of minutes a day.

Remember that Changing the instance type or size will lead to some downtime, so schedule it properly.

### Real life use case

Initially a general purpose instance (`db.m6g`) of size `2xlarge` was chosen as relational database.
Since the database is primarily used during extended business hours, the instance class burstable performance (`db.t4g`) seems the most appropriate choice.

#### Analyzing the metrics

##### CPU Utilization analysis

A `db.m6g.2xlarge` instance has 8 VCPUs. As you can see on the graph, on the 21st of March, the maximum of the CPU Utilization within a 5-minute period, crosses 30% more than a couple of times.

As mentioned before, reducing the VCPUs by a factor of 2, roughly increases the CPUUtilization percentage by a factor of 2. 

Scaling down vertically to 4 VCPUs is the limit since the maximum of CPU Utilization will cross the 50% barrier more than a couple of minutes a day.
`db.t4g.xlarge` has 4 VCPUs, so do not use a smaller instance type than `db.t4g.xlarge`.

![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_CPUUtilization_Before.png)

##### Freeable Memory analysis

A `db.m6g.2xlarge` instance has `32GB` of memory. 
As you can see on the graph, on the 21st of March there is `25GB` of `Freeable Memory` so approximately `7GB` is used.
Therefore, it is possible to estimate that `8GB` of RAM is sufficient.
`db.t4g.large` has 8GB of RAM, so do not go smaller than `db.t4g.large`.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_FreeableMemory_Before.png)

#### Conclusions

Since CPU seems to be the bottleneck, the instance type was changed to `t4g.xlarge` on the 23rd of March.
The effect on the `Freeable Memory` and the maximum of the `CPU Utilization` within a 5-minute period, is shown in the graphs below.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_CPUUtilization.png)
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_FreeableMemory.png)

#### Cost reduction

Initial compute cost = $0.672 X 24 X 30 = $483.84.
compute cost after scale down = $0.277 X 24 X 30 = $199.44
A cost reduction of 58.8%.

### An additional real life use case

At the time of writing a different MySQL RDS instance is running in production.

Enhanced monitoring shows that it only has 2% Free Memory, which is less than the 5% that [AWS advises](https://aws.amazon.com/premiumsupport/knowledge-center/low-freeable-memory-rds-mysql-mariadb/).
![MySQL RDS Free memory](/MySQL_RDS_Instance_FreeMemory.png)
The monitoring shows that the `SwapUsage` metric is not zero.
![MySQL RDS Swap usage](/MySQL_RDS_Instance_SwapUsage.png)
The write latency, read latency, or CPU Utilization is not impacted at those times.
![MySQL RDS Swap usage](/MySQL_RDS_Instance_ReadAndWriteLatency.png)
Scaling up vertically seems reasonable when looking at the `SwapUsage` metric and enhanced monitoring's free memory.
However, since there is no impact on the main performance parameters of the database, scaling up vertically was deemed not necessary.

At this point it is a calculated risk which reduces compute costs significantly.
Keep in mind that scaling up by one instance size increases compute cost by a factor of 2.

#### Alarms

It is strongly recommended having some CloudWatch alarms in place.
You will want to be aware of when the read or write latency exceeds a certain threshold or when the CPU utilization reaches 100% regularly.
Especially when it has a noticeable impact on end users.

## Multi-AZ 

RDS instances can be run as a multi-AZ deployment. 
Choose this option if your database needs to be high available and when you have a low RTO and a very low RPO.
It will provide [better recovery](https://aws.amazon.com/blogs/database/amazon-rds-under-the-hood-single-az-instance-recovery/) from **non-recoverable** instance failures or when an availability zone goes down, compared to single-AZ instances.

Consider disabling multi-AZ when fast automatic recovery is not necessary, which is the case for most test environments.  
Running the test RDS instance in single-AZ deployment will cut its compute and storage cost in half.

### Real life use case

The RDS instances in the test environment run in single-AZ deployment, because the impact of having a non-recoverable RDS instance failure is limited.  
The MySQL RDS instances that run in production have been deployed multi-AZ, as the cost of downtime is rather large.  
The test environment was already configured as single-AZ and the multi-AZ configuration of the MySQL RDS instance was not changed, so no costs were saved.

## Stopping unused RDS instances versus Reserved Instances

Compute costs can be reduced by reserving instances or by stopping RDS instances during periods in which they are not used.
You do not pay for the compute cost when an RDS instance is stopped, so compare this cost reduction carefully with reserving instances.

Also keep in mind that any stopped RDS instance will be restarted automatically after 7 days.

### Stopping Unused RDS instances

Analyze the usage patterns of your RDS instances.
If the database is only used during business hours or extended business hours, consider stopping the RDS instance in periods where it is not used to save on compute costs.
AWS will still charge you for the allocated storage of the RDS instance, but you will not have to pay for the compute part of the RDS instance costs.

If your RDS instances is only used 5 days a week from 9AM until 5PM then you can stop the RDS instance at 5PM and start it again at 9AM on weekdays.
The RDS instance will only be used 40 hours a week, which is about 24% of the time, so you can save approximately 76% on RDS compute costs.

You can use an eventbridge rule that runs on a schedule to triggers a Lambda Function which can stop and start an RDS instance.

### Reserved instances

Once you have decided upon which instance class to use, you should look into reserved instances for AWS RDS.
Reserved instances allows you to save up to almost 70% of your costs compared to the on-demand rate.
The cost saving is heavily instance type dependent. 

#### Size flexibility

Bear in mind that you can also make use of size flexibility for Reserved Instances.
A reserved database instance of a certain size, can be applied to 50% of the usage of a database instance that is one size bigger.  
A reserved single-AZ instance can be applied to 50% of the usage of a multi-AZ database instance of the same size.

If you are sure about the RDS instance family ( `db.m6`, `db.r3`,`db.t4g`,...) and the minimum instance type (`medium`, `large`,`2xlarge`,...) to cover the baseline load, you can reserve the db instance already.
If at some point in the future the load increases, you can add reserved instances to cover the new baseline load.

### Which option should I pick?

Pick the option with the largest cost saving.
If both are approximately equal consider whether you are sure that you will need the RDS instance for the reserved amount of time.
If there is any doubt, go for stopping and starting instances during periods in which they are not used.

### Real life use case

RDS instances in the production environment are not started and stopped because the database is used most of the time.
Starting and stopping RDS instances was not chosen for the test environment either.

The product backlog is already filled to the brim, so it probably would have taken another couple of weeks before we could start working on it.
Reserving the instances is just a couple of clicks, so reserving production and test instances for one year seemed cost optimal given the backlog.

It is difficult to predict the workload on the production database for years to come.
Hence, reserving RDS instances for one year with partial upfront payment seems the best option for now.
Reserving the `t4g` instances in `eu-west-1` leads to a cost reduction of approximately 37%, see [Reserved Instances](https://aws.amazon.com/rds/mysql/pricing/).
Although strictly speaking, stopping and starting instances would have probably been cheaper.

Regarding size flexibility. Once the RDS instance class has been decided upon, you should reserve the minimum expected baseline.
In our AWS account there are 3 test MySQL RDS instances and 3 production MySQL instances.
All of them are primarily used during extended business hours but the loads differ.
Since all access patterns are similar the burstable database instance family `db.t4g` is chosen.

The currently experienced baseline load for a test environment is covered by one `t4g.medium` (single-AZ).
The currently experienced baseline load over the three production environments is covered by 1 `t4g.medium`, 1 `t4g.large` and 1 `t4g.xlarge`. 
All the production environments utilize a multi-AZ configuration.

By making use of size flexibility it is not that difficult to calculate the amount of `t4g.medium` single-AZ instances necessary.
You can calculate the current baseline by adding up all database instance equivalent baselines.

1 `t4g.xlarge` (multi-AZ) = 2 `t4g.large` (multi-AZ)  
= 2 ( 2 `t4g.medium` (multi-AZ)) = 4. ( 2 `t4g.medium` (single AZ))   
= 8 `t4g.medium` (single AZ)  
So one `t4g.xlarge` multi-AZ instance can be reserved by reserving 8 single-AZ `t4g.medium` instances.
Similar calculations show that reserving 14 single-AZ `t4g.medium` instances is cost optimal for the production environment, while 3 single-AZ `t4g.medium` instances is cost optimal for the test environment.

## Allocated storage

There are multiple ways to save costs on allocated storage.

### Choosing the correct storage types

There are two storage types for RDS instances:
- GP2
- IO1

The choice of storage type for RDS determines the type of EBS Volume, the hard drive of the database, attached to the RDS instance.
GP2 is great for variable workloads and should be your default option.
Only Consider IO1 when you need more IOPS than can be provided by GP2 or when there is a requirement on your database to always have some provisioned IOPS.
As of now when RDS instance storage is mentioned, a GP2 database is implied.

As mentioned before, you will pay for every availability zone in which the RDS instance is deployed so choosing a multi-AZ RDS instance will increase your storage costs by at least by a factor of 2.

### Reducing storage size

You can check whether most of the allocated storage of an RDS instance is used by comparing its `FreeStorageSpace` metric to its storage size.
It is tempting to reduce the storage size to what you actually use plus some margin for performance and some additional margin for safety.
Unfortunately it is not that simple.
The allocated storage of an RDS is directly proportional to its IOPS.
So you will have to make sure that the IOPS capacity is sufficient after storage size reduction!


#### IOPS

IOPS is a way of representing the input or output operations per second.
RDS instances backed by a GP2 EBS Volume will give you a BurstBalance of IOPS which is basically a bucket of credits, that can be used for queries and writes to the RDS instance.
RDS instances backed by a GP2 EBS Volume IO1 will give you a provisioned amount of IOPS.

##### IOPS baseline

For provisioned IOPS i.e. IO1 the baseline is the amount of provisioned IOPS.
For GP2 you can calculate the baseline IOPS based on the allocated storage size in GB:  
GP2 baseline IOPS = maximum of 3 IOPS/GB * storage size in GB  
and  
100 IOPS = MAX(3*storageSizeInGB, 100) IOPS  
The minimum amount of IOPS for a GP2 volume is thus 100 IOPS.

Most RDS instances have a variable workload so that GP2 is the better candidate.
In what follows only GP2 volumes are considered.

#####  An example on how to calculate the IOPS baseline for a GP2 backed RDS instance

This is relevant for small volumes as shown by the following example:
Given an RDS instance with 25 GB of allocated storage, the calculated baseline IOPS is the maximum of:  
3 IOPS/GB * storage size in GB = 3 IOPS/GB * 25 GB = 75 IOPS  
and  
100 IOPS  
Since 75 IOPS is smaller than 100 IOPS, the baseline IOPS for this RDS instance will be 100 IOPS.

#### Storage costs

Storage costs are a bit higher than $0.11 per Gigabyte per month, depending on the region.
If you have a single-AZ RDS instance with 1 TB of allocated GP2 storage, it will cost you a bit more than $110 per month.
Having a multi-AZ RDS instance will double the cost, since the secondary instance also has a GP2 storage of the same size attached to it.

Look at the Free Storage Space metric for the RDS instance to see how much Storage space the RDS instance actually uses.
It is tempting to reduce the storage size to what you actually use plus some margin for safety. But do not do that just yet.
Make sure you have enough IOPS capacity after storage size reduction!

#### Reducing the storage size while taking IOPS into account

If IOPS is the bottleneck, you can calculate the required allocated storage so that the BurstBalance never runs out of credits.
This will be based on historical metrics of your database instance.
In order to retrieve these metrics:
1) Go to CloudWatch.
2) Search for `WriteIOPS`.
3) Click on Per-Database Metrics.
4) Check the box for `WriteIOPS` for the database in question.
5) Remove the query `WriteIOPS`.
6) Search for `ReadIOPS`.
7) Check the box for `ReadIOPS` for the database in question.
8) Apply a representative timerange.
9) Select `Average` as the `Statistic` in the `Graphed Metrics` Tab.
10) Select `5 minutes` as the `Period` in the `Graphed Metrics` Tab.
11) In the upper right corner click on `Actions` .
12) After expansion of `Actions` click on `Download as .csv`.


#### Estimating the storage size based on historical IOPS data

A downloadable spreadsheet was made so that a cost-effective storage size without performance degradation can be estimated.  
Follow these steps to make the spreadsheet use your historical data:
1) Open a google spreadsheet.
2) Import the [spreadsheet](https://docs.google.com/spreadsheets/d/e/2PACX-1vSnpbVo4KZKdRghQen-FLAOwft3OKDbOHogU2hn8sDrS35dc5kRkPKfnRVEso7jMTcnQ3nA64D17A9F/pub?output=ods) and choose `replace` to replace the recently opened spreadsheet.
3) Add a Column between Column C and Column D. This is important. Do not skip this step.
4) Highlight the first three columns, then select import.
5) Choose upload.
6) Select the .csv file containing the IOPS metrics.
7) In the Import File configuration select `Replace data at selected cell` as `import location`.
8) In the Import File configuration select `Detect automatically` as `separator type.
9) In the Import File configuration check `Convert text to numbers, dates and formulas`.
10) Click import data in the Import File configuration dialog box.
11) Remove the recently added D column again.
12) Expand the formulas for column D,E,F,G,H and I, by going to row 3666 and double-clicking the bottom right corner of each last filled cell.
13) Go to the `chart` sheet and edit the range of the chart to `data!A6:I{lastrow}` where `{lastrow}` is the last row number of your data.
14) In the `edit chart` dialog, remove the Series you do not need such as `data!B6:I{lastrow}` and `data!C6:I{lastrow}`.
15) Now you can start playing around with the estimated IOPS located in the third row.
16) See what the credits would have been if this amount of IOPS was added to the `BurstBalance` credits every second.
17) Find an IOPS number such that the projected `BurstBalance` is comfortable enough at all times for your situation.
18) Ideally the `BurstBalance` should never reach 0, since then the performance would be impacted heavily.
19) Divide the IOPS number by 3 to find the GP2 storage size you minimally need to sustain the burst necessary for your database access patterns.

#### The cost optimized storage size without performance degradation

The cost optimized storage size without performance degradation for an RDS instance is the maximum of:  
1) the currently used storage size plus 17.5% as recommended by [the RDS performance guidelines](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MonitoringOverview.html) and then some additional margin for safety.
2) the storage size as estimated by using the above-mentioned spreadsheet that takes the historical IOPS into account which already contains some margin for safety.

Configure [Amazon RDS storage autoscaling](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.StorageTypes.html#USER_PIOPS.Autoscaling) if you expect the storage size to grow.

#### Real life use case

The `ReadIOPS` and `WriteIOPS` in the spreadsheet are the production `ReadIOPS` and `WriteIOPS` of the MySQL instance in a limited timeframe.

##### Interpreting the data in the spreadsheet by making use of the graph

On the 15th of May the hypothetical `BurstBalance` was at its lowest point in the selected time interval.
![Graph of projected lowest burst balance](/MySQL_RDS_Instance_BurstBalanceLowestPointInGraph.png)
If only 350 IOPS credits are added to the `BurstBalance` credits every second, then there would be no credits left around 22:00.
This is visible since the value of the blue line is zero on the 15th of May around 22:00.

To make sure there are still credits left, I chose to have at least 20% of `BurstBalance` left at all times.
This means that the `BurstBalance` in this limited time window can never drop below 1120000 IOPS credits.

The green line representing 425 IOPS never drops below 1120000 IOPS credits.
So 425 IOPS is chosen as a safe IOPS baseline to replenish the `BurstBalance`.  

To calculate the size of the GP2 volume in GB to accommodate for this burst pattern, divide the required IOPS replenishing rate by 3, which is 141.6666 GB.

The initially configured Volume size of this RDS instance was 500GB.
Reducing the volume to 141.6666GB, slashes storage costs by another 71.8%.

## Application optimization

Costs can be further reduced by optimizing the application using the database so that less IOPS are used.
If the database contains a lot of data, evaluate whether all this data is still necessary.
Storing fewer data, or using less IOPS give you the opportunity to reduce the GP2 volume so that even more costs can be saved.

## Database logs

Consider not logging the database logs to cloudwatch to reduce Cloudwatch costs.

### Real-life use case

Our logs are still sent to cloudwatch, since the cost is minimal compared to compute and storage, and it helps for troubleshooting.

# Cost savings summary

Compute costs are reduced by 58.8% by rightsizing type and instance size.  
Compute costs are then reduced by 37% by reserving instances for one year.   
Storage costs are reduced by 71.8% by rightsizing the volume while taking BurstBalance into account.  
The compute costs were reduced by approximately 74% and the storage costs were reduced by 71.8%.  

Data transfer and log costs were rather limited, leading to an approximate cost saving of 70%.

