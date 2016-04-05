---
layout: post
title: Open Chef HA Solution
---

Enterprise Chef has a built in HA solution.  Our data center uses the free version of Chef, so I needed to come up with a viable HA solution.  My environment consist of two discrete CentOS 6.7 machines connected with a corssover cable.  DRBD and keepalive are used to sync the machines.

The following solution works for Chef 11 and 12.

## Crossover Cable Setup

- Connect the crossover cable to eth1 on both machines

- Modify the ifcfg-eth1 on server 1.  In this case server 1 static ip is 192.168.1.1:
{% highlight text %}
DEVICE=eth1
HWADDR=XX:XX:XX:XX:XX:XX
BOOTPROTO=static
NM_CONTROLLED=yes
ONBOOT=yes
TYPE=Ethernet
NETMASK=255.255.255.0
IPADDR=192.168.1.1
BROADCAST=198.162.1.255
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
{% endhighlight %}

- Modify the ifcfg-eth1 on server 2.  In this case server 1 static ip is 192.168.1.2:

{% highlight text %}
DEVICE=eth1
HWADDR=XX:XX:XX:XX:XX:XX
BOOTPROTO=static
NM_CONTROLLED=yes
ONBOOT=yes
TYPE=Ethernet
NETMASK=255.255.255.0
IPADDR=192.168.1.2
BROADCAST=198.162.1.255
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
{% endhighlight %}

- Ping test the servers to verify connection

## Install DRBD

- Check for the Xen kernel

{% highlight text %}

rpm -qa kernel\* | grep -ci xen

{% endhighlight %}

If the output is zero use the following commands to install drbd and kmod-drbd

- Insall drbd-utils on both machines

{% highlight text %}
rpm -Uhv drbd83-utils-8.3.16-1.el6.elrepo.x86_64.rpm
{% endhighlight %}

- Install kmod-drbd83 on both machines

{% highlight text %}
rpm -Uhv kmod-drbd83-8.3.16-1.el6.elrepo.x86_64.rpm
{% endhighlight %}

Note:  the two rpms come from the ELRepo.  Make sure they are compatible with the kernel.

## Create the resource for sdc

- Create the resource file  /etc/drbd.d/sdc.res

{% highlight text %}

resource sdc
{
startup {
wfc-timeout 30;
outdated-wfc-timeout 20;
degr-wfc-timeout 30;
}

net {
cram-hmac-alg sha1;
shared-secret sync_disk;
}

syncer {
rate 100M;
verify-alg sha1;
}

on server1.lan {
device minor 1;
disk /dev/sdc1;
address 192.168.1.1:7789;
meta-disk internal;
}

on server2.lan {
device minor 1;
disk /dev/sdc1;
address 192.168.1.2:7789;
meta-disk internal;
}
}
{% endhighlight %}

- initialize the drbd metadata.  This uses the sdc.res file.

{% highlight text %}
/sbin/drbdadm create-md sdc
{% endhighlight %}

output:

{% highlight text %}


  --==  Thank you for participating in the global usage survey  ==--
The server's response is:

Writing meta data...
initializing activity log
NOT initialized bitmap
New drbd meta data block successfully created.
{% endhighlight %}


- start the drbd service
{% highlight text %}
service drbd start
{% endhighlight %}

output:
{% highlight text %}

Starting DRBD resources: [ d(sdc) s(sdc) n(sdc) ]..........
***************************************************************
 DRBD's startup script waits for the peer node(s) to appear.
 - In case this node was already a degraded cluster before the
   reboot the timeout is 30 seconds. [degr-wfc-timeout]
 - If the peer was available before the reboot the timeout will
   expire after 30 seconds. [wfc-timeout]
   (These values are for resource 'sdc'; 0 sec -> wait forever)
 To abort waiting enter 'yes' [  29]:
.
{% endhighlight %}

- Repeat the creation of the sdc.res file and drbd initialization on server 2.

- Verify both machines are set as secondary by checking the proc file.

{% highlight text %}
cat /proc/drbd
{% endhighlight %}

output:

{% highlight text %}
version: 8.3.16 (api:88/proto:86-97)
GIT-hash: a798fa7e274428a357657fb52f0ecf40192c1985 build by phil@Build64R6, 2013-09-27 16:00:43

 1: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:976698024
{% endhighlight %}

- Create the primary on server 1

{% highlight text %}
/sbin/drbdadm -- --overwrite-data-of-peer primary sdc
{% endhighlight %}

- Check the proc file.  There should be a primary and a secondary.  The sync between the two should start immediately.

