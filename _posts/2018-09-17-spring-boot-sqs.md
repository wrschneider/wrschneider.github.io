---
layout: page
title: Spring Boot listener for AWS SQS with Spring Cloud
---

I was surprised how little code I needed to get a Spring Boot application listening to an Amazon SQS queue.

I put a [gist on Github](https://gist.github.com/wrschneider/42407cc2ea70799362cc5b044ebcfabb) to illustrate.

The key is that when you have the right dependencies in your Maven POM, all you have to do is annotate your listener
method with `@SqsListener`:

```
@SqsListener("your-queue-name")
public void listen(DataObject message) {
    LOG.info("!!!! received message {} {}", message.getFoo(), message.getBar());
}
```

The dependency on `spring-cloud-starter-aws` takes care of initializing everything and scanning for annotated methods.

#### Command line arguments and authentication

You specify AWS credentials and region through Spring Boot properties.  I passed these as command line arguments through 
Eclipse, where I was debugging locally / not on AWS:

* `--cloud.aws.region.static=us-east-1` set my region to US East 1 (Northern VA). 
* `--cloud.aws.credentials.useDefaultAwsCredentialsChain=true` tells Spring Boot to use the AWS `DefaultAWSCredentialsChain`,
which will pull credentials from either environment vars or `~/.aws/credentials` file.
* I also set environment variable `AWS_PROFILE` for the default credentials chain to find my credentials under the correct profile.

If I were running in AWS itself, the EC2 instance metadata could have determined the region automatically, and also provided
credentials via the instance profile.

### Testing via AWS console

I used the AWS console to send test messages. The main gotcha is that if you are using JSON messages and using Spring 
to automatically deserialize JSON to your objects via `@JsonProperty` annotations, you will need to specify the message
attribute (header) `contentType` with value `application/json`.   Otherwise the conversion will fail with an unhelpful
error message like "Cannot convert from [java.lang.String] to .... for GenericMessage ...." with no indication why 
there was a failure.

There is another alternative for [reconfiguring the default Spring messaging classes](http://cloud.spring.io/spring-cloud-static/spring-cloud-aws/2.0.0.RELEASE/multi/multi__messaging.html#_consuming_aws_event_messages_with_amazon_sqs)
to ignore the `contentType` header. 
