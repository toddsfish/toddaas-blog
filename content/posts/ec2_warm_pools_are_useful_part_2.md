---
title: 'EC2 Autoscaling Warm Pools are very useful for looooooooong bootstrap times - Part 2'
date: 2023-08-09T5:10:21+10:00
tags:
  [
    'aws',
    'automation',
    'ec2 autoscaling',
    'ec2',
    'autoscaling',
    'lambda function',
    'eventbridge',
    'serverless',
    'lambda',
    'cdk',
    'cloud development kit',
    'autoscaling warm pools',
  ]
draft: true
---

In the previous [part 1]({{< ref "ec2_warm_pools_are_useful_part_1.md" >}} "part 1") of this series of posts on EC2 Autoscaling Warm Pools. I discussed [the need for automation to call complete-lifecycle-action twice]({{< ref "ec2_warm_pools_are_useful_part_1.md#scenario-2---an-autoscaling-group-with-an-existing-lifecycle-hook-applied-on-ec2_instance_launching-alongside-a-warm-pool" >}} "the required for customer automation and scripts to call complete-lifecycle-action twice") if your using EC2 Autoscaling Warm pools. This part 2 post covers the most logical workaround to the somewhat undesired behaviour if you've chosen to use Autoscaling Warm Pools. That is using a Lambda function alongside Cloudwatch EventBridge filtering for Warm Pool related events based on an instances Origin Warm Pool and the destination Autoscaling Group. This EventBridge triggered Lambda workaround saves having to touch existing EC2 UserData scripts or SSM automations to cater for the additional complete-lifecycle-action call.
