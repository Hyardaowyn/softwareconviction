---
title: "Slashing AWS RDS costs for a database instance in an enterprise context"
date: 2022-05-03T18:28:22+02:00
draft: true
---

# A step by step guide on how to reduce costs for an RDS instance
Some obvious and some lesser-known cost savings for AWS RDS and the order in which to apply them are discussed.
Some cost savings come with deliberate tradeoffs and depend on the context in which the RDS instance operates.
To make it more concrete these careful considerations are applied on a real life use case; A MySQL RDS instance.

Finally, an overview of all the cost savings is presented for the MySQL RDS instance.

# What is AWS RDS
AWS RDS is AWS's relational database service.
It supports proprietary and open source database engines.
Amazon Aurora, MySQL, MariaDB, Oracle, SQL Server, and PostgreSQL are the current options to choose from.
This service should be considered if you want to have a relational database without wanting to deal with maintenance, durability, disaster recovery and high availability.

## Real life use case RDS Instance
To make things more concrete a real life use case is considered.
A MySQL RDS instance. The instance is used 6 days a week and experiences its highest load during extended business hours.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_Typical_Daily_CPUUtilization.png)
Housekeeping is performed every night which explains the small peak around 1:00AM.

# Cost Savings
Using AWS RDS can become costly rather quickly. 
Some options in the RDS configuration should be a deliberate choice tailored to the context in which the instance operates.
An oversight during setup or a failure to reevaluate a configuration can increase your RDS bill by multiple factors of 2.

## AWS RDS
The best way to reduce AWS RDS costs is by not using AWS RDS at all!
EC2 is perfectly suited to host a relational database and will cost you approximately 50% less.
If you do not care much about high availability, disaster recovery, security or performance then this is definitely the way to go.
Most businesses however do care, so I would not consider this a viable option anymore in 2022 except for maybe on a hobby project.

## Database instance class, type and size
When starting with AWS RDS consider which instance class you want to use.
Consider the burstable performance class (the t instances), when you have a non-constant workload over 24 hours.
For example when the database is almost exclusively used during the day.
Consider the memory optimized class (R,x and z instances) when you have a read heavy database workload.
Consider the general purpose class for other workloads.

## Database instance class type
Choosing the right database instance class type and size is paramount in reducing AWS RDS costs.
If you are not quite sure how much Memory or CPU the instance will need it is best to over-provision a little and start with a rather large instance class type.
Once a utilization baseline has been established, you can consider scaling down the instance class type vertically.
To scale down the instance class type confidently, check both CPU and Freeable memory metrics.
Scale down when the maximum of the CPU Utilization metric is below 50% for more than 99.9% of the time and when the currently used memory (instance class type memory minus freeable memory) is more than 95% of the memory of the targeted scaled down database instance class type.
Reducing the VCPus by a factor of 2, roughly increases the CPUUtilization percentage by a factor of 2. 
This heuristic can help you estimate how much you can downscale your database instance and when you would cross the 60% CPU Utilization limit more than a couple of minutes a day.
Changing the instance class will always lead to some downtime so schedule it properly.
### Real life use case
Initially a general purpose instance (db.m6g) of size `2xlarge` was chosen for the relational database.
Since the database is primarily used during extended business hours, the instance class burstable performance (db.t4g) seems the most appropriate choice.
#### Analyzing the metrics
##### CPU Utilization analysis
A db.m6g.2xlarge instance has 8 VCPUs. As you can see on the graph, on the 21st of March, the maximum of the CPU Utilization within a 5-minute period, crosses 30% more than a couple of times.
As mentioned before, reducing the VCPUs by a factor of 2, roughly increases the CPUUtilization percentage by a factor of 2. 
Scaling down vertically to 4 VCPUs is the limit since the maximum of CPU Utilization will cross the 50% barrier more than a couple of minutes a day.
db.t4g.xlarge has 4VCPUs, so do not go smaller than db.t4g.xlarge.

