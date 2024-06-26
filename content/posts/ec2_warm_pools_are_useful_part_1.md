---
title: 'EC2 Autoscaling Warm Pools are very useful for looooooooong bootstrap times - Part 1'
date: 2023-08-08T5:10:21+10:00
tags:
  [
    'aws',
    'automation',
    'ec2 autoscaling',
    'ec2',
    'autoscaling',
    'autoscaling warm pools',
    'warmpools',
    'lambda function',
    'eventbridge',
    'serverless',
    'lambda',
    'cdk',
  ]
draft: false
---

## So whats the scenario?

I came up against conflicting customer requirements of minimising the impact of long application bootstrap times in an autoscaling group partnered along side the requirement of keeping their standard barebones EC2 AMIs interchangable. Essentially the customer did not want to have to deal with the bloat or overheads of maintaining a specific prebaked application AMI alongside an image building solution like EC2 Image Builder or a third party solution like Hashicorp Packer. Enter EC2 Autoscaling Warm Pools...

{{< youtube id="Q6TLWqn82J4" >}}

## What are EC2 Autoscaling Warm Pools?

EC2 Auto Scaling Warm Pools are a set of pre-intiallised instances that an autoscaling group can maintain to reduce the latency incurred through long boostrap processes. When a scale out action occurs, the autoscaling group has the ability to draw from the already bootstrapped instance capacity in the Warm Pool rather than launching a fresh new instance incurring bootstrap time.

## Why would you use Autoscaling Warm Pools?

Warm pools are super useful when you have a significant amount of bootstrapping time usually attibuted to bespoke configuration requirements of instances launched into an Autoscaling Group. Additionally using third party AMIs or image building solutions has the overeheads that someone needs to own any bespoke configuration as you build out more diverse AMIs rather just sticking to hardened standard operating environments or golden images. This becomes even more of a burden when the application team is a third party with an ops team in house. On top of the obvious benefit of faster scale-out times there is also the added benefit of cost reductions. When instances enter a warm pool they can be placed into one of three states: Warmed:Stopped, Warmed:Hibernated and Warmed:Running. So for example by keeping warm pool instances in Warmed:Stopped you can effective minimise costs. In this state you only pay for the volumes used, elastic ips associated etc. So when you keep your instances stopped or hibernated your saving on the cost of the instances themselves.

## How do Autoscaling Group Lifecycle Hooks work when using Warm Pools

So I learnt there is abit of a gotcha or side affect by design of Warm Pools, when it comes to using them alongside existing Lifecycle hooks to do bootstrapping of an application on EC2 instance launch.

Lets take a couple of example scenarios...

### Scenario 1 - A Standard Autocaling group with Lifecycle Hook applied on EC2_INSTANCE_LAUNCHING:

In a standard autoscaling group with a Lifecycle Hook applied on EC2_INSTANCE_LAUNCHING scale out. The instance first moves into Pending:Wait state, whereby it will remain in the Pending:Wait state whilst you bootstrap or configure your instance using automation usually in the form of UserData scripts or SSM automations. As such the final step in your automation following the instance bootstrap will be to call the complete-lifecycle-action API to continue or complete the lifecycle action. The autoscaling group then continues moving the instance into InService within the autoscaling group ready to serve client requests.

![Standard Lifecycle Hooks](/img/lifecycle_hooks.png 'Standard Lifecycle Hooks')

### Scenario 2 - An Autoscaling group with an existing Lifecycle Hook applied on EC2_INSTANCE_LAUNCHING alongside a Warm Pool

For an autoscaling group taking advantage of Warmpools alongside an existing EC2_INSTANCE_LAUNCHING Lifecycle Hook. You need to factor in that an instance will enter into a Pending:Wait state twice requiring triggering the complete-lifecycle-action API call **twice**.  
![Lifecycle Hooks with Warm Pools](/img/warm-pools-lifecycle-hooks.png 'Lifecycle Hooks with Warm Pools')

1. The first time you must triger complete-lifecycle-action is when the instance is initially started / launched into an ASG. Usually this is when the instance is bootstrapped with automation UserData scripts or SSM automation whilst in the Warmed:Pending:Wait. The automation finalises with the first complete-lifecycle-action API call to move the instance into Warm Pool Warmed:Running, Warmed:Stopped or Warmed:Hibernated within the Warm Pool.
2. Then you must also call complete-lifecycle-action again on any Scale Out of the autoscaling group when the instances move from a Warm pool into the ASG they goes into Pending, Pending Wait. Then ONLY after the complete-lifecycle-action API has been called the second time does the instance move to Inservice within the autoscaling group.
   Is this a bug or just the default behavior when using warm pools? My guess it has to do with the existing state machine that governs the autoscaling service control plane prior to the launch of this feature.

## So whats the solution?

## You guessed it, more automation!

The easiest way to workaround this potentially undesirable behavior without having your UserData or your SSM automation needing to call the complete-lifecycle-action twice. Is to use a Lambda function thats triggered through a Cloudwatch EventBridge Rule when an instances moves out of a Warm Pool. EC2 Auto Scaling like other AWS services supports publishing its lifecycle events directly to Cloudwatch EventBridge. As such its possible to create an EventBridge rule that filters for Warm Pool related events for example based on the instances Origin Warm Pool and Destination autoscaling group. The rule then triggers a Lambda function which calls the complete-lifecycle-action API to continue for the second time without your UserData or SSM automation scripts having to factor this in. I'll cover this Lifecycle hook automation along with overall Warm Pools in a CDK example as a follow up to this post.

## Other useful references:

Warm Pools
https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-warm-pools.html
https://aws.amazon.com/blogs/compute/scaling-your-applications-faster-with-ec2-auto-scaling-warm-pools/

EC2 Autoscaling Lifecycle Hooks
https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks-overview.html

Lifecycle Hooks with Warm Pools
https://docs.aws.amazon.com/autoscaling/ec2/userguide/warm-pool-instance-lifecycle.html

Warm Pool Patterns for Event Bridge
https://docs.aws.amazon.com/autoscaling/ec2/userguide/warm-pools-eventbridge-events.html
