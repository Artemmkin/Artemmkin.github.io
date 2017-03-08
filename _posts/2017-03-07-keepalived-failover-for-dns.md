---
layout: post
title: Configure failover with Keepalived for services on AWS
tags: [aws, keepalived, failover, dnsmasq]
---
AWS is awesome. It provides a great variety of services with the goal of making the process of building robust infrastructure easier.

When it comes to using Load Balancer or DNS server, [ELB](http://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) and [Route53](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html) are the 2 most popular choices that may cover almost all your needs. Nevertheless, these services have their own limitations. And sometimes you may need to have your own DNS server or load balancer running inside your VPC.

As we all know, every service is prone to failure and we want to make sure we have failover logic in place when it happens.
For load balancer and DNS servers [keepalived](https://github.com/acassen/keepalived) is by far one of the most popular choices for providing failover as well as load balancing (when coupled with [IPVS](https://en.wikipedia.org/wiki/IP_Virtual_Server)).

In this post we'll see how we can provide failover for 2 DNS servers with keepalived.<!--break--> As for DNS servers, we'll use [dnsmasq](https://linux.die.net/man/8/dnsmasq), but again it could be a load balancer or a different DNS server. Our focus here is to see what is keepalived and how we can use it.

### Keepalived role

We'll be using [this ansible role](https://github.com/Artemmkin/keepalived-aws-ansible) to install and configure keepalived as well as dnsmasq servers. For a summary on how to run this role, see [Readme](https://github.com/Artemmkin/keepalived-aws-ansible/blob/master/README.md)

I'll go over the important configuration steps inside the role to explain what is going on.

First of all, before we start working with configuration, we need to spin up 2 EC2 instances (Ubuntu14.04) and assign a secondary IP address to one of them which will be our main (_master_ in keepalived terminology) dnsmasq server from the start.

To assign a secondary IP addresses to an instance, right click on the instance's name and navigate to ```Networking -> Manage IP Addresses```, then click ```Assign new IP``` and ```Yes, update```.

We'll use this secondary IP address as a virtual IP address ([VIP](http://www.webopedia.com/TERM/V/virtual_IP_address.html)). VIP is basically an IP address shared between the hosts. That is when the _master_ server fails, the VIP will be assigned to a _backup_ server. This way all the requests could be directed to a single IP address.

To complete the IP assignment we need to add our secondary IP address to the ```eth0``` interface which we do inside our keepalived role:
~~~yml
{% raw %}
- name: Add secondary IP address
  template:
    src: eth0.cfg.j2
    dest: /etc/network/interfaces.d/eth0.cfg
    mode: 0644

- name: Add secondary IP directly # to avoid reloading ifaces
  command: "ip addr add {{ VIP }}/{{ VIP_subnet }} dev eth0"
{% endraw %}
~~~
So all you need to do is to change ```VIP``` and ```VIP_subnet``` variables inside the defaults. After that, requests could be directed to this IP address.

Next, we install [aws cli](https://github.com/aws/aws-cli). We'll be using aws cli for VIP reassignment as part of our failover.
To use aws cli, we need to have user account credentials.

So before moving forward with our configuration, let's create an IAM user with appropriate access policy.
First, create a policy using the contents of [aws_policy_example](https://github.com/Artemmkin/keepalived-aws-ansible/blob/master/aws_policy_example) file. Then create a user and attach the policy you just created to this user. Once the user is created, change the [vars/main.yml](https://github.com/Artemmkin/keepalived-aws-ansible/blob/master/roles/keepalived/vars/main.yml) file appropriately.
~~~yml
aws_access_key:
aws_secret_key:
region:
~~~
Now getting back to our ansible role, we download one of the latest versions of keepalived and build it. The reason why we go for the latest version is because we're going to use _unicast_ as a way of communication between our keepalived services. The thing is that AWS doesn't allow _multicast_ and the old version of keepalived doesn't support unicast.

After we installed keepalived, we copy a few bash scripts that our keepalived service will be using for checking the status of dnsmasq and taking action when it doesn't find it running.
First, we copy ```test-dnsmasq.sh``` file with the following content:
~~~yml
#!/bin/bash
killall -0 dnsmasq
~~~
This basically checks if dnsmasq service is running. We can judge whether it is running or not by the exit status after running this script. As we'll see in a few moments, keepalived runs this script with specified frequency and checks the exit status. If the exit status is zero, then everything is fine and no action will be taken. If it's a non-zero value, keepalived will enter the _fault_ state (which we'll avoid with the use of ```weight``` parameter, see below).

We also copy ```master.sh``` script:
~~~yml
{% raw %}
#!/bin/bash
VIP={{ VIP }}
Instance_ID=`/usr/bin/curl --silent http://169.254.169.254/latest/meta-data/instance-id`
ENI_ID=`aws ec2 describe-instances --instance-ids $Instance_ID | grep eni -m 1 | awk '{print $2;}'`
ENI=`echo ${ENI_ID::-1} | tr -d '"'`

aws ec2 assign-private-ip-addresses --network-interface-id $ENI --private-ip-addresses $VIP --allow-reassignment
{% endraw %}
~~~
This script will be run in case of transition to a ```MASTER``` state, i.e. when ```test-dnsmasq.sh``` script returns a non-zero value or instance with dnsmasq master server simply dies.
In this script, we first determine our ```Instance ID```, then use its value to find out ```Network Interface ID```, and finally assign VIP to the instance on which we're running the script.

Now we reached the most interesting part which is keepalived configuration ^^
~~~bash
{% raw %}
vrrp_script chk_service {
       script "/etc/keepalived/test-dnsmasq.sh"
       interval {{ interval }}                      # check every 2 seconds
       fall {{ fall }}       # require 2 failures for KO
       rise {{ rise }}       # require 2 successes for OK
       weight -5 # if substracted from master's priority the number should be less than backup's priority
}

vrrp_instance VI_1 {
   state {{ keepalived_role }}
   interface {{ keepalived_shared_iface }}
   virtual_router_id {{ keepalived_router_id }}
   {% if keepalived_role.lower() == "master" %}
   priority {{ priority_master }}
   unicast_src_ip {{ master_ip_address  }} # private IP of the host
   unicast_peer {
      {{ slave_ip_address }} # peer IP
   }
   {% else %}
   priority {{ priority_slave }}
   unicast_src_ip {{ slave_ip_address  }}
   unicast_peer {
      {{ master_ip_address }} # peer IP
   }
   {% endif %}

   track_script {
       chk_service
   }
   notify_master /etc/keepalived/master.sh
}
{% endraw %}
~~~
```vrrp_script``` block serves the purpose of running our ```test-dnsmasq.sh``` script with the specified frequency.
The key part in this block is the ```weight``` parameter. Its value is ```-5``` which means that ```5``` will be subtracted from vrrp instance _priority_ when script returns a non-zero value.
From the start we assign each keepalived instance a priority. Instance with the highest priority will be the ```MASTER```. So in this case, when the script check fails, we subtract a value from instance priority big enough to make it transition into ```BACKUP``` state.

The rest of the configuration is pretty much self-explanatory. The only thing I want to note here is this line:
~~~yml
notify_master /etc/keepalived/master.sh
~~~
This tells keepalived to run ```master.sh``` script when changing its state to ```MASTER```.

### Dnsmasq
We're done with keepalived configuration and it's time to go through dnsmasq installation process. Note that we should have our service installed _before_ keepalived. Here I'm just walking you through the steps of the ansible role we're going to use.

We'll keep dnsmasq installation simple, just enough to check the work of our failover provided by keepalived.

We configure the dnsmasq server to resolve domain names to IP address that we specify in the [defaults](https://github.com/Artemmkin/keepalived-aws-ansible/blob/master/roles/keepalived/defaults/main.yml) section of our role:
~~~yml
domain_dict:
  - {domain_name: "blah.blah.com", address: "1.1.1.1"}
  - {domain_name: "test.test.com", address: "2.2.2.2"}
~~~
We will use these later for tests.
We also specify Google DNS server as one of our upstream servers:
~~~yml
upstream_servers: ["8.8.8.8"]
~~~
So that all the requests regarding domain names we haven't specified in ```domain_dict``` will be redirected to the upstream server.

### It's time to test our failover ^^

Let's spin up a new ubuntu instance. Once it's up and running, change ```/etc/resolv.conf``` file:
~~~yml
nameserver <VIP>
~~~
Change ```<VIP>``` to the secondary IP address we created before.

If you ever worked with Amazon, you probably already know that ```/etc/resolv.conf``` is automatically generated and changing it manually is not a good idea at all. The proper way to do this is to create a new ```DHCP options set``` for a VPC. But for testing keepalived, changing ```/etc/resolv.conf``` will do.

Let's try to resolve a domain name we put in ```domain_dict```. Run on the launched ubuntu instance the following command:
~~~yml
dig test.test.com
~~~
You should receive in return an IP address you put in ```defaults```.

![400x200](/public/img/keepalived/digtest.jpg)

Now let's stop dnsmasq service on our _master dnsmasq_ instance:
~~~yml
service dnsmasq stop
~~~
Try resolving this name again. It may take a few seconds for a ```BACKUP``` dnsmasq server to transition to ```MASTER``` state because of the options we specified in ```vrrp_script``` block.


You should also be able to see in your AWS Console that the secondary IP address was reassigned to the _second_ dnsmasq instance.

Now let's turn dnsmasq service back on on the _first_ dnsmasq instance:
~~~yml
service dnsmasq start
~~~
You should see that VIP was reassigned back to the first instance and resolution of our domain names still works.

The VIP will also be reassigned if you stop the master dnsmasq instance and you will still have dnsmasq server available at the same IP address.

In case something went wrong and not working or you simply wish to see how keepalived changes its state, you can look into ```/var/log/syslog``` for logs.
