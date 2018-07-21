+++
title = "Securing your server's SSH configuration"
author = "Sander Knape"
date = 2016-11-06T12:43:02+02:00
draft = false
tags = ["linux", "security", "ssh"]
categories = []
+++
Are your SSH log files flooding with failing login attempts? I've seen many questions on websites such as Stackoverflow and Stackexchange from worried people that someone is actively targeting their servers with brute-force password logins attempts. Let me get one thing straight first: _you are not special!_ It's part of internet life: many botnets constantly attempt to login to servers. These can be random IP addresses or known ranges such as Amazon AWS EC2 instances or DigitalOcean droplets. There's nothing much you can do about this except for making sure that your server is securely set up.  

One of the key security measures you have to take is properly setting up your SSH configuration. If password login is enabled and you are using an insecure password, a brute-force method might give someone access to your server. In this blog post, I'll discuss some methods for properly setting up SSH on your server.

# Securing your server

There are different ways to tighten up the security on your server. By far the safest approach is to disable password logins altogether, though there are situations where this isn't practical. In total I'll discuss five different methods for hardening your server:

*   Disable password logins
*   Block too many login attempts
*   White-list specific IP addresses
*   Change your SSH port
*   Port knocking

## Disable password logins

By far the most secure method of protecting your servers is by disallowing password logins altogether. Passwords are many times less secure than logging in with a public token. The computing power required to brute-force an SSH key is not within reach for the coming years. A password though, and especially a weak one, can definitely potentially be cracked using brute force.  

Disable password logins by editing the `/etc/ssh/sshd_config` file with your favorite editor. Find the line that contains the `PasswordAuthentication` property and disable it with:

```
PasswordAuthentication no
```

Before restarting the ssh daemon to take the changes into effect, be sure you have created and configured an SSH key to use for login. Without an SSH key you will not be able to login to your server anymore after disabling password logins. If you don't know how to create and configure an SSH key, check out this [DigitalOcean blogpost](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2) that explains exactly how to do so.

## Block too many login attempts

It might very well be that disabling password logins is simply not suitable for your use case. It then becomes especially important to block brute force attacks. There are multiple ways to do this, but let me focus on two specific methods here.

### Recent

First, IPTables has a built in module called "recent" for blocking too many connections attempts. We can specify both an interval and the number of attempts within that specified interval. An example configuration is as follows;

```bash
iptables -N SSHBLOCK
iptables -A INPUT -p tcp -m tcp --dport 22 -m state --state NEW -j SSHBLOCK

iptables -A SSHBLOCK -m recent --name blocks --set --rsource
iptables -A SSHBLOCK -m recent --name blocks --update --seconds 60 --hitcount 5 --rsource -j DROP
```

I am going to assume some at least basic knowledge about iptables here. If you are not familiar with the syntax, be sure to find a tutorial somewhere that explains the basic concepts of configuring iptables. Always be **very** careful when working with firewalls; it's possible to lock yourself out of your system if you make a mistake.  

We begin with creating a new chain in the first rule to which we forward all the SSH connections in the second rule. Though not required per sé, it makes managing the rules related to the rate limiting a bit easier.  

We keep track of the attempted connections in a list called "blocks". The third rule adds the connecting IP address to this list. If the IP address was already in there, it will be appended. The final rule checks if the IP address is already within the list and if it has tried to connect 5 times within 60 seconds. If so, the connection is dropped and the IP address will have to wait 60 seconds for another attempt to make a connection. Anything not picked up by these rules is forwarded and the default chain policy decides what to do with it.  

You can keep track of the connections being blocked by looking at the `/proc/self/net/xt_recent/blocks` file. The exact location of the file might differ with your distribution and kernel version. The "blocks" filename is the same as the name of the list we defined in the iptables rules. Using the `watch` command we can watch this file while attempting some connection attempts:

```bash
watch -n 1 cat /proc/self/net/xt_recent/blocks
```

