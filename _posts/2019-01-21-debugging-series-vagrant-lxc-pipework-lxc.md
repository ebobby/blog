---
layout: post
title: vagrant-lxc / pipework / lxc
subtitle: Debugging Series
date:  2019-01-21
tags: [debugging]
---
I personally consider "debugging" (figuring out how something works and how is
it broken) a very important skill as a software developer and one that I don't
think it is taught explicitly. It is simply left for us to figure it out.

For this reason I wanted to write about it for a while. Specially with examples
of real life problems and debugging sessions as I find they are far more useful
that simple made up examples.

This is the first in what likely will be a long series of posts as software
breaks all the time...

## Software breaks all the time...

I've been using vagrant and virtualbox to setup my development environments for
a few years now. It's a pretty convenient way to create reproducible
environments. Since I moved to linux though I've been using [vagrant-lxc][vagrantlxc] instead
of virtualbox as cointainers are just as isolated as virtual machines and they
are significantly faster.

Recently though, this stopped working as I tried to set up a new dev env. It
simply errored out.

{% highlight console %}
$ vagrant up
Bringing machine 'default' up with 'lxc' provider...
==> default: Importing base box 'emptybox/ubuntu-bionic-amd64-lxc'...
==> default: Checking if box 'emptybox/ubuntu-bionic-amd64-lxc' version '0.1.1546433870' is up to date...
==> default: Setting up mount entries for shared folders...
    default: /vagrant => /home/fms/Projects/local-devenv
==> default: Starting container...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 10.0.3.169:22
    default: SSH username: vagrant
    default: SSH auth method: private key
    default:
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default:
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
==> default: Setting up private networks...
There was an error executing ["sudo", "/usr/local/bin/vagrant-lxc-wrapper", "/home/fms/.vagrant.d/gems/2.6.0/gems/vagrant-lxc-1.4.3/scripts/pipework", "vlxcbr1", "devenv", "192.168.64.4/24"]

For more information on the failure, enable detailed logging by setting
the environment variable VAGRANT_LOG to DEBUG.
{% endhighlight %}

Yeeey... fun.

##Investigation

Apparently when we try to create our container, vagrant is failing to run a
command while trying to set up private networks. We have the command that it's
trying to run but we don't have any error output. Let's try vagrant suggestion
and check the debug log.

{% highlight console %}
$ VAGRANT_LOG=debug vagrant up
.... lots of output ....
==> default: Setting up private networks...
 INFO driver: Configuring network interface for devenv using 192.168.64.4 and bridge vlxcbr1
 INFO driver: Checking whether bridge vlxcbr1 exists
 INFO driver: Creating the bridge vlxcbr1
 INFO subprocess: Starting process: ["/usr/bin/sudo", "/usr/local/bin/vagrant-lxc-wrapper", "brctl", "addbr", "vlxcbr1"]
 INFO subprocess: Command not in installer, restoring original environment...
DEBUG subprocess: Selecting on IO
DEBUG subprocess: Waiting for process to exit. Remaining to timeout: 32000
DEBUG subprocess: Exit status: 0
 INFO driver: Checking whether the bridge vlxcbr1 has an IP
 INFO driver: Adding 192.168.64.254 to the bridge vlxcbr1
 INFO subprocess: Starting process: ["/usr/bin/sudo", "/usr/local/bin/vagrant-lxc-wrapper", "ip", "addr", "add", "192.168.64.254/24", "dev", "vlxcbr1"]
 INFO subprocess: Command not in installer, restoring original environment...
DEBUG subprocess: Selecting on IO
DEBUG subprocess: Waiting for process to exit. Remaining to timeout: 32000
DEBUG subprocess: Exit status: 0
 INFO subprocess: Starting process: ["/usr/bin/sudo", "/usr/local/bin/vagrant-lxc-wrapper", "ip", "link", "set", "vlxcbr1", "up"]
 INFO subprocess: Command not in installer, restoring original environment...
