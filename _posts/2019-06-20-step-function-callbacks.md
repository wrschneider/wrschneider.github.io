---
layout: post
title: Asynchronous callbacks with AWS Step Functions and Lambda
description: how to build an AWS Step Function workflow with serverless Lambda functions and asynchronous waitForTaskToken callbacks
---

AWS Step Functions recently [added support for callback patterns](https://aws.amazon.com/about-aws/whats-new/2019/05/aws-step-functions-support-callback-patterns/)
for long-running tasks.

This addresses one of the weaknesses of AWS Step Functions.  Previously, if you wanted to have
a workflow step invoke a long-running process on an EC2 instance via SSM, the main workaround is
to have a polling cycle in the workflow.

* First lambda task invokes SSM (or other long-running process)
* Second lambda task checks if SSM command has completed
* Loop back to polling task if still in progress

There are two problems with this:

* Clutter in the workflow - harder to understand what's actually happening
* Extra charges for lambda executions and state transitions that aren't doing anything useful

The new callback workflow is a significant improvement.  Instead the workflow would be something like

* First lambda invokes SSM, passing it a task token
* Workflow pauses until `SendTaskSuccess` API called with task token
* When SSM command completes, call `SendTaskSuccess` to resume workflow

Here is an example of what a step machine definition might look like:

```json
{
  "StartAt": "AsyncLambda",
  "States": {
    "AsyncLambda": {
      "Type": "Task",
      "Resource":"arn:aws:states:::lambda:invoke.waitForTaskToken",
       "Parameters":{
            "FunctionName":"async_lambda",
            "Payload":{
               "token.$":"$$.Task.Token"
            }
         },
      "Next": "AsyncDone"
    },
    "AsyncDone": {
      "Type": "Pass",
      "Result": "Async func completed!",
      "End": true
    }
  }
}
```

and the corresponding Lambda

```python
def lambda_handler(event, context):
    token = event['token']
    ssm = boto3.client('ssm')
    ssm.send_command(
            InstanceIds=[....],
            DocumentName="AWS-RunShellScript",
            Parameters={'commands': [f'your-command.sh --token {token}']}
            )
```

The shell script on the EC2 instance might look like

```bash
token=# parse from args

./some-long-running-process.sh

# signal success on completion
aws stepfunctions send-task-success --task-token $token --task-output '{"status": "success"}'

```
