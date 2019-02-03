---
layout: post
title: Get Spark to use your AWS credentials file for S3
---

Spark can access files in S3, even when running in local mode, given AWS
credentials.  By default, with `s3a` URLs, Spark will [search for credentials
in a few different places](https://hadoop.apache.org/docs/current/hadoop-aws/tools/hadoop-aws/index.html#Changing_Authentication_Providers):

*  Hadoop properties in `core-site.xml`:

```
fs.s3a.access.key=xxxx
fs.s3a.secret.key=xxxx
```

* Standard AWS environment variables `AWS_SECRET_ACCESS_KEY` and
`AWS_ACCESS_KEY_ID`

* EC2 instance profile, which picks up IAM roles

However it will *not* by default pick up credentials from the `~/.aws/credentials`
file, which is useful during local development if you are authenticating to
AWS through SAML federation instead of with an IAM user.

The way to make this work is to set the `fs.s3a.aws.credentials.provider`
to `com.amazonaws.auth.DefaultAWSCredentialsProviderChain`, which will
work exactly the same way as the AWS CLI -- it will honor the AWS environment
variables as well as the credentials file with the `AWS_PROFILE` environment
variable to select from profiles.

It's not clear to me why this was not included by default, but it's typically
only an issue for individual developers running Spark in local mode for testing.
Once you're running on an EMR cluster it's a non-issue because you will likely
be running with EMRFS anyway with `s3:` URLs, and even if you use `s3a` you'd
pick up IAM roles from the EC2 instance profile.
