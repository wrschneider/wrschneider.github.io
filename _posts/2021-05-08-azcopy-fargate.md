---
layout: post
title: Run azcopy from AWS Fargate
description: Get tokens from Fargate endpoint for calling S3 services
---

Microsoft provides the `azcopy` tool for copying data between Azure storage accounts and AWS S3.  If you're otherwise serverless or
fully containerized, and don't already have an EC2 instance up, it makes sense to run `azcopy` in a Fargate task.

There are plenty of solutions out there for getting `azcopy` to work in a Docker container.  The only thing that is not totally
straightforward is getting AWS credentials to call S3.  For whatever reason, [`azcopy` doesn't natively support getting credentials from
AWS profiles, EC2 instance profiles, or ECS task roles](https://github.com/Azure/azure-storage-azcopy/issues/1341), only environment variables.

So, the workaround I came up with is to wrap `azcopy` in a shell script that gets the credentials from the Fargate task metadata endpoint:

```bash
response=$(wget -O - 169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI)

export AWS_ACCESS_KEY_ID=$(echo "$response" | jq -r '.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo "$response" | jq -r '.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo "$response" | jq -r '.Token')

azcopy $@
```

You could then `COPY` this script in your Dockerfile, as a wrapper around `azcopy`.  You would also need to have `wget` and `jq` available in your 
Docker image.
