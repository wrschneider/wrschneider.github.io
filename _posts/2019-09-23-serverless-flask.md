---
layout: post
title: Serverless Flask with Terraform
description: deploy a Python Flask app to Lambda, and manage the AWS resources with Terraform
---

I followed [Andrew Griffith's blog post](https://andrewgriffithsonline.com/blog/180412-deploy-flask-api-any-serverless-cloud-platform/) to
deploy a serverless Flask app, with Lambda and API Gateway, managing the AWS resources with Terraform.

One gotcha from the article: I didn't read it carefully enough the first time and missed the part about Python 3 support.  If
I had used the author's fork (`flask-lambda-python36`) I would have saved myself troubleshooting "is not JSON Serializable"
errors.

The general idea is replacing `Flask` with `FlaskLambda`, which wraps the underlying `Flask` object.  During local
development, requests behave exactly like they would with vanilla Flask.  When deployed to Lambda, conditional logic in the
`FlaskLambda` detects it is getting a request from API Gateway rather than directly from a browser, and unpacks the event accordingly.

I like this approach for a few reasons:

* Low-friction local development: You can run the same Python codebase locally, serving up the same Flask routes through a local HTTP
listener.  Flask will handle routes the same way as they will be handled in AWS.  You don't have to re-deploy anything to AWS to
test.   Local development works the same as plain old Flask development, no differently than if you were planning to deploy
to an EC2 instance or Docker container.

* Consistency: Frameworks like Zappa and Serverless are great, but they also manage your infrastructure for you. If you
are managing other aspects of your infrastructure in Terraform (security groups, IAM roles etc.) it's nice to have your
Lambdas managed in the same place and able to reference those resources directly.  I haven't worked with the AWS Serverless
Application Model much either, but I'd imagine it would have similar challenges integrating with Terraform-managed resources.

* Maintainability: One approach I've seen is not using Flask, but rather treating each route as a distinct Lambda with a distinct
API gateway resource.  This is tricky to maintain over time as it ends up requiring touching Terraform every time you add a route.
It also means you have to maintain a minimal Flask app to support local development, where you have Flask routes wrapping the
Lambda handlers.

I suppose this ought to be possible with Django as well; maybe there is some way to use only the Lambda handler from Zappa but
not use it for deployment?
