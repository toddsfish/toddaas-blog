---
title: 'EC2 Autoscaling Warm Pools are very useful for looooooooong bootstrap times - Part 2'
date: 2024-04-04T5:10:21+10:00
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
    'gateway load balancer',
    'firewalls',
    'cloud development kit',
    'autoscaling warm pools',
    'warmpools',
    'cdk construct',
    'construct hub',
  ]
draft: false
---

Overdue for Part 2 so thought I'd include some of my learnings along the way...

## Why Warm Pools again?

Taking a step back to networking for a brief second and Warm pools from [Part 1]({{< ref "ec2_warm_pools_are_useful_part_1.md" >}}). Warm Pools are very useful for scaling network appliances for example a set of firewall appliances in an Autoscaling Group behind a Gateway Load Balancer (GWLB). This is simply because notoriously network appliances often take a long time to boot getting all their bootstrapping and inspection, IPS/IDS jazz together. Therefore by using Warmpools a network appliance can be quickly placed in service in the Autoscaling Group, without the necessity of overprovisioning the Autoscaling Group with running instances incurring unnecessary costs. With that said if your using Warmpools alongside Lifecycle Hooks you need to take into account the additional overheads of automation to move instances from the Warmpool to the Autoscaling Group. This post covers the CDK Construct I wrote for Warmpools to extend the existing functionality of the default [CDK WarmPool class](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_autoscaling.WarmPool.html) to support this requried automation for use with future Autoscaling Groups backed by Firewall appliances or EC2 instances using Lifecycle Hooks. CDK constructs allow you to abstract, reuse, repeat.

## CDK Constructs

I've been using the AWS Cloud Development Kit (CDK) for a few years now since version 1 (now version 2) predominately for proof of concepts, testing AWS services and as part of automating my troubleshooting workflow in replicating customer environments. Mainly the constructs I've been using are the AWS service constructs. That said as I've got into using the CDK more and more. I've found myself using other Construct libraries which speed up implementation of existing best practice patterns [practice patterns](https://docs.aws.amazon.com/solutions/latest/constructs/welcome.html) or just mashing together AWS service constructs with abstractions though Open Source constructs available via [Construct Hub](https://constructs.dev/). CDKs ability to share abstactions of cloud infrastructure underpinned by around a modern programming language is what makes it great. AWS has also opensourced the Construct Hub https://github.com/cdklabs/construct-hub itself. for the use case of a private construct library within an organisation to share reuseable IaC constructs and patterns.

Now onto the CDK constructs, when it comes to the AWS service Level 2 (L2) constructs whilst they are supposed to operate a higher level of abstraction i.e. not require initialisation of the same properties of a CFN resource. L2 constucts do not cater to all usaage scenario. This means at times they often represent resources similar to the lowest level abstraction of Level (L1) constructs basically the raw CFN resources and resource properties. This is where the composition over inheritance principle with CDK allows you to extend an existing CDK constructs functionality to include other constructs and cater to custom use cases. For example in automating Autoscaling groups using Warm pools to move instances from the Warm pool into service automatically, something which the default L2 [CDK WarmPool class](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_autoscaling.WarmPool.html) does not do by default.

{{< youtube id="pnFisVo40nk" >}}

## Projen

What I discovered along the way when wanting to share my construct with the CDK community. Is that there is some barrier to entry (contribute) when it comes to meeting the requirements to have a a CDK Construct in Construct Hub. To add it was a manual process to release changes or updates to my construct whilst meeting these requirements and then push these changes to NPM which are synced to Construct Hub. This is where a tool called Projen https://projen.io/ helped alot. Projen does a lot of the heavy lifting required in terms of scaffolding a consumable CDK construct or any other supported project. This includes generating CI/CD pipelines based around Conventional Commits https://www.conventionalcommits.org/en/v1.0.0/ in the form of Github Actions workflows. For example a release workflow for validating a CDK construct and releasing the construct to the NPM public repositories conforming to Construct Hub (https://constructs.dev/contribute) requirements. Projen also has a workflow for keeping any code dependancies nicely updated automatically through PRs on the repository.

## My Warm Pool CDK Construct

So how does my Warm pools construct work what does it do? Essentially the the construct extends [CDK WarmPool class](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_autoscaling.WarmPool.html) with Lambda function thats triggered through a Cloudwatch EventBridge Rule when an instances moves out of a Warm Pool. EC2 Autoscaling like most AWS services publishes its events to Cloudwatch EventBridge. As such its simple to create an EventBridge rule that filters for Warm Pool origin related events based on the autoscaling group then trigger the Lambda calling complete-lifecycle-action and placing the instance in service.

[![View on Construct Hub](https://constructs.dev/badge?package=%40pandanus-cloud%2Fcdk-autoscaling-warmpool)](https://constructs.dev/packages/@pandanus-cloud/cdk-autoscaling-warmpool)

## A CDK App Example

Heres a simple CDK app using the my Warm Pool Contruct.
https://github.com/toddsfish/warmpools-test
You can also use this example CDK app if you just want to play around with the behaviour of Autoscaling Warm Pools in general.
