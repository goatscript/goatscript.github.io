---
layout: post
title: Cookbook CI - Jenkins, Gitlab and Docker prt. 2
img:
  medium:	jenkins-setting.png
---

This continues the setup of Chef cookbook development CI solution I'm using at home and work.  With the Jenkins, Gitlab, Chef and Docker installations complete, the shift of this blog is on the final setup of the CI workflow.  As part of the example, I'll be using the gsntp cookbook found on my [github](https://github.com/goatscript/gsntp) account.  There is vast amounts of blogs, tutorials and official documentation on how to run Chef and Test Kitchen.  This blog assumes some basic understanding of how to run Test Kitchen and Chef.

## Create docker registry

This step isn't mandatory, but for accuracy I store docker images on a local registry.  The setup is strait forward and well documented on the [Docker website](https://docs.docker.com/registry/deploying/).

First setup the local registry.

{% highlight bash %}
docker run -d -p 5000:5000 --restart=always --name registry -v `pwd`/data:/var/lib/registry registry:2
{% endhighlight %}

Then pull down the official CentOS image.
{% highlight bash %}
docker pull centos
{% endhighlight %}

Push to the local registry
{% highlight bash %}
docker push localhost:5000/centos
{% endhighlight %}

## Create the .kitchen.yml file

Add the docker driver and docker cli pathway to .kitchen.yml.  Use chef-zero and test the default cookbook.  A later blog post will go over proxy settings workarounds for docker, vagrant and packer.

- sample .kitchen.yml file
{% highlight bash %}
---
	driver:
	  name: docker
	  use_sudo: false
	  docker: /usr/bin/docker
#
	provisioner:
	  name: chef_zero

	platforms:
	  - name: centos-6.4

	suites:
	  - name: default
	    run_list:
	      - recipe[gsntp::default]
	    attributes:
{% endhighlight %}

- Run Test Kitchen.  If you run into the following nokogiri error make sure the chefdk is >= 0.10.0.
{% highlight bash %}
An error occurred while installing nokogiri (1.6.6.4), and Bundler cannot continue.
Make sure that `gem install nokogiri -v '1.6.6.4'` succeeds before bundling.
{% endhighlight %}

- Once Test Kitchen passes, the next step is to setup the Jenkins job.

## Create the Jenknis job.

Set Jenknins to monitor the gsntp gitlab repo.

- Add the Git plugin to Jenkins.
- Create a Jenkins job for the cookbook
- Edit the configuration file by clicking Configure from menu
- Under Source Code Management choose the Git bullet point
- Enter the repository url git@xx.xx.xx.xx:/user/cookbook.git
- Enter gitlab as the repository browser
- The URL is http://xx.xx.xx.xx:/user/cookbook
- The Version is 7.0 or whichever version you happen to use
- For simplicity sake set the Build Triggers to Poll SCM every minute
- Under the Build, select Execute shell and run the following commands
{% highlight bash %}
cd /path/to/cookbook
test kitchen
knife cookbook upload cookbook-name
{% endhighlight %}

Below is a sample Jenkins configuration.

![jenkins configuration](/assets/jenkins-setting.png)

- At this point Jenkins should monitor the gitlab cookbook repo.  When changes are pushed to gitlab, Jenkins will test the cookbook and push the changes to the chef server upon passing.










{% highlight bash %}

{% endhighlight %}