DEBUG subprocess: Selecting on IO
DEBUG subprocess: Waiting for process to exit. Remaining to timeout: 32000
DEBUG subprocess: Exit status: 0
 INFO subprocess: Starting process: ["/usr/bin/sudo", "/usr/local/bin/vagrant-lxc-wrapper", "/home/fms/.vagrant.d/gems/2.6.0/gems/vagrant-lxc-1.4.3/scripts/pipework", "vlxcbr1", "devenv", "192.168.64.4/24"]
 INFO subprocess: Command not in installer, restoring original environment...
DEBUG subprocess: Selecting on IO
DEBnUG subprocess: stderr: Found more than one container matching devenv.
DEBUG subprocess: Waiting for process to exit. Remaining to timeout: 32000
DEBUG subprocess: Exit status: 1
ERROR warden: Error occurred: There was an error executing ["sudo", "/usr/local/bin/vagrant-lxc-wrapper", "/home/fms/.vagrant.d/gems/2.6.0/gems/vagrant-lxc-1.4.3/scripts/pipework", "vlxcbr1", "devenv", "192.168.64.4/24"]

For more information on the failure, enable detailed logging by setting
the environment variable VAGRANT_LOG to DEBUG.
 INFO warden: Beginning recovery process...

 .... more output ....
{% endhighlight %}

We don't get much additional detail from that. So my next step is simply to try
to run the command by hand and see if I can catch anything.

(EDIT: It was pointed out to me that the error I was looking for is indeed
present in the debug log and I missed it.)

{% highlight console %}
$ sudo /usr/local/bin/vagrant-lxc-wrapper /home/fms/.vagrant.d/gems/2.6.0/gems/vagrant-lxc-1.4.3/scripts/pipework vlxcbr1 devenv 192.168.64.4/24
Found more than one container matching devenv.
$
{% endhighlight %}

A clue! pipework seems to think we have two containers with the same name.

{% highlight console %}
$ sudo lxc-ls
devenv
$ sudo ls /var/lib/lxc
devenv
{% endhighlight %}

Well. We don't. What is going on? Let's see if we can figure out what pipework
is trying to do. Thankfully, it is a bash script so we can simply open it in our
favorite text editor and search for "more than one", which brings us to this
piece of code:

(This code is included in vagrant-lxc 1.4.3, under scripts/pipework and starting in line 146.)

{% highlight shell %}
{% raw %}
# Second step: find the guest (for now, we only support LXC containers)
while read _ mnt fstype options _; do
  [ "$fstype" != "cgroup" ] && continue
  echo "$options" | grep -qw devices || continue
  CGROUPMNT=$mnt
done < /proc/mounts

[ "$CGROUPMNT" ] || {
    die 1 "Could not locate cgroup mount point."
}

# Try to find a cgroup matching exactly the provided name.
N=$(find "$CGROUPMNT" -name "$GUESTNAME" | wc -l)
case "$N" in
  0)
    # If we didn't find anything, try to lookup the container with Docker.
    if installed docker; then
      RETRIES=3
      while [ "$RETRIES" -gt 0 ]; do
        DOCKERPID=$(docker inspect --format='{{ .State.Pid }}' "$GUESTNAME")
        [ "$DOCKERPID" != 0 ] && break
        sleep 1
        RETRIES=$((RETRIES - 1))
      done

      [ "$DOCKERPID" = 0 ] && {
        die 1 "Docker inspect returned invalid PID 0"
      }

      [ "$DOCKERPID" = "<no value>" ] && {
        die 1 "Container $GUESTNAME not found, and unknown to Docker."
      }
    else
      die 1 "Container $GUESTNAME not found, and Docker not installed."
    fi
    ;;
  1) true ;;
  *) die 1 "Found more than one container matching $GUESTNAME." ;;
esac
{% endraw %}
{% endhighlight %}

It may look complicated if you haven't done shell scripting before, but if you
look at it closely it is relatively easy to find out what it is doing. Let's
dissect it.

In the line:

{% highlight shell %}
N=$(find "$CGROUPMNT" -name "$GUESTNAME" | wc -l)
{% endhighlight %}

