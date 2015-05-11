---
layout: post
title: Managing DUL's ArchivesSpace Environment
author: Derrek Croney
tags: linux ArchivesSpace
category: ArchivesSpace
---

ArchivesSpace is the system selected by Rubenstein Technical Services (aka RTS) to replace Archivists Toolkit to facilitate the management of Digital Collection Finding Aids.

### Setting Up ArchivesSpace (aka Upgrading ArchivesSpace)

When notified by Rubenstein Technical Services to upgrade the ArchivesSpace software, these are the steps I currently perform:

#### Downloading latest software

Unless instructed otherwise by RTS, the latest version of ArchivesSpace is located here:
[https://github.com/archivesspace/archivesspace/releases]()

Look for a file whose name resembles this pattern: `archivesspace-vX.X.X.zip` (where X.X.X represents a version number).  When located, use either 'wget' or 'curl' to download it directly to the server:

Let's take version 1.2.0 as an example:

{% highlight bash %}
curl -o archivesspace.zip https://github.com/archivesspace/archivesspace/releases/download/v1.2.0/archivesspace-v1.2.0.zip
{% endhighlight %}

On the production server, the complete process (including stopping the current service) looks like this:

{% highlight bash linenos %}
sudo -u webadmin -i
cd /srv/apps/archivesspace
./archivesspace.sh stop
cd ..
mv archivesspace archivesspace.old
curl -o archivesspace-v1.2.0.zip https://github.com/archivesspace/archivesspace/releases/download/v1.2.0/archivesspace-v1.2.0.zip
unzip archivesspace-v1.2.0
mv archivesspace archivesspace-1.2.0

# this step is done with future upgrades in mind (optional)
ln -s archivesspace-1.2.0 archivesspace
{% endhighlight %}

#### Configuring the software
Following the directions outlined in the README ([located here](https://github.com/archivesspace/archivesspace)) will make it easy to install and configure the software.

That said, we have some minor tweaks of our own, particularly the assigned ports:

{% highlight ruby linenos %}
AppConfig[:backend_url] = "http://localhost:9089"
AppConfig[:frontend_url] = "http://localhost:9080"
AppConfig[:solr_url] = "http://localhost:9090"
AppConfig[:public_url] = "http://localhost:9081"
{% endhighlight %}
