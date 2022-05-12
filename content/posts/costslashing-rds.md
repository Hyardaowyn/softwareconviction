---
title: "Slashing AWS RDS costs for a database instance in an enterprise context"
date: 2022-05-03T18:28:22+02:00
draft: false
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

## database instance class type
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
As mentioned before, reducing the VCPus by a factor of 2, roughly increases the CPUUtilization percentage by a factor of 2. 
Scaling down vertically to 4 VCPUs is the limit since the maximum of CPU Utilization will cross the 50% barrier more than a couple of minutes a day.
db.t4g.xlarge has 4VCPUs, so do not go smaller than db.t4g.xlarge.

![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_CPUUtilization_Before.png)
##### Freeable Memory analysis
A db.m6g.2xlarge instance has 32GB of memory. As you can see on the graph, on the 21st of March there is 25GB of Freeable Memory so approximately 7 GB is used. Therefor it is possible to estimate that 8GB of RAM is enough.
db.t4g.large has 8GB of RAM, so do not go smaller than db.t4g.large.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_FreeableMemory_Before.png)

#### Conclusions
Since CPU seems to be the bottleneck, the instance type was changed to t4g.xlarge on the 23rd of March.
The effect on the Freeable Memory and the maximum of the CPU Utilization within a 5-minute period, is shown in the graphs below.
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_CPUUtilization.png)
![MySQL RDS CPU Utilization](/MySQL_RDS_Instance_FreeableMemory.png)

#### Cost reduction
Initial compute cost = $0.608 X 24 X 30 = $437.76.
compute cost after scale down = $0.258 X 24 X 30 = $185.76
a cost reduction of 57.5%.

### An additional real life use case
At the time of writing a different MySQL RDS instance is running which ignores AWS's advice (https://aws.amazon.com/premiumsupport/knowledge-center/low-freeable-memory-rds-mysql-mariadb/) since at peak Memory usage it only has 3% Free Memory.
The monitoring shows that the swap Usage metric increases regularly, so scaling up vertically seems reasonable.
However there is no visible impact on write latency, read latency nor on CPU Utilization.
At this point it is a calculated risk which reduces costs significantly.
Keep in mind that downscaling one instance size reduces the cost by a factor 2. 
Choosing to not scale up one instance size in this case, saves a considerable amount of money.
It is very much recommended having some CloudWatch alarms in place, should the read or write latency exceed a certain arbitrary threshold.

## Multi-AZ 
RDS instances can be run with multi AZ availability. 
Choose this option if your database needs to be high available.
It will protect you from hardware failure and from an availability zone going down as a whole.
Having your RDS instance run with multi AZ availability provides faster recovery, since failover is a breeze. 
Consider disabling multi AZ when faster recovery is unnecessary which is the case for most test environments.
Running the test RDS instance in single AZ availability will cut its compute and storage cost in half.

### real life use case
The RDS instances in our test environment run in single AZ availability because the impact of having an RDS instance fail is limited.
The MySQL RDS instance that runs in production have been deployed multi AZ as the cost of downtime is rather large. 
Our test environment was already configured as single AZ and the multi-AZ configuration of the MySQL RDS instance was not changed, so no costs were saved.

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

### Which option should I pick?
Pick the option with the largest cost saving.
If both are approximately equal consider whether you are sure that you will need the RDS instance for the reserved amount of time, if there is any doubt, choose stopping and starting instances during periods in which they are not used.

### Real life use case
We do not stop unused RDS instances since both on our test and on our production environment, the database is used from 6AM up until 2AM, while also having some batch operations running between 2AM and 6AM.
Quite evidently we chose to reserve our RDS instances.
In our particular case we were running multiple instance types of the RDS t4g class. Since we cannot confidently predict the workload on our database for the coming year, we chose to reserve our RDS instances for one year with partial upfront payment, leading to a cost reduction of approximately 37% (see Reserved Instances on https://aws.amazon.com/rds/mysql/pricing/).

### Size flexibility
Bear in mind that you can also make use of size flexibility for Reserved Instances.
If you are sure about the chosen RDS instance class and the minimum instance type you need, then go ahead and reserve the minimum instance type already.
The size flexibility will allow two reserved RDS instances to count as one reserved RDS instance of one instance type higher.
The following scenario might this more clear. 
Suppose you currently have an rds.t4g.medium database running in a single AZ.
You do not know whether the database workload will increase in the coming months, but you are sure that it will not decrease.
It might be possible that the database needs to be deployed Multi-AZ in the coming months.
In this case, choose to reserve one rds.t4g.medium single AZ instance already.
If at some point in time you need to scale up to an rds.t4g.large single AZ instance, you can simply reserve another rds.t4g.medium single AZ instance. 
One reserved rds.t4g.large single AZ instance counts as two reserved rds.t4g.medium single AZ instances.
For multi AZ the same trick applies. One reserved multi AZ rds.t4g.medium instance counts as two reserved single AZ rds.t4g.medium instances.

## Allocated storage
There are multiple ways to save costs on allocated storage.
### Choose the correct storage types
There are two storage types for RDS instances:
- GP2
- IO1

The choice of storage type for RDS determines the type of EBS Volume, the hard drive of the database, attached to the RDS instance.
GP2 is great for variable workloads and should be your default option.
Only Consider IO1 when you need more IOPS than can be provided by GP2 or when there is a requirement on your database to always have some provisioned IOPS.
As of now when RDS instance storage is mentioned, a GP2 database is implied.

As mentioned before, you will pay for every availability zone in which the RDS instance is deployed so choosing a multi AZ RDS instance will increase your storage costs by at least by a factor of 2.
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

Most RDS instances have a variable workload so that GP2 is the better candidate. In what follows only GP2 volumes are considered.

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
If you have a single AZ RDS instance with 1 TB of allocated GP2 storage, it will cost you a bit more than $110 per month.
Having a multi AZ RDS instance will double the cost, since the secondary instance also has a GP2 storage of the same size attached to it.
Look at the Free Storage Space metric for the RDS instance to see how much Storage space the RDS instance actually uses.
It is tempting to reduce the storage size to what you actually use plus some margin for safety. But do not do that just yet. You will have to make sure that you have enough IOPS capacity after storage size reduction!

##### Reducing the storage size while taking IOPS into account
If IOPS does not pose a problem, then decrease the storage size to what you actually use plus some margin for safety. Autoscaling storage or rather auto increasing storage is possible, but is out of scope for this post.
If IOPS is the bottleneck, then calculate the storage size that replenishes the BurstBalance of the RDS instance fast enough, so it never gets depleted.



## application optimization

## database logs


###### footnotes
<sup>2</sup>: The Freeable Memory metric is not always accurate, consider using enhanced monitoring's Free Memory metric when The Freeable Memory drops below 15% of the configured memory of the RDS instance or when the Swap Usage metric increases regularly. 