![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_CPUUtilization_Before.png)
##### Freeable Memory analysis
A `db.m6g.2xlarge` instance has `32GB` of memory. 
As you can see on the graph, on the 21st of March there is `25GB` of `Freeable Memory` so approximately `7GB` is used.
Therefor it is possible to estimate that `8GB` of RAM is sufficient.
db.t4g.large has 8GB of RAM, so do not go smaller than db.t4g.large.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_FreeableMemory_Before.png)

#### Conclusions
Since CPU seems to be the bottleneck, the instance type was changed to `t4g.xlarge` on the 23rd of March.
The effect on the `Freeable Memory` and the maximum of the `CPU Utilization` within a 5-minute period, is shown in the graphs below.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_CPUUtilization.png)
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_FreeableMemory.png)

#### Cost reduction
Initial compute cost = $0.608 X 24 X 30 = $437.76.
compute cost after scale down = $0.258 X 24 X 30 = $185.76
a cost reduction of 57.5%.

### An additional real life use case
At the time of writing a different MySQL RDS instance is running which ignores AWS's advice (https://aws.amazon.com/premiumsupport/knowledge-center/low-freeable-memory-rds-mysql-mariadb/) since at peak Memory usage it only has 3% Free Memory.
The monitoring shows that the swap Usage metric increases regularly, so scaling up vertically seems reasonable.
However, there is no visible impact on write latency, read latency nor on CPU Utilization.
At this point it is a calculated risk which reduces costs significantly.
Keep in mind that downscaling one instance size reduces the cost by a factor 2. 
Choosing to not scale up one instance size in this case, saves a considerable amount of money.
It is very much recommended having some CloudWatch alarms in place, should the read or write latency exceed a certain arbitrary threshold.

## Multi-AZ 
RDS instances can be run with multi-AZ availability. 
Choose this option if your database needs to be high available.
It will protect you from hardware failure and from an availability zone going down as a whole.
Having your RDS instance run with multi-AZ availability provides faster recovery, since failover is a breeze. 
Consider disabling multi-AZ when faster recovery is unnecessary which is the case for most test environments.
Running the test RDS instance in single-AZ availability will cut its compute and storage cost in half.

### real life use case
The RDS instances in our test environment run in single-AZ availability because the impact of having an RDS instance fail is limited.
The MySQL RDS instance that runs in production have been deployed multi-AZ as the cost of downtime is rather large. 
Our test environment was already configured as single-AZ and the multi-AZ configuration of the MySQL RDS instance was not changed, so no costs were saved.

## Stopping unused RDS instances versus Reserved Instances
More costs can be saved by either stopping RDS instances during periods in which they are not used but this has to be carefully compared to reserving instances.
### unused RDS instances
Look at the usage patterns of your RDS instances.
If the database is only used during business hours or extended business hours, consider stopping the RDS instance when it is not used to save on compute costs.
This can be done by triggering an AWS Lambda Function at certain times.
AWS will still charge you for the allocated storage of the RDS instance, but you will not have to pay for the compute part of the RDS instance costs.
If your RDS instances is only used 5 days a week from 9AM until 5 PM then you can stop the RDS instance at 5PM and start it again at 9AM on weekdays.
The RDS instance will only be used 5*8 hours a week, which is about 24% of the time, so you can save approximately 76% on RDS compute costs.

