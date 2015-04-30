Welcome to vmprestage!
======================


Summary
---------

This is a small helper script that you can utilize to copy VirtualBox virtual machines running on a Linux host from one or more directories in which you may hold those machines, into a pre-staging area from which backup tools such as the excellent [BRU](http://www.tolisgroup.com) can pick them up. I call this process pre-staging, since backup tools like BRU have their own staging mechanism, and this tool is typically run in advance. The script follows the idea of differentiating between machines that are frequently used (or, "hot"), those that are unfrequently used ("warm") and finally those that may mostly not ever be used ("cold"). This is a classification that you make. In my example, I hold virtual machines in directories, or source locations, like:

```
/data-intern/vmware
/data-extern/vmware
```

and I have hence subdirectories per classification:

```
/data-intern/vmware/hot
/data-intern/vmware/warm
/data-intern/vmware/cold
/data-extern/vmware/hot
/data-extern/vmware/warm
/data-extern/vmware/cold
```

Notice that I use `vmware` as directory name only for historical reasons; the script here works specifically with VirtualBox. So as one example, I may have a virtual machine HOME_HTTPD, that I keep always running, inside e.g.

```
/data-intern/vmware/hot/HOME_HTTPD
```

For my pre-staging area I have

```
/data-backup/bru-prestage/vms-hot
/data-backup/bru-prestage/vms-warm
/data-backup/bru-prestage/vms-cold
```

When I want to run the pre-stage process, I want the following to happen:

1. Get a list of virtual machines that are currently running
2. For each of the source locations, locate virtual machines that are there
3. If any of those virtual machine directories contains a virtual machine that is currently running, pause it, copy its files to the pre-staging area, and then resume it
4. For all remaining directories, i.e. those that may contain virtual machines that are not running or some completely different content, copy it over to the pre-staging area
5. Since I want to maintain knowledge about where the virtual machine files originally were located, and I may have multiple source directories, I reproduce - somewhat redundantly - the whole path, so that e.g.

```
/data-intern/vmware/hot/HOME_HTTPD
```
becomes

```
/data-backup/bru-prestage/vms-hot/data-intern/vmware/hot/HOME_HTTPD
```


Installation
---------

Copy the script to some directory where you hold your executables, like `/usr/local/bin` or some other location that is in your path, and make it executable:

```
chmod 755 /usr/local/bin/vmprestage
```

For you Ubuntu users, I assume you are root. Make sure that you have the `vboxmanage` tool of VirtualBox in your path. That should be the case already by a default VirtualBox installation.


Make sure that you have Bash Version 4:

```
# bash --version
bash --version
GNU bash, Version 4.3.8(1)-release (x86_64-pc-linux-gnu)
```

This will make it typically not work on MacOS; I was [too lazy](http://c2.com/cgi/wiki?LazinessImpatienceHubris) to write the script in Perl, and wanted to utilize associative arrays. Feel free to do that differently.

Apart from this, you need the obvious candidates such as find, rsync, mkdir, in other words, nothing surprising.


Configuration
---------

The configuration of the script happens at the top. You have to set the location of your VirtualBox.xml configuration file:

```
VBOXCFG=/root/.VirtualBox/VirtualBox.xml
```

This file specifically contains the location of virtual box machines.

Then, you need to configure the locations inside which you hold your virtual machines, as a colon-separated list:

```
SOURCES=/data-intern/vmware:/data-extern/vmware
```

Finally, you need to configure where you have your pre-staging area:

```
STAGEAREA=/data-backup/bru-prestage/vms-
```

To that path, the words `hot`, `warm`, and `cold` are going to be appended.


Preparation
---------

Once configured, you may need to relocate your virtual machines in order to comply with the `SOURCES` locations specified above. The good thing is, you can do that. The bad things are, you will need to pause all virtual machines for that, and VirtualBox does not tell you how to do it - so I do. This process works only with VirtualBox version 4: Since version 4, all virtual machine information is contained in a machine's directory, and only one file contains the list of all virtual machines: the `VirtualBox.xml` specified above by `VBOXCFG`. 

***Disclaimer*** 

*You really should know what you are doing if you do this on a  productive system. I don't guarantee that this is going to work on your end, of course.*

So first of all, we will record a list of currently running virtual machines into a file `/tmp/savedvms` and then stop them:

```
rm /tmp/savedvms
for i in $(ps ax | grep VBox | grep comment | grep -v grep | awk '{ print $7};'); do echo Suspending $i...; echo $i >> /tmp/savedvms; vboxmanage controlvm $i savestate; done
```

If you don't like that kind of scripting, you are of course welcome to just do that by hand.

Next, we are going to modify the file VirtualBox.xml. It contains elements like

```
<MachineEntry uuid="{0ac5b1ed-af7d-48ba-85e6-3ba7091d2d35}" src="/data-intern/vmware/HOME_HTTPD/HOME_HTTPD.vbox"/>
```

which we would according to our strategy change to, e.g.,

```
<MachineEntry uuid="{0ac5b1ed-af7d-48ba-85e6-3ba7091d2d35}" src="/data-intern/vmware/hot/HOME_HTTPD/HOME_HTTPD.vbox"/>
```

Now, the whole business about stopping all virtual machines is that the virtual box service (vbox service) holds a copy of that file's data in memory, and I found no way to tell it to be so kind as to re-read the data from disk. If you find that out, you qualify for a beer. Otherwise, we kill the service:

```
ps ax | grep VBoxSVC | grep -v grep | awk '{ print $1;}' | xargs kill -1
```

If you have [PHPVirtualBox](http://sourceforge.net/projects/phpvirtualbox/), you may also stop it's service at this point:

```
/etc/init.d/vboxweb-service stop
```

Since we are at that, I could also have chosen to stop the other service correctly instead of killing it, but then you need to maintain the dependency, or order in which to do it:

```
/etc/init.d/vboxautostart-service stop
/etc/init.d/vboxballoonctrl-service stop
/etc/init.d/vboxweb-service stop
/etc/init.d/vboxdrv stop
```

and then restart them:

```
/etc/init.d/vboxdrv start
/etc/init.d/vboxautostart-service start
/etc/init.d/vboxballoonctrl-service start
/etc/init.d/vboxweb-service start
```

The `kill -1` was of course a poor man's test to have it re-read its configuration.

Now with the services all back up again, it is time to resume the virtual machines:

```
for i in $(cat /tmp/savedvms); do echo Resuming $i...; vboxmanage startvm $i -type vrdp; done
rm /tmp/savedvms
```

Verify now that your machines are working according to your new locations. Notice, also, that I used `-type vrdp` since the whole context of this idea is of course that you have your virtual machines running in a server kind of fashion, and not from the VirtualBox desktop client.

Also, at this time make sure that your pre-staging area is there and available, as you configured by the `STAGEAREA` variable.


Usage
---------

Using the tool is very simple:

```
vmprestage hot
```

Alternatively, you use `warm` or `cold` as parameter. You can also use `-h` for help, and the parameter parsing function that I have reused from [vcompress](https://github.com/mnott/vcompress) is of course complete overkill, but I may want to use some more tuning later.


Tips & Tricks
---------

VirtualBox allows to create snapshots, which is normally a good way to capture states of your virtual machines that you have confirmed to be, for example, working in some sense. The technical effect of creating snapshots is that with that snapshot created, all subsequent changes to disk files are not going into the main disk files, but into snapshot files, until you merge back the snapshot or delete it.

While snapshots have some other challenges - e.g. they need to be merged before you can use the excellent [CloneVDI](https://forums.virtualbox.org/viewtopic.php?f=6&t=22422) tool to compress your virtual machine files - they create an awesome opportunity for our case to speed up the copying process:

For every machine that you regularly copy out of a running state - i.e., you suspend it, copy the files, and then resume it, create a snapshot first. This will mean that your subsequent changes to disk files will go to a snapshot file, and hence the copy duration will be relatively short as compared to copying the complete container. So, for example, once you have set up a virtual machine and hence most data in a central disk file, create a snapshot and name it e.g. "Working for Backup" - from that point on, while the machine is running, changes to disk are going into the snapshot file, which is normally smaller than the main disk file or files. The snapshot file will also grow at some point, and hence you have an incentive to re-merge that snapshot regularly - which will then be an easy thing for you to do, since because of a clear naming of that snapshot, you will know that there are no functional changes that you might also want to discard.

Creating a snapshot means that you will have a one-time lengthy process of copying all the files (and you would have that anyway), but since we are using rsync, subsequent copy processes will be very fast.


Caution
---------

By the way I set up the script, as I use a directory by directory process, if you would later move a virtual machine from one location to another, e.g., from a `hot` to a `warm` location, this script would not remove the file from the `hot` pre-staging area. I may change that later, but have not done that so far. Feel free to grow the script there.



Background
---------


Copying virtual machine container (disk) files is a long process. For that very reason, BRU has a concept of agents where you deploy an agent into your virtual machine, and the BRU server process will then contact that client as part of a backup process and fetch data from that client while the client keeps running. This is a truly great function, and is particularly valid for scenarios where the data in the client is in overseeable locations. In that case, you'd do a container backup infrequently, and then fetch only the data that you change from inside that container.

Virtual machines on my end have a somewhat different purpose: They never really contain user data in the first place. They are more functional containers around things that I do, e.g. providing for an isolated runtime environment for a given project; the actual data will normally reside outside that container. An exception is if the container holds e.g. a database that will locally store its data, and where I want to have the database files within that container; notice that all these are typically applications that are not performance relevant. I do a lot of development, and hence am oriented towards having a given functional container, i.e., my virtual machine, always consistent and independent from its outside world, except for e.g. user data that it may want to create. Another example is a web server where I then have a cron job inside the container virtual machine extracting the database content and static files and push it out to some location external to that container, for backup.

Having said that, in my use case, virtual machines are for me things that I configure once and then want to forget about what I did - in other words, I don't care so much about the data they hold (which is either very little, or it is outside the container anyway), but I do care a lot having a functioning copy of that container virtual machine.

Therefore, for my use case, I rather need complete copies of a given virtual machine so that I can go back to any version of the virtual machine should I need to.

That's why I needed to come up with a way to automatically stage the virtual machine container files to some area where BRU can pick them up. And since it does not make sense to copy container files that are currently in use, I need a way to suspend the virtual machines, copy the files, and then resume them. Since I want to have a minimal downtime, I treat first the virtual machines that are running, and suspend, copy and resume them on a one by one basis, before copying other machines that are not running, and other content.

And since I have a *lot* of virtual machines, I wanted to have a classification allowing me to separate between things I always need - hot -, things I rarely need - warm -, and things that are mostly never used again, like old project environments or templates.




----------

Parameters
---------

Parameters can be used in any order. Which is funny, as we have one parameter only at this point, but we keep this area here for adding more, in the future.

```
# ./vmprestage
Usage: ./vmprestage [options] [hot|warm|cold]

-h| --help                 :  Print this help.
```

