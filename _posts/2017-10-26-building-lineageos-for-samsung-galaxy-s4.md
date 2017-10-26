---
layout: post
title: "Building LineageOS for Samsung Galaxy S4"
date: 2017-10-26
---
### Security and older Android devices
It is typical for a phone manufacturer to only support their handsets
for a limited time. Companies have an incentive to push their newer
devices, lest they risk no longer making sales and going out of
business. As a result, customers are eventually left with devices that
no longer receive software updates and contain known
vulnerabilities that go unpatched.

My phone is a Galaxy S4, the jfltetmo variant. This phone was
[released in 2013](http://www.ibtimes.co.uk/samsung-galaxy-s4-launched-new-york-446284)
and had its final software update in 2015 to
[Android 5.0.1](http://www.androidne.ws/234146/atts-samsung-galaxy-s4-is-receiving-android-5-0-1-lollipop-update.html). There are currently
[288 known CVEs](http://www.cvedetails.com/vulnerability-list/vendor_id-1224/product_id-19997/version_id-188442/Google-Android-5.0.1.html)
for this version of Android, where 121 CVEs have a
[CVSS score](https://www.first.org/cvss/) of 9+. That is a bleak picture for
anyone with a Galaxy S4.

One way to avoid the security issues related to having an older device is to
install an open source variant of Android which is
up-to-date. For this article I'll detail how I went about doing just that.

### LineageOS

[LineageOS](https://lineageos.org/) is an open source Android project which
maintains an Android distribution for a number of devices. Each maintained
device has at least one maintainer, so that if a build is provided
for a device there is at least one person with that device who is a part
of LineageOS. In theory this should allow issues to be caught by
the LineageOS developers, or at least if issues do occur a LineageOS developer
should have the resources to resolve the issue.

The project does have builds for the Galaxy S4 available. However, my specific
variant, the jfltetmo, is not supported. To produce a build, I'll need to create one
for myself.

### Building LineageOS for the jfltetmo

The build instructions for a similar Galaxy S4 are available on the
[LineageOS wiki](https://wiki.lineageos.org/devices/jfltexx/build). The
following details the process I went through to produce the build.

To download the source, one will need to use the `repo` tool. This tool
is responsible for collecting all the git repos necessary for a given
branch. The tool is written by Google, and can be downloaded and put
into the PATH:

{% highlight shell %}
$ mkdir -p ~/bin
$ PATH=$PATH:~/bin
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
{% endhighlight %}

Next, time to download the latest released branch from LineageOS for
the Galaxy S4. In this case, it is the cm-14.1 branch:

{% highlight shell %}
$ mkdir -p lineage
$ cd lineage
$ repo init -u https://github.com/LineageOS/android.git -b cm-14.1
{% endhighlight %}

This does not download all of the individual git repos, but instead
populates the directory with folders which are unfetched git repos.
The following then downloads the contents of the git repos and checks
them out to the cm-14.1 branch. Note that this will take some time,
based on one's Internet connection.

{% highlight shell %}
$ repo sync
{% endhighlight %}

At this point it is time to prepare the device specific code. Note that
for some devices, which also includes the Galaxy S4, there are binaries provided
by the device manufacturer for which there is no source. As a result, in order
to make a build those binaries will need to be used.

Before those binaries are retrieved, the device specific code is setup as much
as it can without the binaries. The following starts this process. The first
part of the breakfast command will report errors about missing Makefiles.
This is expected, and will be corrected shortly.

{% highlight shell %}
$ source build/envsetup.sh
$ breakfast jfltexx
{% endhighlight %}

Now to get the missing binary files. One could use an existing phone
and extract the binaries from it. I do have a phone that works. However,
it turned out to not contain all the binaries that LineageOS expects to
exist. This may be because my phone was running another open source Android
variant which did not package all of Samsung's binary files.

To get all the binary files, I opted to download a LineageOS image for
a similar Galaxy 4 which was supported and extract them.
[This link](https://wiki.lineageos.org/extracting_blobs_from_zips.html)
gives details about the following steps.

First, I downloaded a [jfltexx image](https://download.lineageos.org/jfltexx)
in a separate directory outside of the source tree:

{% highlight shell %}
$ cd ..
$ mkdir system_dump
$ cd system_dump
$ wget https://mirrorbits.lineageos.org/full/jfltexx/20171025/lineage-14.1-20171025-nightly-jfltexx-signed.zip
$ unzip lineage-14.1-20171025-nightly-jfltexx-signed.zip
{% endhighlight %}

This zip file contained a block based OTA system image, system.new.dat, and
a file which describes the image, system.new.dat. These two files need to be
processed to create an ext4 image which can be mounted in Linux. There are
tools on GitHub in the xpirt/sda2img project to accomplish this:

{% highlight shell %}
$ git clone https://github.com/xpirt/sdat2img
$ python sdat2img/sdat2img.py system.transfer.list system.new.dat system.img
{% endhighlight %}

The system.img file is an ext4 file system image. It can be mounted, which allows
the image's file system to be explored:

{% highlight shell %}
$ mkdir -p system
$ sudo mount system.img system
{% endhighlight %}

Now that the file system is extracted, the necessary binary files can be
copied into the build. In the source tree navigate to the
device/samsung/jfltexx directory. An extract-files.sh script will take
care of copying the binary files.

{% highlight shell %}
$ cd ../lineage/device/samsung/jfltexx
$ ./extract-files.sh ../../../../system_dump/system
# Go back to the top of the soruce tree
$ cd ../../..
{% endhighlight %}

With the binary files in place, the breakfast command can be rerun, which will
complete the build setup.

{% highlight shell %}
$ breakfast jfltexx
{% endhighlight %}

As of this writing the jfltexx build will not flash to a jfltetmo. This
is because the jfltexx build does not declare it can be used on a jfltetmo.
The jfltexx build is used for several other Galaxy S4 variants, so it is
worth trying on the jfltetmo and should work. To allow the jfltexx build to
install on the jfltetmo, the BoardConfig make file for the jfltexx needs to
be modified. This file is located:

{% highlight shell %}
$ vi device/samsung/jfltexx/BoardConfig.mk
{% endhighlight %}

and the following change needs to be made:

{% highlight makefile %}
# Assert
- TARGET_OTA_ASSERT_DEVICE := jfltexx,i9505,GT-I9505,jgedlte,i9505g,GT-I9505G,jflte
+ TARGET_OTA_ASSERT_DEVICE := jfltexx,i9505,GT-I9505,jgedlte,i9505g,GT-I9505G,jflte,jfltetmo
{% endhighlight %}

With this change, time to start the build.

{% highlight shell %}
$ croot
$ brunch jfltexx
{% endhighlight %}

If the build is successful, you should see something similar to the following at the end:

{% highlight shell %}
[100% 1363/1363] build bacon                                                    
Package Complete: /home/zatoichi/lineage/out/target/product/jfltexx/
lineage-14.1-20171026_034737-UNOFFICIAL-jfltexx.zip                             
make: Leaving directory '/home/zatoichi/lineage'                    

#### make completed successfully (18:08 (mm:ss)) ####   
{% endhighlight %}

With the build completed, time to flash the build to the phone. See
[these](https://wiki.lineageos.org/devices/jfltexx/install) instructions for
details. Remember to backup before flashing, in case it does not work out.