{% highlight text %}
version: 8.3.16 (api:88/proto:86-97)
GIT-hash: a798fa7e274428a357657fb52f0ecf40192c1985 build by phil@Build64R6, 2013-09-27 16:00:43

 1: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent C r-----
    ns:22560768 nr:0 dw:0 dr:22561440 al:0 bm:1376 lo:0 pe:13 ua:0 ap:0 ep:1 wo:f oos:954138920
        [>....................] sync'ed:  2.4% (931776/953804)M
        finish: 6:11:12 speed: 42,820 (40,864) K/sec
{% endhighlight %}

- Check the proc file to verify the sync is complete.

{% highlight text %}
version: 8.3.16 (api:88/proto:86-97)
GIT-hash: a798fa7e274428a357657fb52f0ecf40192c1985 build by phil@Build64R6, 2013-09-27 16:00:43

 1: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:976698024 nr:0 dw:0 dr:976698696 al:0 bm:59613 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
{% endhighlight %}

## Setup Keepalived

- Install gcc kernel-headers kernel-devel and keepalived

- Create /etc/sysconfig/network-scripts/ifcfg-eth0:0 on both machines.

{% highlight text %}
DEVICE=eth0:0
TYPE=Ethernet
ONBOOT=yesvi 
IPV6INIT=no
NM_CONTROLLED=no
BOOTPROTO=none
IPADDR=11.22.33.44
NETMASK=XX.XX.XX.XX
GATEWAY=XX.XX.XX.XX
{% endhighlight %}

- bring up eth0:0 on both machines

{% highlight text %}
ifup eth0:0
{% endhighlight %}

- Modify or add the following to /etc/sysctl.conf to allow forwarding.

{% highlight text %}
# Controls IP packet forwarding
net.ipv4.ip_forward = 1
{% endhighlight %}

- Create /etc/keepalived/keepalived.conf file.

{% highlight text %}
! Configuration File for keepalived

global_defs {
   notification_email {
   }
   router_id CHEF-primary
}

vrrp_instance CHEF_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        11.22.33.44
    }
}
{% endhighlight %}

At this point the Primary and Secondary servers are in sync.  Test the failover until satisfied.  If I have time in the future I may post how I ran these tests, but there shouldn't be any surprises.

##  Create a Shared Server

- If not already done, install Chef server on the primary and secondary servers.

- Modify the Chef server with a new shared fqdn.  Do this by modifying the chef-server.rb file.  In this example I'm using nightwing.lan

{% highlight text %}
api_fqdn    "nightwing.lan"
{% endhighlight %}

- Add the shared fqdn and associated VIP(eth0:0) created previously to /etc/hosts.

- reconfigure the chef server

{% highlight text %}
chef-server-ctl reconfigure
{% endhighlight %}

- Test the chef server configuration.  There should be zero errors.

{% highlight text %}
chef-server-ctl test
{% endhighlight %}

- Make new directories /drbd/var/opt/, /drbd/var/log/, /drbd/var/chef/ and /drbd/etc/

- Copy /var/opt/chef-server, /var/log/chef-server and /var/chef/backup to the appropriate /drbd directory, making sure to keep permissions.

{% highlight text %}
cp -rp /var/opt/chef-server /drbd/var/opt/
cp -rp /var/log/chef-server /drbd/var/log/
cp -rp /var/chef/backup /drbd/var/chef/
{% endhighlight %}

- Copy /etc/chef-server to /drbd/etc
{% highlight text %}
cp -rp /etc/chef-server /drbd/etc/
{% endhighlight %}

- Create symbolic links for each of the copied directories.

{% highlight text %}
ln -s /drbd/var/opt/chef-server /var/opt/chef-server
ln -s /drbd/var/log/chef-server /var/log/chef-server
ln -s /drbd/var/chef/backup /var/chef/backup
ln -s /drbd/etc/chef-server /etc/chef-server
{% endhighlight %}

-  Reconfigure the Chef server once the symbolic links are created for /var/opt/chef-server, /var/log/chef-server, /var/chef/backup and /etc/chef-server.  There shouldn't be any errors

{% highlight text %}
chef-server-ctl reconfigure
{% endhighlight %}

- If the webui is part of your environment, then run chef-server-ctl restart after the reconfigure.  This will clear the cache and make the webui accessible.

- Create the same symbolic links on the secondary.

If the primary goes offline, promote the secondary to primary and the symbolic links will point to the shared chef server residing on the DRBD volume.