It's trying to find something in the path `$CGROUPMNT` that is named after our
container "devenv" and if it finds more than one it returns the error we
got. What is `$CGROUPMNT`?

We find that here:

{% highlight shell %}
while read _ mnt fstype options _; do
  [ "$fstype" != "cgroup" ] && continue
  echo "$options" | grep -qw devices || continue
  CGROUPMNT=$mnt
done < /proc/mounts
{% endhighlight %}

It's iterating the contents of `/proc/mounts` and searching for anything that has
cgroup as file system and devices in the path.

Let's try that out:

{% highlight console %}
$ cat /proc/mounts | grep cgroup
tmpfs /sys/fs/cgroup tmpfs ro,nosuid,nodev,noexec,mode=755 0 0
cgroup2 /sys/fs/cgroup/unified cgroup2 rw,nosuid,nodev,noexec,relatime,nsdelegate 0 0
cgroup /sys/fs/cgroup/systemd cgroup rw,nosuid,nodev,noexec,relatime,xattr,name=systemd 0 0
cgroup /sys/fs/cgroup/blkio cgroup rw,nosuid,nodev,noexec,relatime,blkio 0 0
cgroup /sys/fs/cgroup/net_cls,net_prio cgroup rw,nosuid,nodev,noexec,relatime,net_cls,net_prio 0 0
cgroup /sys/fs/cgroup/cpu,cpuacct cgroup rw,nosuid,nodev,noexec,relatime,cpu,cpuacct 0 0
cgroup /sys/fs/cgroup/hugetlb cgroup rw,nosuid,nodev,noexec,relatime,hugetlb 0 0
cgroup /sys/fs/cgroup/rdma cgroup rw,nosuid,nodev,noexec,relatime,rdma 0 0
cgroup /sys/fs/cgroup/memory cgroup rw,nosuid,nodev,noexec,relatime,memory 0 0
cgroup /sys/fs/cgroup/perf_event cgroup rw,nosuid,nodev,noexec,relatime,perf_event 0 0
cgroup /sys/fs/cgroup/freezer cgroup rw,nosuid,nodev,noexec,relatime,freezer 0 0
cgroup /sys/fs/cgroup/devices cgroup rw,nosuid,nodev,noexec,relatime,devices 0 0
cgroup /sys/fs/cgroup/pids cgroup rw,nosuid,nodev,noexec,relatime,pids 0 0
cgroup /sys/fs/cgroup/cpuset cgroup rw,nosuid,nodev,noexec,relatime,cpuset 0 0

$ cat /proc/mounts | grep cgroup | grep devices
cgroup /sys/fs/cgroup/devices cgroup rw,nosuid,nodev,noexec,relatime,devices 0 0
{% endhighlight %}

So now we have the file path, let's run that find command that the script is running.

{% highlight console %}
$ find /sys/fs/cgroup/devices -name "devenv"
/sys/fs/cgroup/devices/lxc.payload/devenv
/sys/fs/cgroup/devices/lxc.monitor/devenv
{% endhighlight %}

It returns two entries which seem related and valid. So, what's going on?

At this point I am thinking that [lxc][lxc] itself, the container implementation, may
be the problem. vagrant-lxc hasn't been updated in a while.

Let's find out (this is [archlinux][archlinux] specific):

{% highlight console %}
$ cat /var/log/pacman.log | grep lxc | grep upgraded
[2018-11-28 10:24] [ALPM] upgraded lxc (1:3.0.2-1 -> 1:3.0.3-1)
[2018-12-17 17:05] [ALPM] upgraded lxc (1:3.0.3-1 -> 1:3.1.0-1)
{% endhighlight %}

So it seems that lxc was upgraded a few weeks ago (it's 2019-01 at the time of
this writing) and since I haven't used it during these weeks I didn't run into
this before.

My next step is to downgrade my installed version and see if that fixes this:

{% highlight console %}
$ sudo pacman -U lxc-1_3.0.3-1-x86_64.pkg.tar.xz
[sudo] password for fms:
loading packages...
warning: downgrading package lxc (1:3.1.0-1 => 1:3.0.3-1)

