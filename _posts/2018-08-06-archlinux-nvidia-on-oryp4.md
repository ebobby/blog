---
layout: post
title: "HOWTO: Use Oryx Pro's discrete GPU in Archlinux"
date:  2018-08-06
tags: [linux]
---

_Although this has only been tested in [System76's Oryx Pro][oryp4] I suppose it
can be used in other distributions as well with a bit of work._

## Introduction

In a [previous post][previous] I explained how to use your discrete GPU for
specific applications while still running your main screen on your integrated
GPU.

This is pretty convenient because you don't have to reboot to play a game or use
your GPU in a specific application, but it made other things more difficult like
using external monitors since they are all wired to the discrete GPU.

In this post we will replicate [Pop! OS][popos] ability to boot exclusively into
your discrete GPU and have the ability to use external monitors (technically you
can use external monitors with bumblebee but it requires [some effort][multiple-mons]).

## No bumblebee

If you have installed bumblebee we have to get rid of it as it includes some
configuration to blacklist nvidia drivers and that will interfere with our set
up.

{% highlight console %}
sudo systemctl disable --now bumblebeed.service
sudo pacman -Rns bumblebee
sudo gpasswd -d fms bumblebee
{% endhighlight %}

## Install required software

You will need this packages to pull this off:

### Install drivers

There is usually a discussion about using nouveau and how [Nvidia][nvidia] is
evil and how much they suck but I still want to get as much of my video card as
I am able (with Linux) so I use their drivers.

{% highlight console %}
sudo pacman -S nvidia nvidia-utils nvidia-settings
{% endhighlight %}

### Install system76-power

This package is available in the [AUR][aur]. I personally use aurman.

{% highlight console %}
aurman -S system76-power
{% endhighlight %}

This utility allows you to switch GPU's, turn on the discrete card on or off, or
switch power profiles. Overall, if you have a System76 machine you want this.


### Configuration

Something important of note is that (at the time of this writing) the
nvidia-utils package includes the following (very important) configuration file:

{% highlight conf %}
# /usr/share/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf
Section "OutputClass"
    Identifier "intel"
    MatchDriver "i915"
    Driver "modesetting"
EndSection

Section "OutputClass"
    Identifier "nvidia"
    MatchDriver "nvidia-drm"
    Driver "nvidia"
    Option "AllowEmptyInitialConfiguration"
    Option "PrimaryGPU" "yes"
    ModulePath "/usr/lib/nvidia/xorg"
    ModulePath "/usr/lib/xorg/modules"
EndSection
{% endhighlight %}

It enables the modsetting driver and loads the nvidia driver (if
available). This fortunately doesn't get in the way when using the Intel GPU.

One thing to note is that using the modesetting driver breaks xbacklight. If you
don't use a desktop environment and rely on it to change the screen backlight
you will have to change how you do it.

### Telling Xorg to use the discrete GPU

In order to use your discrete GPU we depend on a technology called [PRIME][prime]. It
allows you to offload rendering to different GPUs, and more importantly for the
matter at hand, to have one GPU (integrated) to display the output rendered by
another GPU (your discrete).

Where you put the following instructions depends on how you run your X server.
Run startx manually

{% highlight shell %}
# Check if we are using nvidia graphics and tell xorg to get the
# output from NVIDIA-0.
if system76-power graphics | grep -q 'nvidia'; then
  xrandr --setprovideroutputsource modesetting NVIDIA-0
  xrandr --auto
  xrandr --dpi 96
fi
{% endhighlight %}

You can put them in `.xinitrc.`

### Using a desktop manager

Create the following script:

{% highlight shell %}
#!/bin/sh
#/usr/bin/setup-nvidia.sh

if system76-power graphics | grep -q 'nvidia'; then
  xrandr --setprovideroutputsource modesetting NVIDIA-0
  xrandr --auto
  xrandr --dpi 96
fi
{% endhighlight %}

And use one of the following methods mentioned [here][display-managers] as it is
dependant of which manager you use.

## Conclusion

That is it!. Now you just need to run:

{% highlight console %}
sudo system76-power graphics nvidia
{% endhighlight %}

Reboot.

Enjoy. If you can't get it working or have any suggestion, please reach out.

[oryp4]:         https://system76.com/laptops/oryx
[previous]:      https://ebobby.org/2018/07/15/archlinux-on-oryp4/
[popos]:         https://system76.com/pop
[multiple-mons]: https://wiki.archlinux.org/index.php/Bumblebee#Multiple_monitors
[nvidia]:        https://www.nvidia.com/
[aur]:           https://aur.archlinux.org/
[prime]:         https://forums.developer.nvidia.com/t/prime-and-prime-synchronization/44423
[display-managers]: https://wiki.archlinux.org/index.php/NVIDIA_Optimus#Display_Managers
