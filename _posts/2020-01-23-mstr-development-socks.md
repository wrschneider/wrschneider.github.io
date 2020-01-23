---
layout: post
title: MSTR Web development over SOCKS proxy and AWS SSM
description: How to do MicroStrategy (MSTR) Web development when the I-servers are in AWS and you don't have SSH access
---

*The problem*: You want to work on MicroStrategy (MSTR) Web customizations in your local Eclipse/Tomcat environment, but you don't have connectivity to your I-server.  Your
I-server lives in AWS, your corporate network blocks outbound port 22 access, and you don't have a VPN or direct connect.

*Solution*: I had previously talked about AWS's new [SSH over SSM feature](http://wrschneider.github.io/2019/09/10/ssm-ssh-tunnel.html).

You can do port forwarding over this like any other SSH connection.

The catch with MSTR development is that, because of MSTR's old-school approach to clustering, MSTR will attempt to connect to multiple 
cluster nodes on the same port (34952 by default).  In other words, if you forward with something like `-L 34952:Iserver:34952`, and tell
MSTR Web that your I-server is "localhost", it will still try to connect to two different hostnames after establishing the initial
connection.

One workaround I came up with is to use a SOCKS proxy, which is like a mini-VPN inside SSH.  A process using the SOCKS proxy will use 
the proxy for all TCP connections, so MSTR Web can access the actual IP addresses for each indiviudal I-server in the cluster.  Instead of forwarding
an individual port via localhost, you use the actual IP addresses.

General steps for setting this up:

* `ssh -D 7777 [ec2 instance ID]` will establish a SOCKS proxy on localhost:7777 with the SSH connection
* edit your `/etc/hosts` file to resolve your I-server hostnames to their actual IP addresses (EC2 private IPs)
* add `-DsocksProxyHost=localhost -DsocksProxyPort=7777` to your Tomcat JVM options. 

Note that DNS requests do *not* go through SOCKS.  Editing `/etc/hosts` is a workaround for that.  Be mindful of hostnames that might resolve
differently in your corporate network vs. the internet/EC2.
