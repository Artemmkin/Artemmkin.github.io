---
layout: post
title: Set up Cisco ASA AnyConnect VPN with 2FA to multiple AWS VPCs  (part I )
tags: [aws, vpn, freeradius, 2fa, ciscoasa]
---

In the **1st** part of this blog series, we install and configure FreeRADIUS server which we'll use for two-factor authentcation([2FA](https://en.wikipedia.org/wiki/Multi-factor_authentication)).

__Prerequisite:__ you should have Cisco ASA up and running and it must have at least 2 interfaces (inside and outside)

I'll briefly go over the steps required to install and configure FreeRADIUS server on Ubuntu 14.04. We'll be using [freeradius](https://github.com/Artemmkin/2FAVPN/tree/master/roles/freeradius) ansible role from this repository [2FAVPN](https://github.com/Artemmkin/2FAVPN) for installation and configuration of FreeRADIUS server. Of course, you can do every step manually, but that's not very "devopsy" :)
<!--break-->

### So here is what freeradius role will do for us

As we are going to use Time-Based One Time Password (TOTP) for our 2FA authentication, we start with installing ```ntp``` to keep our clock in sync. Then we install FreeRADIUS itself, tools required for debugging, and [google_authenticator](https://github.com/google/google-authenticator-libpam) pam module which FreeRADIUS will use for authenticating the users.
~~~yml
sudo apt-get update
sudo apt-get install ntp freeradius freeradius-utils libpam-google-authenticator
~~~

After we installed all the packages, we need to tweak a few configuration files.
We start with changing ```/etc/freeradius/radiusd.conf file```. We need to change ```user``` and ```group``` to ```root```:
~~~yml
user = root
group = root
~~~
Google-authenticator program that we'll be using in just a few moments generates a file with a secret key for a system user. FreeRADIUS server must have access to this file to be able to perform the authentication, that is why we changed user and group values to root.

Now we need to tell our FreeRADIUS server that our authentication type will be ```pam module```. We add the following line to ```/etc/freeradius/users```:  
~~~yml
DEFAULT        Auth-Type := PAM
~~~
After that we uncomment the ```#pam``` line inside ```/etc/freeradius/sites-enabled/default``` to enable the use of pam module by FreeRADIUS server.
~~~yml
#  pam # uncomment this line
~~~
Finally, we edit ```/etc/pam.d/radiusd``` file to configure our FreeRADIUS to authenticate users based on their system password and Google Authenticator token.

Our ```/etc/pam.d/radiusd``` file will look like this:
~~~yml
#@include common-auth
#@include common-account
#@include common-password
#@include common-session

auth requisite pam_google_authenticator.so forward_pass
auth required pam_unix.so use_first_pass
~~~
Also, we need to change clients config to tell FreeRADIUS server to handle requests from Cisco ASA, so we add the following lines to ```/etc/FreeRADIUS/clients.conf```:
{% raw %}
~~~yml
client ASA {
  ipaddr = {{ asa_internal_iface_ip }}
  secret = {{ shared_secret }}
  }
~~~
{% endraw %}
The ```ipaddr``` value should equal to the IP address of the Cisco ASA's ```inside``` network interface. You can change the corresponding variables in the defaults.

We're done with FreeRADIUS configuration and it's time to create our first system user and generate secret key for this user.
~~~yml
- name: Generate secret key file for {{ admin_username }} user
  command: sudo -u {{ admin_username }} google-authenticator  -t -d -r 1 -R 30 -w 3 -s /home/{{ admin_username }}/.google_authenticator -f
~~~
When running ```google_authenticator``` command, it'll prompt you for choosing different options. To avoid that and automate the process, we pass the values of those options along with the command.

Once the secret key was generated, we will show the contents of {% raw %} ```/home/{{admin_username}}/.google_authenticator```{% endraw %} with ```debug``` module. At almost the very end of ansible run, you'll see something similar to this

![400x200](/public/img/CiscoVPN/secret_key.gif)

After that we enter our secret key in Google Authenticator app so that it could start generating TOTPs for us.

We can check that our FreeRADIUS server works as expected by running a local authentication test. Run the following command on the machine where FreeRADIUS is running.
~~~yml
radtest <username> <unix_password><google_auth_token> localhost 18120 testing123
~~~
If configured correctly, you should see a message that Access-Accept packet was received.

![400x200](/public/img/CiscoVPN/access_packet.jpg)

If something went wrong, you can use these commands to start FreeRADIUS in debug mode:
~~~yml
service freeradius stop
freeradius -XXX
~~~

To be continued ...
