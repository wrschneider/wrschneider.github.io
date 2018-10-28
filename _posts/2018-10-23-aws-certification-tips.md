---
layout: post
title: Five tips for AWS certification exam
---

I just passed the exam for AWS Solution Architect - Associate level.  I wanted to share some observations and tips 
from my personal experience with the exam.  (Disclaimer: this is only my personal experience so YMMV.)   

One general observation is that most of the questions follow the pattern of "here's a problem; which AWS services 
would you choose for your solution?"  Also, I felt like most of the things you need to know for the test are useful to
know if you're working in AWS anyway, so I don't feel like I spent a lot of effort on test prep for its own sake.

More specific observations:

1. Security groups: There were multiple variations of the same question about security groups.  You should know the typical
usage patterns: EC2 instances only allow ingress (inbound connections) from the security group attached to an ELB, RDS 
only allows ingress from the security group attached to the EC2 instance, etc.  The general idea here is leveraging 
security groups to enforce least-access.

2. Regions vs. Availability Zones: this was another recurring theme.  The key points to know is that regions are 
not a solution for availability, and some solutions for high-availability are limited to AZs within a single region. Two
main examples are ELB, and multi-AZ RDS.  Multi-region is a solution for performance/latency, rather than availability.

3. S3 storage classes (IA, RRS, Glacier etc.): This is a popular topic for the cost optimization, so good to know these well.

4. Favor AWS-native solutions whenever possible: this is a good idea in general anyway.  The less custom code and 
tooling you have to maintain, the better. 

5. Time: The type of questions on the test mostly do not benefit from extensive thought.  If the correct answer isn't obvious
right away, move on.  If it's obvious which answers are wrong, eliminate them and guess from the remaining options, and 
flag the question to go back later.  Sometimes a subsequent question will help you choose between the remaining options,
especially if you just can't remember the exact name for something.
