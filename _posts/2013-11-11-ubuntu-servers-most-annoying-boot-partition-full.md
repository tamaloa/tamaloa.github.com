---
layout: post
title: "Working around ubuntu server's most annoying /boot partition full"
excerpt: "Ubuntu server editions have the annoying habbit of not removing old unneeded kernels. With unattended security updates activated this leads to filled up /boot partitions which can be really nasty. This script removes all unneeded kernel images and headers."
description: ""
category: 
tags: [ubuntu, server]
---
{% include JB/setup %}

Ubuntu server editions (up to 12.04LTS) are mostly easy to take care of.
With unattended security updates activated (as recommended by the
installer) it is okay to focus on your own applications and to rely on
the main system being okay.
Ubuntu frequently releases updated kernel images which are automatically
installed. This works very well and old images are kept just in case the
newly installed image fails.
Unfortunately older kernel images are kept indefinitely. This leads to
the /boot partition slowly filling up. Additionally the default
partioning (at least with a 100GB disk) provisions only \~230M for
/boot. Thus in under a year /boot is filled which may cause the whole
system to lock down (google for "ubuntu /boot partition full" and be
shocked :)).

It seems wise to keep a few older images around just in case, but not 20
or more! The ubuntu guys seem to have noticed this as well [^1] and
newer Ubuntu releases (since 13.04) have a single command
`sudo apt-get autoremove --purge` to remove all old kernels except for
the most recent two. Unfortunately there exists no single script to call
in the most recent server-LTS and selecting all kernels by hand for
several servers is really annoying. \[UPDATE\] I am now experiencing the
same problem of /boot filling up on 14.04 server machines and the before
mentioned single command does not take care of this. So the following
script is also used for 14.04 servers.\[/UPDATE\]

The scripts on the web either removed all old kernels (keeping only the
current and thus leaving no rollback opportunity) [^2] or deleted all
except for the youngest two (the current kernel could be older) [^3].
Thus i put together the following script which purges all kernel images
and headers except for the current running kernel (which is the output
of uname -r), the actual base kernel and the two last kernels in the
listing (propably the most recent ones and propably at least one has run
successfully before :).

    #!/bin/bash

    dpkg --get-selections | `#show all installed packages` \
      grep 'linux-image-*' | `#select all installed images` \
      awk '{print $1}' | `#select only package name)` \
      egrep -v "linux-image-$(uname -r)|linux-image-generic" | `#remove current and base kernel from list` \
      head -n -2 | `#remove two recent kernels from list` \
      sed 's/^linux-image-\(.*\)$/\1/' | `#capture image version` \
      while read n
      do 
        echo 'Purging unneeded kernel images and headers for: '$n
        sudo apt-get --yes purge linux-image-$n    #purge images
        sudo apt-get --yes purge linux-headers-$n  #purge headers
      done

If added to roots crontab via

    @reboot /root/purge-unneeded-kernels.sh

\
this hopefully keeps /boot well below 50% df.

Note - the "adding comments to multiline shell command via backticks"
worked for me in some cases but in some i had to remove them to run the
script :/

[^1]: [lists.ubuntu.com Distro-provided mechanism to clean up old
    kernels](https://lists.ubuntu.com/archives/ubuntu-devel/2012-February/034775.html)

[^2]: [ubuntugenius.wordpress.com
    Blogpost](http://ubuntugenius.wordpress.com/2011/01/08/ubuntu-cleanup-how-to-remove-all-unused-linux-kernel-headers-images-and-modules/)

[^3]: [lists.ubuntu.com
    shellscript](https://lists.ubuntu.com/archives/ubuntu-devel/2012-February/034767.html)
