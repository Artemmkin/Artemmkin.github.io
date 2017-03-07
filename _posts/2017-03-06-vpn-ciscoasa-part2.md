---
layout: post
title: Set up Cisco ASA AnyConnect VPN with 2FA to multiple AWS VPCs (part II)
tags: [aws, vpn, freeradius, 2fa, ciscoasa]
---

In this part we will be working with Cisco ASA configuration.

__Prerequisite:__ you should have Cisco ASA up and running and it must have at least 2 interfaces. In this post we use 2 interfaces (```outside``` and ```inside```) for the purposes of keeping things simple. You should also have AnyConnect image for your OS as well as AnyConnect client to connect to VPN.

We are going to set up VPN service for 2 [peering VPCs](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-peering.html). You can have as many VPCs as you want. This will work for one VPC as well.

We will do almost every step from the command line, except for the case when we copy AnyConnect image to Cisco ASA. This we will require the use of Cisco ASA's GUI which is called ASDM. <!--break-->I want to note that this can also be done via command line with a ```copy``` command which supports http(s), tftp. But because I don't have neither artifactory nor tftp server set up, I will do this via ASDM. You may also try ```scp``` command, but in my case it takes forever to run.

### Upload the image
So the first thing I will do is to enable http server to access ASDM:
~~~yml
! we create a user with a password because running ASDM requires login credentials
ciscoasa(config)# username cisco password cisco
ciscoasa(config)# http server enable  
ciscoasa(config)# http 0 0 management
~~~
Management interface is the default interface which we will later rename to _outside_.

Now if you type the ```https://<ciscoasa_public_ip>``` in your browser, you should be able to run ASDM.

In ASDM, navigate to ```Configuration -> Remote Access VPN -> Network (Client) Access  ->  AnyConnect Client Software```, then click ```Add``` and upload AnyConnect image. After upload is finished, click ```OK``` and then ```Apply``` button to apply the changes. You can now close ASDM, it will no longer be needed.

### Time for configuration
We'll be using [this](https://github.com/Artemmkin/2FAVPN/blob/master/asa_config/ciscoasa-config.txt) config file from [2FAVPN](https://github.com/Artemmkin/2FAVPN) repository we used previously. You can download it, make the necessary changes and then run it from the command line. I'll go over the important lines here to explain what exactly we do and what you need to change in this config.

As it's not a rare case, let's pretend that Cisco ASA also acts as a router for some of the hosts in our ```inside``` network. That's why I put this sample NAT configuration at the beginning of the config file:
~~~yml
access-list some_nat permit ip any any
access-group some_nat in interface outside
object network obj-any
  subnet 0.0.0.0 0.0.0.0
  nat (inside,outside) dynamic interface
~~~
You can see that source IP address of all the traffic that comes from ```inside``` interface and goes out of ```outside``` interface will be dynamically translated to the IP address of our ```outside``` interface. We should be aware of this fact as this will affect the return traffic of VPN connections.

Then we create a network object for our VPN subnet:
~~~yml
object network VPN
subnet 192.168.100.0 255.255.255.0
~~~
Hosts that connect to VPN will receive IP address from this subnet.

As I mentioned moments ago, the NAT rule we used earlier will directly affect the return traffic of VPN connections and we need to do something about it. Here we have basically 2 options of either using NAT Exemption for our VPN traffic or adding another NAT rule for VPN traffic itself. We go with the second one, because the first option will require changing _VPC's route table_ to configure all traffic destined to VPN subnet go through Cisco ASA's inside interface.

We use PAT for our VPN connections:
~~~yml
nat (outside,inside) source dynamic VPN interface
~~~
This basically says to change source IP address of all the packets coming from our VPN subnet to IP address of Cisco ASA's ```outside``` interface when packets transition from ```outside``` to ```inside``` interface.

Now **the juicy part** of all this is the NAT rule we need to add to make our VPN work with multiple peering VPCs.
~~~yml
nat (outside,outside) source dynamic VPN interface
~~~
Here we're using dynamic PAT to translate the source IP address of the packets coming from VPN subnet into IP address of Cisco ASA's outside interface at the moment they arrive at the ```outside``` interface. The problem here is that there is no interface that connects 2 peering VPCs and [transitive peering doesn't work](https://docs.aws.amazon.com/AmazonVPC/latest/PeeringGuide/invalid-peering-configurations.html).

So suppose we have VPC1 that has our Cisco ASA in it and peering VPC2. For packets coming from VPN subnet to be routed to VPC2, they must have source address from one of the VPC1 subnets. And the easiest thing to do here is to use dynamic PAT to translate source IP address of the packets from VPN subnet into IP address of Cisco ASA's ```outside``` interface right at the moment those packets arrive at the ```outside``` interface.

I aslo want to note another line in our config file which is crucial for work with peering VPCs:
~~~yml
same-security-traffic permit intra-interface
~~~
This allows flow of traffic that comes in on an interface and is routed back out of the same interface.

We are done with the hard part of configuration, now we just need to configure Cisco ASA to use FreeRADIUS for VPN authentication and enable VPN.

We create stardard access-list with ACE for each of our VPCs. Change the subnets according to your setup.
~~~yml
access-list VPN-ACL standard permit 10.60.60.64 255.255.255.224
access-list VPN-ACL standard permit 10.10.10.64 255.255.255.224
~~~  
Create pool of addresses for our VPN connections:
~~~yml
ip local pool VPN-Pool 192.168.100.1-192.168.100.50 mask 255.255.255.0
~~~
And configure FreeRADIUS server. Make sure to change host IP address as mentioned in the comments.
~~~yml
aaa-server FreeRADIUS protocol radius
reactivation-mode depletion deadtime 3
! Change to the IP address of your FreeRADIUS server host
aaa-server FreeRADIUS (inside) host 10.60.60.85
timeout 60
authentication-port 1812
accounting-port 1813
retry-interval 10
! This key should match the key specified in ansible role "freeradius" (see defaults)
key superSecret
no mschapv2-capable
~~~
I omit the part when we create local group policy and connection profile as there is nothing difficult about it.

To enable AnyConnect VPN we use these commands:
~~~yml
webvpn
enable outside
! Change the package name
anyconnect image disk0:/anyconnect-linux-64-4.3.04027-k9.pkg
anyconnect enable
~~~~

If you read the comments in the [configuration file]() and make the necessary changes everything should work. But in case you run into some problems, ASDM could be very helpful to see what went wrong. Using ASDM you can check your current configuration, you can test you FreeRADIUS server, look at the logs and even use ```packet tracer``` (although I prefer command line version of this).

Now if you launch AnyConnect client, you should be able to connect to VPN using the ```username```, ```password``` and ```TOTP``` from Authenticator app which we created in the [first](/2017/03/05/vpn-ciscoasa-part1/) part of this blog series and be able to reach any of the hosts on our peering VPCs.


P.S. Just make sure security groups are right ;)
