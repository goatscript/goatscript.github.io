---
layout: post
title: Automate Sysprep on KVM Without DHCP
---

I ran into sysprep automation issues in a datacenter where DHCP wasn't allowed.  In our environment we may need hundreds of Window machines built, and prior to Chef, this was done by cloning a golden image across multiple KVM hosts.  Once the Windows guest are cloned, sysprep is run.  Automating Sysprep without DHCP presented a unique challenge.

To solve the issue I used guestfish to push customized sysprep files into the KVM Windows guests prior to first boot.  Guestfish is part of the libvirtd toolkit which provides a way to peer into KVM guest and, in this case, push files up to the virtual machines.  The process is simple and as follows.

- Clone a KVM Windows guest and leave the machine powered off
{% highlight bash %}
virt-clone -o seed-name -n name-of-vm -f path-to-create-vm --force
{% endhighlight %}

- Create a custome sysprep file for the KVM Windows guest.  Since building unattend.xml files for sysprep is well documented, I'll ommit that section.  Check [Microsoft's site](https://technet.microsoft.com/en-us/library/cc732280(v=ws.10).aspx) for more information.

- Use guestfish to push the custom unattend.xml file to the KVM Windows guest.
{% highlight bash %}
guestfish -d vm-hostname -i upload /path/to/unattend.xml /Windows/System32/sysprep/unattend.xml
{% endhighlight %}
- Power on the KVM Windows guest
{% highlight bash %}
virsh start vm-hostname
{% endhighlight %}
Cloning hundreds of KVM Windows guests and automating sysprep without DHCP becomes trivial, since the sysprep files and guestfish pushes are easily scripted.  Hope this helps someone....
