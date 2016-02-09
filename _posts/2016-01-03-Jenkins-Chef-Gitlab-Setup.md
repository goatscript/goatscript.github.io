---
layout: post
title: Cookbook CI with Jenkins, Gitlab and Docker
---

My current cookbook development environment consists of a local Gitlab repo which Jenkins queries, runs automated tests and uploads to the Chef server upon passing.  The Chef server and Gitlab repo run as KVM guests on a CentOS 6 host.  The Jenkins server and Chef workstation are seperate descrete CentOS 6 machines.

## Jenkins Intall

Lets begin with the Jeninks install.

- yum update
- check java version
{% highlight bash %}
java -version
output:
java version "1.7.0_85"
OpenJDK Runtime Environment (rhel-2.6.1.3.el6_7-x86_64 u85-b01)
OpenJDK 64-Bit Server VM (build 24.85-b03, mixed mode)
{% endhighlight %}

- Download and Install Jenkins
{%highlight bash %}
wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo
rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
yum install jenkins
{% endhighlight %}

- Update iptables
{% highlight bash %}
iptables -I INPUT -p tcp -m tcp --dport 8080 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 80 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT
service iptables save
{% endhighlight %}

- Start Jenkins and enable service
{%highlight bash %}
service jenkins start
chkconfig jenkins on
{% endhighlight %}

## Chef Server Install

- Create a machine with a fqdn.  I'm using a KVM CentOS 6.6 guest named grant.lan.
- Add the fqdn to /etc/hosts
{% highlight text %}
echo "ip_address fqdn_hostname" >> /etc/hosts
{% endhighlight %}

- Download Chef server 12
- Install Chef server 12
{% highlight bash %}
rpm -Uhv chef-server-12.rpm
{% endhighlight %}

- Reconfigure the Chef server
{% highlight bash %}
run chef-server-ctl reconfigure
{% endhighlight %}

- Create the Chef server admin.  Replace username and user with appropriate values.
{% highlight bash %}
chef-server-ctl user-create username first_name last_name email_address@server.com password --file /home/user/user.pem
{% endhighlight %}
- Create the Chef Organization and associate a user.
{% highlight text %}
chef-server-ctl org-create org_short_name "org_name" --association_user username --file /home/user/org-validator.pem
{% endhighlight %}

## Create Chef workstation on CentOS 6.

- Add hostname to /etc/hosts
- Download the chefdk rpm from Opscode
- Install chefdk rpm
{% highlight bash %}
rpm -Uhv chefdk.rpm
{% endhighlight %}

- Verify the setup is correct
{% highlight bash %}
chef verify
{% endhighlight %}

- Add ruby to the .bash_profile and refresh the bash profile.  Alternatively, logging in and out will update the bash profile.
{% highlight text %}
echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile
. ~/.bash_profile
{% endhighlight %}

- Install and setup git.  Check https://git-scm.com for guidance.
- Create the local chef-repo
{% highlight bash %}
chef generate repo chef-repo
{% endhighlight %}

- Copy the validator.pem and admin.pem locaed on the chef server to ~/chef-repo/.chef.  If missing, create the .chef directory.


- Create a knife.rb file in .chef.
{% highlight ruby lineno %}
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                "user"
client_key               "#{current_dir}/user.pem"
validation_client_name   "org-validator"
validation_key           "#{current_dir}/org-validator.pem"
chef_server_url          "https://chef_server_fqdn.lan:443/organizations/org"
cookbook_path            ["#{current_dir}/../cookbooks"]
{% endhighlight %}

- At this point, I ran knife ssl per the Chef docs and got the following error.
{% highlight text %}
knife ssl fetch
output:
	WARNING: Certificates from grant.lan will be fetched and placed in your trusted_cert
	directory (/root/chef-repo/.chef/trusted_certs).

	Knife has no means to verify these are the correct certificates. You should
	verify the authenticity of these certificates after downloading.

	ERROR: Errno::EHOSTUNREACH: No route to host - connect(2) for "grant.lan" port 443
{% endhighlight %}
Since this is a test environment, just turn off the ssl verificaiton.
{% highlight text %}
echo 'ssl_verify_mode :verify_none' >> /etc/chef/client.rb
cat /etc/chef/client.rb
output:
log_level        :info
log_location     STDOUT
chef_server_url  'https://chef_server_fqdn.lan:443/organizations/org'
validation_client_name 'org-validator'
ssl_verify_mode :verify_none
{% endhighlight %}
- Run the chef-client
{% highlight bash %}
chef-client
{% endhighlight %}

- Verify the node shows up in the list
{% highlight bash %}
knife node list
{% endhighlight %}