In another shell, attempt to connect to your server a few times. As you do this, you will see your IP address being listed in the file. Don't be suprised if you see some other IP addresses popping up in the meantime.  

The total number of IP addresses and number of connections per IP address that are stored by the recent module can be configured. The following two parameters are especially of interest:

*   _ip\_list\_tot_. The total number of IP addresses stored.
*   _ip\_pkt\_list_tot_. The total number of connections per IP address stored.

The current values for these parameters can be found in the `/sys/module/xt_recent/parameters/{ip_list_tot, ip_pkt_list_tot}` files. Changing these values requires changing the parameters for the xt_recent kernel module, which is out of scope for this blog post. It's good to know though that the file won't grow until your mount is completely full!

### Fail2Ban

A second method for blocking too many SSH connection attempts is by using [Fail2Ban](https://github.com/fail2ban/fail2ban). Fail2Ban can do _much_ more than blocking SSH login attempts. It also supports blocking connections to software such as Apache, Nginx, HaProxy and MySQL. For now, let's setup a simple configuration for blocking our SSH login attempts.  

The GitHub page I linked to includes the installation instructions, however be sure to first check if a package exists for your distribution. Once installed, create the file _/etc/fail2ban/jail.local_ with the following contents:

```
[DEFAULT]
# ban for 60 seconds
bantime = 60

# ban when 3 attempts are made within 60 seconds
findtime = 60
maxretry = 3

# block through iptables
banaction = iptables-multiport

[sshd]
# enable the above settings for sshd
enabled = true
```

I've added comments to the configuration to explain the different settings. Restart Fail2Ban to take the new settings into effect. Try to login a few times to your server and you will find that you can not connect anymore. After 60 seconds you will be able to attempt a login again.  

The advantage of IP tables compared with Fail2Ban is that you do not need to install an additional package to your system. On the other hand, Fail2Ban is easier to setup and it supports much more than just blocking ssh connections. Both tools do a good job, so pick the right one for your situation!  

**Important**: I cannot stress enough that whatever brute-force-blocking mechanism you have setup, its use can totally be undone by using an insecure password. If you really can not disable password logins, always be sure to use a secure password.

## White-list specific IP addresses

If you only login to your server from one or a few specific IP addresses, it's an option to white-list only those IP addresses in your firewall to the SSH port. You can open up your firewall for your home address and work address, for example. However, if you're at a friends house and want to login to your server, you are out of luck. There will certainly be use cases where this is a perfectly viable setup though.  

There are two ways for whitelisting specific IP addresses for your SSH daemon. First, we can use iptables for this. If your current iptables rules list is empty, the following rules will suffice:

```bash
iptables -A INPUT -p tcp -m tcp --dport 22 -s [your_ip_address] -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 22 -m state --state NEW -j DROP
```

The first rule will accept your IP address and allow you to make a connection. Anyone else goes through to the second rule and is blocked if they try to connect to port 22. What happens to any other connection attempt depends on the default chain policy. If you already have some existing rules, be sure to tailor this for your specific ruleset.  

Second, you can change your sshd configuration to allow only specific IP addresses. In `/etc/ssh/sshd_config`, we can use the following syntax:

```
AllowUsers [username]@[ip_address]
```

The `[username]@[ip_address]` part can be used multiple times to allow multiple IP addresses and users.

## Change your SSH port

By default, you connect to port 22 to connect with the SSH daemon (sshd) on a server. Changing this port will still allow you to connect and will probably block at least 90% of the automated scripts that try to break into your server. However, whatever software you use to connect with your server over SSH will need to be changed to connect with the new port. If you have any automation scripts or external software that uses SSH to connect, be sure to check beforehand if they have support for connecting to a different port.  

Changing your SSH port is very easy. Open up the `/etc/ssh/sshd_config` file in your favorite editor and change the following line:

```
#Port 22
```

Uncomment the line and change the port to a new value. It's recommended to change your port to a value > 49152. Following the [IANA port assignment guidelines](http://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml), this will minimize the chance of your port colliding with an another service that also uses this port. Finally, be sure to restart your sshd deamon to take the change into effect (`systemctl restart sshd` when using systemd). Connecting to a different port with SSH is easy with the `-p` parameter:

```bash
ssh -p [port] [user]@[host]
```

Remember that whoever knows your SSH port can still execute any brute force attempts if no other mechanisms are in place to battle that. A port scanner will still show your open port, so keep that in mind.

## Port knocking

This is by far the most creative way to add an additional layer of protection. By default, we close our SSH port which will block anyone who tries to connect. Only when a specific sequence of connection attempts (the "knocking") on specific ports is made, the SSH port is opened temporarily so that someone can connect. These other ports don't need to be open; the knockd service will also listen on closed ports.  

As knockd will open up your SSH port after a correct sequence, we need to make sure that this port is closed by default. Closing your SSH port is always tricky as it might block you out of your server. It's best to first attempt this on a test server before doing it on your actual server. Given that no other iptables rules are set, the following rule will block any _new_ incoming SSH connections. This means that your current open connection will not be dropped:

```bash
iptables -A INPUT -p tcp -m tcp --dport 22 -m state --state NEW -j DROP
```

The knockd service is available for Debian distributions through installation with apt-get. For other distributions you might need to install from source which you can find on the [knockd website](http://www.zeroflux.org/projects/knock). On Ubuntu, install the service with the following command:

```bash
apt-get install knockd
```

Next, open up the knockd configuration file (/etc/knock.conf) with your favorite text editor. As you will see, there is already an example configuration set. This configuration requires us to close the SSH port with a different sequence of connection attempts. We can change this so that the port is automatically closed after a defined interval, such as 20 seconds. Yes, you will have 20 seconds to login to your server after the knocking sequence! You can use the following configuration for this:

```
[options]
UseSyslog

[SSH]
sequence = 8888,7777,6666
tcpflags = syn
seq_timeout = 10
start_command = /sbin/iptables -I INPUT 1 -s %IP% -p tcp --dport 22 -j ACCEPT
cmd_timeout = 20
stop_command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
```

The configuration is pretty self explanatory. We configure three ports that consequently need to be connected to. We also specify that this sequence needs to be performed within 10 seconds and that the port will close again after 20 seconds. Opening and closing (or "starting" and "stopping") the SSH port is done through two iptables rules. Of course, change this line if you have changed your SSH port to another one.  

Before we can start knockd we need to edit the `/etc/default/knockd` file. Find the line that says `START_KNOCKD=0` and change this to `START_KNOCKD=1`. Save the file and start knock with `service knockd start`.  

"Knocking" on this server from another server can be done with the same knockd package. On a different server, install the same package. We can use the following command now:

```bash
knock [ip_address] 8888 7777 6666
```

Now, connect to the server with SSH and you should be able to connect to the server. Logout, wait a second or 20 and try to login again. The port should now be closed again.  

Similar to changing your SSH port, this approach requires you to change the way you login to your server. Again, if you use any external services that login to your server, make sure they will be able to do so with knockd enabled. You do not need to use the knockd service per sé to "knock" on a server. Connecting to the port sequence with nmap will yield the same result. Keep this in mind if you need to automate the "knocking" for a different service that connects with your server.

# Conclusion

There are many ways to properly harden the security of your server. In this blog post we specifically looked at SSH. The internet is a scary place with many botnets looking to get into any poorly configured servers. It's a good thing then that there are many ways to protect your server:

*   Whenever possible: disable password logins.
*   If that really isn't possible: use a strong password!
*   Block brute force attacks by blocking too many login attempts.
*   Consider adding some extra _obscurity_ such as changing your SSH port or enabling port knocking.

Both changing your SSH port and enabling port knocking can be considered a _security through obscurity_ approach: hiding our service instead of really securing it. When used in combination with a solid security mechanism such as a strong password, I consider such an approach as an additional layer of security. Practicalities aside, it will only make it harder to get into your server.
