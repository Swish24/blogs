---
title: 'Unlocking Automation: Delivering S3 Event Alerts to Discord or Slack Using AWS Lambda'
slug: automated-S3-event-alerts
tags: cloudformation, python, lambda, aws, AWS Certified Solutions Architect – Professional
domain: blog.dvsn.ai
subtitle: Automated monitoring alerts for S3 to your custom endpoint
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1708675373550/gsoJUDQP2.png?auto=format
ignorePost: false
hideFromHashnodeCommunity: false
seoTitle: 'Automated S3 Event Alerts: Delivering to Discord or Slack with Webhooks & Lambda'
seoDescription: 'Discover the power of automation as we guide you through setting up S3 event alerts seamlessly. Learn how to deliver alerts to Discord or Slack using webhooks, Lambda, and your own custom endpoints. Elevate your AWS S3 monitoring game with step-by-step instructions and insights into optimizing your workflow'
enableToc: true
saveAsDraft: true
---

While preparing for my AWS Solutions Architect Professional certificate, there was mention of event delivery using SNS topics.

AWS SNS (Simple Notification Service) is typically used to deliver alerts such as emails, push notifications and text messages.

But maybe, you wanted to deliver a notification to a webhook endpoint such as [Slack](https://api.slack.com/messaging/webhooks) or [`Discord`](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)? Maybe even your own custom endpoint.

The AWS Services used in this lab:

* AWS S3
    
* AWS SNS (Simple Notification Service)
    
* AWS SQS (Simple Queue Service)
    
* AWS Lambda
    
## Code resources
The link to the CloudFormation stack can be found here on Github:  
[CloudFormation-Scripts/s3-sns-alerts-lambda at main · Swish24/CloudFormation-Scripts (github.com)](https://github.com/Swish24/CloudFormation-Scripts/tree/main/s3-sns-alerts-lambda)

You will need a Slack, Discord or custom webhook url to successfully deploy the stack

Slack and Discord have some easy-to-follow documentation on creating a webhook for their respect platforms. For your own webhook endpoint, the payload is in JSON format.

Slack: [Sending messages using incoming webhooks | Slack](https://api.slack.com/messaging/webhooks)

Discord: [Intro to Webhooks – Discord](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)

# Deploying the CloudFormation Stack

CloudFormation is a method of Infrastructure as code, there are alternatives such as [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html), which allows you to create AWS resources within your preferred programming language.

Navigate to the CloudFormation console and be sure you have selected your desired AWS region. This is the region where all of the resources will be created. Select "Create Stack".

Within this menu we are going to be sure "Template is ready" is selected, and we are going to "Upload a template file"

Once your file is upload it should show the S3 url, we do not need the s3 url, but you can find the template within that S3 bucket if you are looking for it in the future.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1707616979606/cab88a80-da71-4187-b8b2-b5ec2dd944db.png align="center")

Stack name will be the name of the CloudFormation Stack

S3BucketName will be the name of the s3 bucket you are going to create to monitor for events.

Webhook type can be of Discord, Slack, Custom

Webhook url will be the url of your discord, slack or custom endpoint webhook

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708665592718/t_lSd89ec.png?auto=format align="center")

Select next, then next once again on the following page.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708665765140/8d3d68fc-d2b9-4d7b-a670-54665c11ff40.png align="center")

You will be required to acknowledge that this stack will be creating IAM roles within your account, then hit submit to deploy the stack.

Within a couple minutes, you should see a CREATION\_COMPLETE, and all our resources will be created.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708667226296/adeedc19-d49d-4a83-8d8c-be9e5d45e4c9.png align="center")

## Testing Notification Delivery

Now, we will test out the alert delivery.

Navigate to the resources tab on your newly created CloudFormation stack, this will allow us to see the created resources, and navigate to them within the console.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708667338615/df2058de-df7b-4d59-a000-3fae39c5c223.png align="center")

Click your S3 bucket to navigate to the S3 console, we are going to upload a file to trigger the alert.

Select the upload button

Select Add files, then navigate to the "test.yml file" you downloaded from the repo or use any file from your computer.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708667792515/9f984662-ae89-4c4c-a480-8e6f6093e4f0.png align="center")

Before we take a look at our delivered alert, let's take a look at our python code within our lambda to see what information will be delivered to our webhook.

Head back to your CloudFormation stack, and select your Lambda Function

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708668284235/aecb9b60-eb3d-4560-8906-25abbba67d5c.png align="center")

Select the code tab, and let's look at the python code snippet for the Discord Alert.

```python
site == 'Discord':
      webhook = webhookUrl
      embedData = {
          "title": eventName,
          "color": color,
          "fields": [
              {
                  "name": "Event Time",
                  "value": f"<t:{unix_timestamp}>",
              },
              {
                  "name": "IP Address",
                  "value": sourceIPAddress,
              },
              {
                  "name" : "Bucket Name",
                  "value": s3BucketName,
              },
              {
                  "name": "Object Name",
                  "value": objectName,
              },
          ],
          "footer": {
              "text": "Lambda S3 Monitor"
          },
          "timestamp": eventTime
      }
```

Here we can see that we are sending a few fields, Event Time, formatted to local OS time, IP Address where the S3 action was performed from, the name of the bucket, and the name of the object.

### Validating Notification Delivery

Here we can see the alert was delivered to discord, with the payload data we confirmed within the code.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708672201903/70c2ee76-e228-45ad-894f-ffd65b82c6b2.png align="center")

Is there another way to confirm the alert was delivered?

Within our lambda function, we are going to click the monitoring tab to access logs within [CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_GettingStarted.html) to see our function execution.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708670326546/92026c5d-1f87-408a-9eac-d7b8fc30b7a9.png align="center")

Within the CloudWatch console, select the latest log stream

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708670379678/f42a488f-49e6-49de-bb17-cd6d79758658.png align="center")

Here we can see our lambda did execute, and logged "Discord", with no errors logged, and only a single execution.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708672902055/404bdf53-d44d-4b23-a75b-dfec73fe716f.png align="center")

Lambda will continue to retry if there was a delivery failure or an execution of your lambda. Be sure to return a successful response within your lambda functions!