### Reserved instances
Once you have decided upon which instance class to use, you should look into reserved instances for AWS RDS.
Reserved instances allows you to save up to almost 70% of your costs compared to the on-demand rate.
The cost saving is heavily class type dependent. 

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
Stopping and starting the RDS instance in our production environment because the database is used most of the time.
We did not stop the MySQL RDS instances in our test environment, although this probably would have been more cost-effective.
Our sprint backlog is already filled to the brim, so it would have probably taken a couple of weeks in order to work on stopping and starting RDS instances.
Reserving the instances is just a couple of clicks, so we decided to reserve our production and test instances for one year.
In our particular case we were running multiple instance types of the RDS `t4g` class.
We cannot confidently predict the workload on our database for years to come. Hence, reserving RDS instances for one year with partial upfront payment seems the best option for now.
This leads to a cost reduction of approximately 37% (see Reserved Instances on https://aws.amazon.com/rds/mysql/pricing/).


Regarding size flexibility. Once the RDS instance class has been decided upon, you should reserve the minimum expected baseline.
In our AWS account there are 3 test MySQL RDS instances and 3 production MySQL instances.
All of them are primarily used during extended business hours but the loads differ.
Since all access patterns are similar the burstable database instance family `db.t4g` is chosen.
The currently experienced baseline load for a test environment is covered by one `t4g.medium` (single-AZ).
The currently experienced baseline load over the three production environments is covered by 1 `t4g.medium`, 1 `t4g.large` and 1 `t4g.xlarge`. 
All the production environments utilize a multi-AZ configuration.

By making use of size flexibility it is not that difficult to calculate the amount of t4g.medium single-AZ instances necessary.
You can calculate the current baseline summing up all database instance equivalent baselines.
1 `t4g.xlarge` (multi-AZ) = 2 `t4g.large` (multi-AZ) = 2 ( 2 `t4g.medium` (multi-AZ)) = 4. ( 2 `t4g.medium` (single AZ)) = 8 `t4g.medium` (single AZ)
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
You can check whether most of the allocated storage of an RDS instance is used by comparing its Free Storage Space metric to its storage size.
It is tempting to reduce the storage size to what you actually use plus some margin for safety.
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
GP2 baseline IOPS = maximum of 3 IOPS/GB * storage size in GB and 100 IOPS = MAX(3*storageSizeInGB,100) IOPS
The minimum amount of IOPS for a GP2 volume is thus 100 IOPS.

Most RDS instances have a variable workload so that GP2 is the better candidate.
In what follows only GP2 volumes are considered.

#### RDS instances backed by GP2 storage.
#####  An example on how to calculate the IOPS baseline
This is relevant for small volumes as shown by the following example:
Given an RDS instance with 25 GB of allocated storage, the calculated baseline IOPS is the maximum of:
3 IOPS/GB * storage size in GB = 3 IOPS/GB * 25 GB = 75 IOPS
and 
100 IOPS
Since 75 IOPS is smaller than 100 IOPS, the baseline IOPS for this RDS instance will be 100 IOPS.
##### Storage costs
Storage costs are a bit higher than $0.11 per Gigabyte per month, depending on the region.
If you have a single-AZ RDS instance with 1 TB of allocated GP2 storage, it will cost you a bit more than $110 per month.
Having a multi-AZ RDS instance will double the cost, since the secondary instance also has a GP2 storage of the same size attached to it.
Look at the Free Storage Space metric for the RDS instance to see how much Storage space the RDS instance actually uses.
It is tempting to reduce the storage size to what you actually use plus some margin for safety. But do not do that just yet.
Make sure you have enough IOPS capacity after storage size reduction!

##### Reducing the storage size while taking IOPS into account
If IOPS does not pose a problem, then decrease the storage size to what you actually use plus some margin for safety.
Autoscaling storage or rather auto increasing storage is possible, but is out of scope for this post.
If IOPS is the bottleneck, then calculate the storage size that replenishes the BurstBalance of the RDS instance fast enough, so it never gets depleted.
Here is a spreadsheet (https://docs.google.com/spreadsheets/d/e/2PACX-1vRjkTJQIaRGKEuXjo4Bn3imwopOW_1Xx08jeLz8Xl-BYxyXC3gSlugFsRBrtKXTOwngI_nxq5hq_HZ6/pub?output=ods) that you can use to import your current WriteIOPS and ReadIOPS metrics.
Then you can play around with the estimated IOPS to see the effect on the BurstBalance credits.

##### Real life use case
The ReadIOPS and WriteIOPS in the spreadsheet are the production ReadIOPS and WriteIOPS


## application optimization

## database logs


###### footnotes
<sup>2</sup>: The Freeable Memory metric is not always accurate, consider using enhanced monitoring's Free Memory metric when The Freeable Memory drops below 15% of the configured memory of the RDS instance or when the Swap Usage metric increases regularly. 

