---
title: "The effect of memory configuration on AWS Lambda's network throughput."
date: 2022-04-07T18:28:22+02:00
draft: false
---

# The effect of memory configuration on AWS Lambda's network throughput
This post will investigate the effect of various memory configurations on the perceived network throughput of AWS Lambda.
A brief introduction of AWS Lambda is given and the throughput during upload of a fixed size image is analyzed.
Finally, I will do a cost analysis and summarize the findings.
# What is AWS Lambda
AWS Lambda is AWS's implementation of a serverless function.
A short-lived container that can execute custom code.
It allows you to move fast, without having to provision a server and the maintenance burden that comes with it.
Oh, and you only pay for the time your code is actually executing.
What is there not to love?

# The Problem: Slow Throughput Speed of Lambda
AWS Lambda is probably my favorite AWS service, and I have been using it extensively for over three years.
Recently, I noticed something peculiar. 
A Lambda function that streams image content from A to B regularly took over 50 seconds to complete.

The code of this Lambda function was written in Python. The Lambda had a VPC configuration, and was configured with `256 MB` memory.

The average transferred image size is about `20 MB`, so I had expected most of the executions to end quickly. 
Let's say 2 to 3 seconds tops.
Carefully examining the metrics, I noticed that a duration of 50 seconds was not exactly exceptional. Something was definitely going on here.

The documentation on network throughput speeds of Lambda is unfortunately rather limited. This [blogpost](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-2/) briefly mentions:
> Generally, CPU-bound Lambda functions see the most benefit when memory increases, whereas network-bound see the least.

Most documentation however, is focused on the relation between Lambda memory and CPU.
Not a promising start... 
but the network throughput seemed so low that increasing the memory was worth a shot.
Increasing the memory from `256 MB` to `5120 MB` immediately resulted in a severe drop in Lambda duration. 


The configured memory was impacting the execution time, but by how much? Let's set up an experiment
## The experiment

To make the tests reproducible, I used the same `45 MB` image in every test. 
I only changed the memory of the lambda function. I ran multiple tests for each memory configuration to get a good average duration.
The average throughput is calculated based on the `45 MB` image.
### The results


| RAM (MB)	 | duration (s) | 	average throughput (MB/s) |
|-----------|--------------|----------------------------|
| 256	      | 18.10	       | 2.49                       |
| 256	      | 17.60	       | 2.56                       |
| 256	      | 19.50	       | 2.31                       |
| 1024	     | 3.00         | 15.03                      |
| 1024	     | 3.50         | 	12.89                     |
| 1024	     | 6.00         | 	7.52                      |
| 1024	     | 6.00         | 	7.52                      |
| 1024	     | 3.00         | 	15.03                     |
| 1536	     | 2.00         | 	22.55                     |
| 1536	     | 2.30         | 	19.61                     |
| 1536	     | 2.60         | 	17.35                     |
| 1536	     | 2.40         | 	18.79                     |
| 1536	     | 2.10         | 	21.48                     |
| 2560	     | 1.80         | 	25.06                     |
| 2560	     | 1.90         | 	23.74                     |
| 2560	     | 1.90         | 	23.74                     |
| 2560	     | 1.60         | 	28.19                     |
| 2560	     | 1.90         | 	23.74                     |
| 5120	     | 2.05         | 	22.00                     |
| 5120	     | 1.40         | 	32.21                     |
| 5120	     | 1.87         | 	24.12                     |

## Graphical representation
![Execution time vs Configured memory](/ExecutionTimeVSConfiguredMemory.png)
## Interpretation of the Results

One does not need to be a math-wiz to see where this is going. Increasing the memory of the AWS Lambda function increases throughput drastically.
These experimental results suggest that the network throughput of AWS Lambda is impacted by the memory configuration, briefly mentioned [here](https://docs.aws.amazon.com/lambda/latest/operatorguide/computing-power.html).
However, since increasing memory also increases the CPU, as mentioned in [the docs](https://docs.aws.amazon.com/lambda/latest/dg/configuration-function-common.html#configuration-memory-console), it's still possible that the CPU is the bottleneck.

## Cost Analysis
AWS Lambda will bill you for every GB-s you are using<sup>1</sup>. So one would expect that increasing the memory, i.e. the GB part of GB-s, increases the cost. However, this can be offset by the reduced Lambda execution time (the s part of GB-s).
By increasing the memory from `256 MB` to `1536 MB` the cost dropped because of the reduced execution time.

## Benefits
* faster execution time
* lower cost
* lower lambda timeout
* faster retry possible
# Conclusion: Increase memory to increase network throughput for Lambda
Increasing the memory of a Lambda function increases the CPU power and network throughput drastically.


###### footnotes
<sup>1</sup>: This is not completely true, you are not billed for the initialization of the Lambda function.