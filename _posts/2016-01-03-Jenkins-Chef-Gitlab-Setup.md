---
layout: post
title: Cookbook CI - Jenkins, Gitlab and Docker prt. 1
---

My current home cookbook development environment consists of a local Gitlab repo which Jenkins queries, runs automated tests and uploads to the Chef server upon passing.  For demo purposes, the Chef server and Gitlab repo run as KVM guests on a CentOS 6 host.  The Jenkins server and Chef workstation are separate discrete CentOS 6 machines.

UPDATE:  A similar setup exists now in my work environment.

## Jenkins Intall

- yum update
- check java version
{% highlight bash %}
java -version
output:
java version "1.7.0_85"
OpenJDK Runtime Environment (rhel-2.6.1.3.el6_7-x86_64 u85-b01)
OpenJDK 64-Bit Server VM (build 24.85-b03, mixed mode)
{% endhighlight %}

- Download and install Jenkins
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

- Start Jenkins and add to chkconfig
{%highlight bash %}
service jenkins start
chkconfig jenkins on
{% endhighlight %}

## Chef Server Install

- Build a machine, making sure to use a fqdn.  My test machine runs CentOS 6.6
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
- Download the chefdk rpm from [Opscode](https://downloads.chef.io/chef-dk/)
- Install chefdk rpm
{% highlight bash %}
rpm -Uhv chefdk.rpm
{% endhighlight %}

- Verify the setup is correct
{% highlight bash %}
chef verify
{% endhighlight %}

- Add ruby to the \.bash\_profile and refresh the bash profile.  Alternatively, logging in and out will also refresh the .bash_profile.
{% highlight text %}
echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile
. ~/.bash_profile
{% endhighlight %}

- Install and setup git.  Check the [Git site](https://git-scm.com for guidance) for guidance.
- Create the local chef-repo
{% highlight bash %}
chef generate repo chef-repo
{% endhighlight %}

- Copy the validator.pem and admin.pem located on the chef server to ~/.chef.  If missing, create the .chef directory.


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
Since this is a test environment, just turn off the ssl verification.
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

## Gitlab Install and Setup

For simplicity sake, Gitlab is essentially a local Github-like repository.  This provides a solution when Github functionality is needed within a private cloud.  Visit [Gitlab](https://about.gitlab.com/) for more information.

- Follow the [official Gitlab install for CentOS 6](https://about.gitlab.com/downloads/#centos6)

- Create a new repository on Gitlab for the test cookbook.

- Navigate to the test cookbook and setup gitlab remote

{% highlight bash %}
git remote add origin gti@chef_workstation:path_to_cookbook/test_cookbook.git
{% endhighlight %}

- Push to master branch

{% highlight bash %}
git push -u origin master
{% endhighlight %}

## Docker Install and Setup
Basic install of Docker on CentOS 6.
- Add the epel repo
{% highlight bash %}
rpm -iUhv http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
{% endhighlight %}

- After a yum update, install docker
{% highlight bash %}
yum install -y docker-io
{% endhighlight %}

- Start the docker service
{% highlight bash %}
service docker start
{% endhighlight %}

- Add docker to chkconfig
{% highlight bash %}
chkconfig docker on
{% endhighlight %}