---
layout: post
title: New AWS SSM feature to tunnel SSH with port forwarding support
description: A new AWS SSM feature allows SSH connections to be tunneled, eliminating the need for a bastion host
---

AWS SSM already had a "session manager" feature that allowed users to get command prompts through a web browser.  The big
advantage this had over providing an SSH bastion host is that SSM is covered by the same governance context as other AWS services:
authentication and authorization via IAM, with audit via CloudTrail.

SSM session manager only solves for a command prompt, though.  It doesn't give you a way
to establish connectivity to resources in private subnets for development purposes -- for example, I'm running Eclipse locally and
need to test code that depends on Redshift.  Typically, that use case relies on a VPN, Direct Connect, or SSH port forwarding through
a bastion host.

But, a [new feature for AWS SSM](https://aws.amazon.com/about-aws/whats-new/2019/07/session-manager-launches-tunneling-support-for-ssh-and-scp/)
allows you to tunnel a legit SSH connection through AWS SSM, *and it supports port forwarding*.

The [documentation is pretty good](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html). I
also found [an AWS blog post](https://aws.amazon.com/blogs/aws/new-port-forwarding-using-aws-system-manager-sessions-manager/) on the topic
and [another blog post](https://www.tripwire.com/state-of-security/security-data-protection/cloud/aws-session-manager-enhanced-ssh-scp-capability/) with
decent instructions.  

The general idea is it works through SSH's `ProxyCommand`.  The `ProxyCommand` option is used to invoke `aws ssm start-session`
to establish a connection between your SSH client and `sshd` on the target EC2, rather than connecting directly over
port 22.  The AWS CLI is making API calls over HTTPS, so you only need port 443 open locally; and those calls are secured
by IAM authentication and policies.  So this works even if your network blocks outbound traffic besides ports 80/443.

Most of the instructions work by setting up `.ssh/config` to recognize any hostname of the form `i-*` (EC2 instance ID)
to be proxied through SSM.  So once you set it up, your SSH command line would look like `ssh user@i-12345...`

Because we're using `aws ssm` commands to tunnel a real SSH connection, though, other SSH options like port forwarding
work as usual.  You can add `-L 5439:redshift-cluster.xxx..redshift.amazonaws.com:5439` and then use tools like SQL Workbench
locally to connect to Redshift.  So if you were setting up a VPN or a Direct Connect just for use cases like this, you might
not need to do that anymore.

The downside is that you need to have a username/password on the target SSH server, or your SSH public key needs to be in
`.ssh/authorized_keys`.  One way to get started is, if you can already get a shell through Session Manager in the AWS console, you
can log in that way and add your public key to `.ssh/authorized_keys` under `ssm-user`.  Then you can `ssh ssm-user@i-12345...`
Otherwise you'd need some mechanism to set up individual accounts per user.