...

$ vagrant up
Bringing machine 'default' up with 'lxc' provider...
==> default: Importing base box 'emptybox/ubuntu-bionic-amd64-lxc'...
==> default: Checking if box 'emptybox/ubuntu-bionic-amd64-lxc' version '0.1.1546433870' is up to date...
==> default: Setting up mount entries for shared folders...
    default: /vagrant => /home/fms/Projects/local-devenv
==> default: Starting container...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 10.0.3.229:22
    default: SSH username: vagrant
    default: SSH auth method: private key
    default:
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default:
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
{% endhighlight %}

It works! So it was the newer version of lxc that broke my setup. Let's see what
this version has in the cgroup mount point.

{% highlight console %}
$ find /sys/fs/cgroup/devices -name "devenv"
/sys/fs/cgroup/devices/lxc/devenv
{% endhighlight %}

Yep. This returns one entry. The newer version changed that and it broke what
pipework expected. But now we know exactly what and we can fix it:

## The fix

{% highlight shell %}
{% raw %}
# Try to find a cgroup matching exactly the provided name.
M=$(find "$CGROUPMNT" -name "$GUESTNAME")
N=$(echo "$M" | wc -l)
case "$N" in
  0)
    # If we didn't find anything, try to lookup the container with Docker.
    if installed docker; then
      RETRIES=3
      while [ "$RETRIES" -gt 0 ]; do
        DOCKERPID=$(docker inspect --format='{{ .State.Pid }}' "$GUESTNAME")
        DOCKERCID=$(docker inspect --format='{{ .ID }}' "$GUESTNAME")
        DOCKERCNAME=$(docker inspect --format='{{ .Name }}' "$GUESTNAME")

        [ "$DOCKERPID" != 0 ] && break
        sleep 1
        RETRIES=$((RETRIES - 1))
      done

      [ "$DOCKERPID" = 0 ] && {
        die 1 "Docker inspect returned invalid PID 0"
      }

      [ "$DOCKERPID" = "<no value>" ] && {
        die 1 "Container $GUESTNAME not found, and unknown to Docker."
      }
    else
      die 1 "Container $GUESTNAME not found, and Docker not installed."
    fi
    ;;
  1) true ;;
  2)  # LXC >=3.1.0 returns two entries from the cgroups mount instead of one.
      echo "$M" | grep -q "lxc\.monitor" || die 1 "Found more than one container matching $GUESTNAME."
      ;;
  *) die 1 "Found more than one container matching $GUESTNAME." ;;
esac
{% endraw %}
{% endhighlight %}

We add a new case, if we get exactly two entries from find, check if one of them
is lxc.monitor, one of lxc 3.1.0 new additions, and if so, don't error out.

We bring lxc up to date, apply this patch by hand to vagrant-lxc pipework copy
and try again:

{% highlight console %}
$ vagrant up
Bringing machine 'default' up with 'lxc' provider...
==> default: Importing base box 'emptybox/ubuntu-bionic-amd64-lxc'...
==> default: Checking if box 'emptybox/ubuntu-bionic-amd64-lxc' version '0.1.1546433870' is up to date...
==> default: Setting up mount entries for shared folders...
    default: /vagrant => /home/fms/Projects/local-devenv
==> default: Starting container...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 10.0.3.100:22
    default: SSH username: vagrant
    default: SSH auth method: private key
    default:
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default:
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
{% endhighlight %}

Success!!

## Conclusion

We fixed our broken dev environment and everything is almost right in the world again.

The other thing we need to do, of course, is to submit [this patch][patch] to pipework
and submit another patch to vagrant-lxc to use this new version of pipework so
people that run it with lxc >3.1.0 have a working setup again.

[vagrantlxc]: https://github.com/fgrehm/vagrant-lxc
[lxc]:        https://linuxcontainers.org/
[archlinux]:  https://archlinux.org
[patch]:      https://github.com/jpetazzo/pipework/pull/231
