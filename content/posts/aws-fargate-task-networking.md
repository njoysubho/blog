---
title: "AWS Fargate Task Networking"
date: 2020-03-31T15:34:39+02:00
draft: false
---

In order to explore more about deploying on ECS using terraform and github action I created a small project but as I try to deploy it on AWS ECS I found some interesting issue.
-  So I created a services in a subnet , the subnet has rule that route all internet traffic to an internet gateway. 
- The service has a simple task.
- I created all the infrastructure using terraform and   deployed using github.

 All good till now, but interesting thing happens, the task failed to startup,
  I tried to look for cloudwatch logs unfortunately nothing was there ?
  
  My first impression was may be the application failed to startup (which was unlikely as my integration test were able to ram up spring container) so I pulled the container image locally and ran container it was fine and start up. 
  
  My next impression was may be the health check timeout is too small however adjusting it to 10 second(Quite a large number the app is a very basic Micronaut app) does not solve the issue.
  
   In the mean time I was searching for container logs and I finally found them in TASK -> Stopped TASK -> Expanding the container . 
   
   The log show something interesting it seems task cannot connect to ECR endpoint. This was clearly pointing now some issue with subnet and its connectivity to internet. I start looking into security groups for the task and it was all fine permitting OUTBOUND internet traffic. Next I looked into Subnet route table here I can see it is a public subnet routing internet traffic to internet gateway. Then why task can't reach ECR ?? . Below is where the answer lies

```
1.    If you're running a task using an Amazon Elastic Compute Cloud (Amazon EC2) launch type and your container instance is in a private subnet, or if you're running a task using the AWS Fargate launch type in a private subnet, confirm that your subnet has a route to a NAT gateway in the route table.

2.    If you're running your task using an Amazon Elastic Compute Cloud (Amazon EC2) launch type and your container instance is in a public subnet, confirm that the instance has a public IP address. Or, if you're running a task using the Fargate launch type in a public subnet, choose ENABLED for Auto-assign public IP when you launch the task. This allows your task to have outbound network access to pull an image.
```
I am running my task with FARGATE launch type but didn't want to assign public IP to tasks. 
 So solution was to 
 - Deploy a NAT Gateway.
 - Create a private subnet and configure rule in that private subnet to route outbound traffic to NAT Gateway. 
 - Launch ECS service in the private subnet. Voila, now tasks can reach ECR and successfully launch.

Some Resources to learn more about FARGATE task network

https://aws.amazon.com/premiumsupport/knowledge-center/ecs-pull-container-api-error-ecr/

https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html