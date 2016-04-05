---
layout: post
title: Proxy Settings - Docker, Vagrant and Packer
---

Working behind a proxy often means spending time troubleshooting settings to get things to work properly.  I ran into proxy issues working with Docker, Vagrant and Packer.  Below describes my workarounds when working with CentOS 6.

# Docker

- Create a Dockerfile.  In this example I'm using the CentOS 6 official base image.
- Set the proxy for the bash profile
{% highlight text %}
RUN echo 'export {http,https,ftp}_proxy="http://xx.xx.xx.xx:xx"' >> /root/.bash_profile
{% endhighlight %}
- Set the proxy settings in /etc/yum.conf
{% highlight text %}
RUN echo 'proxy=http://xx.xx.xx.xx:xx' >> /etc/yum.conf
RUN echo 'proxy_username'=""' >> /etc/yum.conf
RUN echo 'proxy_password'=""' >> /etc/yum.conf
{% endhighlight %}
- In order to get the bash profile to reload, I added a source command upon entry.
{% highlight text %}
ENTRYPOINT /bin/bash | source /root/.bash_profile
{% endhighlight %}

There are probably different, or better, ways to do this, but for now this works and allows me to use my dockerfiles behind a proxy.

# Vagrant

There are several proxy configurations needed when working with Test Kitchen and Vagrant.  To create a successful build custom Vagrant.erb, vagrant.rb and .kitchen.yml files are needed.

- Install the vagrant-proxyconf plugin for Test Kitchen
{% highlight bash %}
vagrant plugin install vagrant-proxyconf
{% endhighlight %}
- Include the proxy settings in the Vagrant.erb file
{% highlight ruby %}
if Vagrant.has_plugin?("vagrant-proxyconf")
  c.proxy.http = "http://xx.xx.xx.xx:xx"
  c.proxy.https = "https://xx.xx.xx.xx:xx"
  c.proxy.no_proxy = "localhost,127.0.0.1"
end
{% endhighlight %}
- Create a vagrant.rb file to load proxy settings.
{% highlight ruby %}
unless Vagarant.has_plugin?("vagrant-proxyconf")
  raise "Missing vagrant-proxyconf.  Install vagrant-proxyconf in order to proceed."
end

Vagrant.configure(2) do |config|
  config.proxy.http = "#{ENV['http_proxy']}"
  config.proxy.https = "#{ENV['https_proxy']}"
  config.proxy.no_proxy = "localhost,127.0.0.1"
end
{% endhighlight %}

- To tie everything together, modify the .kitchen.yml files to add proxy settings and locations of the Vagrantfile.erb and vagrant.rb files.  Sample .kitchen.yml file:
{% highlight yaml %}
---
driver:
  name: vagrant
  driver_config:
    http_proxy: http://xx.xx.xx.xx:xx
    https_proxy: https://xx.xx.xx.xx:xx
  network:
    - ["private_network", {type: "dhcp"}]

provisioner:
  name: chef_zero

busser:
  ruby_bindir: /opt/chef/embedded/bin/

platforms:
  - name: centos-6.2
    driver:
      box: centos-6.2
      box_url: https://opscode-vm-bento.s3.amazonaws.com/vagrant/boxes/opscode-centos-6.2.box
      provision: true
      require_chef_omnibus: "12.7.2"
      vagrantfile_erb: /root/chef-repo/cookbooks/gsntp/Vagrantfile.erb
      vagrantfiles:
        - vagrant.rb

suites:
  - name: default
    run_list:
      - recipe[gsntp::default]
    attributes:

{% endhighlight %}

- Kitchen test should now successfully build the vagrant box with correct proxy settings.

# Packer
Again with Packer, I ran into proxy issues.  My solution leverages a similar approach used with Docker and Vagrant.  Basically I added the proxy configurations to the packerfile.json.

- Under the provisioners section add shell commands to setup the proxy settings.
- Example of a packerfile.json
{% highlight text %}
{
  "builders": [{
    "type": "virtualbox-iso",
    "guest_os_type": "RedHat_64",
    "guest_additions_mode": "upload",
    "guest_additions_path": "/VBoxGuestAdditions.iso",
    "headless": true,
    "iso_checksum_type": "md5",
    "iso_checksum": "26fdf8c5a787a674f3219a3554b131ca",
    "iso_url": "file:///home/CentOS-6.2-x86_64-bin-DVD1.iso",
    "ssh_username": "user",
    "ssh_password": "password",
    "ssh_pty": true,
    "ssh_wait_timeout": "1000s",
    "shutdown_command": "sudo -S shutdown -P now",
    "http_directory": "http",
    "boot_wait": "30s",
    "boot_command": [
      "<tab> text ks=http://{{.HTTPIP}}:{{.HTTPPort}}/nightwing.cfg<enter><wait>"
    ]
  }],
  "post-processors": ["vagrant"],
  "provisioners": [{
    "type": "shell",
    "inline": [
      "sleep 30",
      "export HTTP_PROXY='http://xx.xx.xx.xx:xx'”,
      "export HTTPS_PROXY='http://xx.xx.xx.xx:xx'”,
      "echo 'proxy=http://xx.xx.xx.xx:xx' >> /etc/yum.conf",
      "echo 'proxy_username=\"\" >> /etc/yum.conf'",
      "echo 'proxy_password=\"\" >> /etc/yum.conf'",
      "sudo mkdir /tmp/vboxguest",
      "sudo mount -t iso9660 -o loop /VBoxGuestAdditions.iso /tmp/vboxguest",
      "cd /tmp/vboxguest",
      "sudo ./VBoxLinuxAdditions.run"
    ]
  }]
}
{% endhighlight %}