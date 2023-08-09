---
title: "EC2 Autoscaling Warm Pools are very useful for looooooooong bootstrap times - Part 1"
date: 2023-08-08T5:10:21+10:00
tags: ["aws","automation", "ec2 autoscaling", "ec2", "autoscaling", "lambda function", "eventbridge", "serverless", "lambda", "cdk"]
draft: false
---

## So whats the scenario?
I came up against some a customer requirement of minimising the impact of long application bootstrap times in an autoscaling group partnered along side the requirement of keeping their standard barebones EC2 AMI interchangable.  Essentially the customer did not want to have to deal with the bloat or overheads of maintaining a specific prebaked application AMI alongside an image building solution like EC2 Image Builder or a third party solution like Hashicorp Packer.  Enter EC2 Autoscaling Warm Pools...

{{< youtube id="Q6TLWqn82J4" >}}

### What are EC2 Autoscaling Warm Pools?
https://aws.amazon.com/blogs/compute/scaling-your-applications-faster-with-ec2-auto-scaling-warm-pools/

### Why would you use Autoscaling Warm Pools?
Warm pools are super useful when you have an application that has a significant amount of bootstrapping time usually attibuted to bespoke application configuration requirements applied on instance launch into an Autoscaling Group.

### How do Autoscaling Group Lifecycle Hooks work when using Warm Pools
So I learnt there is abit of a gotcha or side affect by design of Warm Pools, when it comes to using them alongside existing Lifecycle hooks to do bootstrapping of an application on EC2 instance launch.

Lets take a couple of example scenarios:

#### Scenario 1 - A Standard Autocaling group with Lifecycle Hook applied on EC2_INSTANCE_LAUNCHING:
In a standard autoscaling group with a Lifecycle Hook applied on EC2_INSTANCE_LAUNCHING scale out. The instance first moves into Pending:Wait state, whereby it will remain in the Pending:Wait state whilst you bootstrap or configure your instance using automation usually in the form of UserData scripts or SSM automations.  As such the final step in your automation following the instance bootstrap will be to call the complete-lifecycle-action API to continue or complete the lifecycle action.  The autoscaling group then continues moving the instance into InService within the autoscaling group ready to serve client requests.

![Standard Lifecycle Hooks](/img/lifecycle_hooks.png "Standard Lifecycle Hooks")

#### Scenario 2 - An Autoscaling group with an existing Lifecycle Hook applied on EC2_INSTANCE_LAUNCHING alongside a Warm Pool
For an autoscaling group taking advantage of Warmpools alongside an existing EC2_INSTANCE_LAUNCHING Lifecycle Hook. You need to factor in that an instance will enter into a Pending:Wait state twice triggering the Lifecycle Hook **twice**.  
![Lifecycle Hooks with Warm Pools](/img/warm-pools-lifecycle-hooks.png "Lifecycle Hooks with Warm Pools")
1. The first time its triggered when the instance is started / launched and bootstrapped with automation UserData scripts or SSM whilst in the Warmed:Pending:Wait.  The automation finalise with the first complete-lifecycle-action API call to move the instance into Warm Pool Warmed:Running, Warmed:Stopped or Warmed:Hibernated within the Warm Pool.
2. Then again on Scale Out of the autoscaling group when the instances move from the Warm pool into the ASG they goes into Pending, Pending Wait and then ONLY after the complete-lifecycle-action API has been called again does the instance move to Inservice.

### So whats the solution?
### You guessed it, more automation!

The easiest way to solve this potentially undesirable behavior of your UserData or your SSM automation having to deal with calling the complete-lifecycle-action twice can easily be solved with a Lambda function thats triggered through a Cloudwatch EventBridge Rule.  EC2 Auto Scaling like other AWS services supports publishing its lifecycle event directly to Cloudwatch Event Bridge.  As such its possible to create an EventBridge rule that filters for Warm Pool related events based on the instances Origin Warm Pool and Destination autoscaling group.  This EventBridge rule then triggers a Lambda function which calls the complete-lifecycle-action API to complete the Lifecycle Action for the second time without your UserData or SSM automation scripts having to factor this in.  I'll cover this solution along Warm Pools in a CDK example as part of part 2 of  this post.

### Other useful references:
Warm Pools
https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-warm-pools.html

EC2 Autoscaling Lifecycle Hooks
https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks-overview.html

Lifecycle Hooks with Warm Pools
https://docs.aws.amazon.com/autoscaling/ec2/userguide/warm-pool-instance-lifecycle.html

Warm Pool Patterns for Event Bridge
https://docs.aws.amazon.com/autoscaling/ec2/userguide/warm-pools-eventbridge-events.html