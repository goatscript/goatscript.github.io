---
layout: post
title: CentOS 6.2 yum proxy settings
---

Working with Vagrant and Test Kitchen I ran into yum proxy issues.  There is a subtle difference between CentOS 6.2 and CentOS 6.4 that eluded me for a bit.  My initial work was with CentOS 6.4 and after successfully getting the build to work, I ported the settings into CentOS 6.2, where Test Kitchen failed.  Further investigation revealed a yum issue.  Packages were handled as expected on CentOS 6.4, but failed on CentOS 6.2.  After further investigation, the issue was with /etc/yum.conf in cases where the proxy doesn't require a username or password.

The following /etc/yum.conf works on CentOS 6.4, but not on CentOS 6.2.

{% highlight text %}
#
# Regular yum.conf setting go here
#
proxy=http://xx.xx.xx.xx:xx
proxy_username=
proxy_password=
{% endhighlight %}

The CentOS 6.2 /etc/yum.conf file requires an empty string for the username and password.  At this time I'm not sure why I thought adding an empty string would work....but it does.

The following /etc/yum.conf works on CentOS 6.2 and CentOS 6.4.
{% highlight text %}
#
# Regular yum.conf setting go here
#
proxy=http://xx.xx.xx.xx:xx
proxy_username=""
proxy_password=""
{% endhighlight %}

To complete the Vagrant configuration, modify the vagrant.rb to use the needed /etc/yum.conf proxy settings.
Sample vagrant.rb file.
{% highlight ruby %}
unless Vagarant.has_plugin?("vagrant-proxyconf")
  raise "Missing vagrant-proxyconf.  Install vagrant-proxyconf in order to proceed."
end

Vagrant.configure(2) do |config|
  config.proxy.http = "#{ENV['http_proxy']}"
  config.proxy.https = "#{ENV['https_proxy']}"
  config.proxy.no_proxy = "localhost,127.0.0.1"
  config.vm.provision "shell", inline: "sed -i 's/proxy_username=/proxy_username=\"\"/g' /etc/yum.conf"
  config.vm.provision "shell", inline: "sed -i 's/proxy_username=/proxy_password=\"\"/g' /etc/yum.conf"
end
{% endhighlight %}

After making the changes, Vagrant CentOS 6.2 Test Kitchen runs should successfully access yum repositories through the proxy.

