--- 
layout: post
title: EC2 proxy to RDS for a static IP address
---

RDS instances in AWS do not get a static IP address.  This is usually a good thing, not a 
problem.  This provides flexibility to preserve availability while the physical RDS host may shift around for
resizing, or failing over to a different availability zone (AZ).  In either case, clients 
connect to RDS by hostname, and AWS magically updates the hostname to point at the IP address for the currently 
active host. 

The only time this creates a challenge is when you want to connect to RDS from 
a private/corporate network and have to update firewall or VPN tunnel configuration to allow connections to 
RDS. If this isn't an issue for you, you can stop reading this.

The problem is firewall rules, VPN tunnels, and NAT rules all work on IP addresses, not hostnames.  You can't
configure your firewall to unblock traffic to an RDS instance if you don't know its IP address.  

The workaround I found was to put an EC2 server in front of RDS as a TCP proxy.  You can give a static IP address
to an EC2 instance with the `PrivateIpAddress` property of `AWS::EC2::Instance`, or with an
`AWS::EC2::EIPAssociation` resource  for a static publicly-routed IP.  Then you use that EC2 instance to forward
traffic on to the RDS instance by hostname.  The EC2 instance's IP address then becomes the database's static IP for
firewall purposes.

There's lots of different ways you can forward traffic from EC2 to RDS.  You can pick whichever one best suits you:

- `socat`: e.g., `socat TCP-LISTEN,[port],fork,reuseaddr TCP:[hostname]:[port]`. 
  - Pros: simple and convenient, easy to install with `yum install socat`.  
  - Cons: not widely known; forks a process per connection so not good for high volume

- `haproxy`
  - Pros: Robust, scalable
  - Cons: AWS Linux packages [do not include latest version with runtime DNS resolution](https://stackoverflow.
  com/questions/37520737/get-or-install-haproxy-1-6-on-amazon-linux-only-comes-with-1-5-in-epel); must be built 
  from source, and even then requires some contortions to resolve hostnames at runtime.

- `nginx`
  - Pros: you might already be using it
  - Cons: overkill for a port forwarder
  
- `ssh` port forwarding
  - Pros: widely understood, ssh/sshd already installed by default
  - Cons: requires establishing and authenticating an SSH connection, which is overkill when you *only* 
  want port forwarding

*On security group setup*: Put the EC2 instance and RDS instance in two different security groups, and then those 
security groups can refer to each other.  This is a perfect use case for  CloudFormation's 
`AWS::EC2::SecurityGroupIngress` and `AWS::EC2::SecurityGroupEgress` resources ("typically to allow security groups 
to reference each other").   Since you don't know the RDS instance's IP address, you can refer to 
the RDS instance's security group.  The EC2 security group would have an egress rule to RDS security group and vice 
versa.  

It's a good idea to otherwise lock down the EC2 security group.  The EC2 instance should only allow outbound 
access to the target RDS instance, and DNS for hostname resolution.  The RDS instance should only allow inbound access through the EC2 proxy.

*Other CloudFormation tips*: Keep the EC2 instance in a stack by itself so it can be rebuilt 
independently.  You can make cross-stack references to the RDS instance from the EC2 stack.  Also, you can 
put a script in the `UserData` property in EC2 to inject the RDS hostname (from the cross-stack reference)
into Upstart config files (`/etc/init/your-proxy-service.conf`) so your proxy service will start automatically on 
boot and refer to the correct hostname.
