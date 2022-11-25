---
title: "RabbitMQ using AMQPS and STOMPWS on AWS ECS Fargate"
date: 2022-05-15T13:10:22+02:00
authors: ["Geert Van Wauwe"]
draft: true
---


# An in-depth guide on how to set up RabbitMQ on Fargate
This guide discusses how to set up RabbitMQ on AWS Fargate when using both the STOMP and AMQP protocol over TLS for the exchange of messages.

## The use case
A third party application requires RabbitMQ for messaging between clients and server.
One client uses AMQP(S) while the other uses STOMP over WebSockets.
No need for five nines of uptime.
It is not a big problem if RabbitMQ is down for a couple of minutes,
although the third party application client needs to be restarted then.
The RTO is 4 hours as long as the frequency is limited.
The message size is 4 kb maximum, and at most 1000 messages per minute need to be distributed over at most 200 clients.

## Options and tradeoffs for hosting RabbitMQ
Working in an AWS environment, 3 options come to mind:
- The AmazonMQ service
- Externally managed hosting such as CloudAMQP, IBM or StackHero
- self-hosted on AWS ECS

### Initial setup
In order to optimize for a speedy delivery of our application, CloudAMQP was chosen as an externally managed hosting service.
It did not take long to set up and little to no issues were encountered, so great value there.
However, the costs were a bit steep, so we decided to for cheaper alternatives on AWS.

### AmazonMQ service
AmazonMQ unfortunately does not support STOMP over websockets yet. 

### AWS Fargate
The comfortable RTO makes self-hosting possible.
No need for high bandwidth so Fargate can be chosen over ECS on EC2.
It has the highest potential for cost saving.

Unfortunately not a lot of documentation can be found on setting up Rabbitmq on AWS, which has the risk of increasing development cost significantly.

### CloudAMQP
An already working setup is already present which allows constant evaluation of any chosen option.




