---
layout: post
title: Installing Archlinux on System76's 2018 Oryx Pro (oryp4)
date:  2018-07-15
tags: [linux]
---

I recently decided to move from OSX to Linux and settled on a System76 new Oryx
Pro (Jun 2018). They officially support Ubuntu and their own [Pop! OS][popos]
which work really well. I decided to use Archlinux though as it is my
distribution of choice.

This post contains the details of how I managed to set it up and get all the
functionality you get in Pop! OS.

## Turn off discrete graphics card

This machine has a Nvidia discrete graphics card, either a 1060/1070 GTX
depending on your configuration. Because of the way this card interacts with the
kernel ACPI code there are times when it can hang up the system if you run `lspci`
or some sort of hardware scanning tool like `neofetch`.

Let's turn it off during install to avoid running into any problems :

{% highlight console %}
echo 1 > /sys/bus/pci/devices/0000:01:00.0/remove
{% endhighlight %}

Don't worry. We'll get it working later on.

## Initial install

Do a regular [UEFI install][uefi-install]. You have you make sure that your EFI
partition is `/boot/efi` instead of `/boot` since this is required for
System76's firmware update utilities to work.

My setup looks like this:

{% highlight console %}
Device            Start       End   Sectors   Size Type
/dev/nvme0n1p1     2048    411647    409600   200M EFI System
/dev/nvme0n1p2   411648  17188863  16777216     8G Linux swap
/dev/nvme0n1p3 17188864 976773134 959584271 457.6G Linux filesystem

Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  197M  122K  197M   1% /boot/efi
/dev/nvme0n1p3  450G   19G  408G   5% /
{% endhighlight %}

## Boot loader

System76's utilities require you to pass `ec_sys.write_support=1` as an argument
for the kernel. You need to configure your bootloader for that. In my case I use
grub so here is my `/etc/defaults/grub` changes to pass this on and a few other
(like the resume swap partition for hibernation).

{% highlight conf %}
...
GRUB_TIMEOUT = 0
GRUB_CMDLINE_LINUX_DEFAULT="quiet nowatchdog ec_sys.write_support=1 resume=/dev/nvme0n1p2"
...
{% endhighlight %}

Install it:

{% highlight console %}
grub-install --target=x86_64-efi --efi-directory=/boot/efi
grub-mkconfig -o /boot/grub/grub.cfg
{% endhighlight %}

# System76 utilities (post-setup)

System76 offers some utilities for Ubuntu and Pop! OS to enable better support
for their hardware and firmware upgrades. The [AUR][aur] has all of these packages
too. _Disclaimer: I maintain a couple of them._

## system76-dkms (aur)

This is system76 kernel module that gives support to the keyboard backlight,
nvidia sound (hdmi) and a few other things. If you have a System76 machine you
definitely want this:

{% highlight console %}
git clone https://aur.archlinux.org/system76-dkms.git
cd system76-dkms
makepkg -sci
sudo modprobe system76
{% endhighlight %}

## system76-firmware-daemon (aur)

This package contains a daemon that is used by `system76-driver` to schedule
firmware updates.

{% highlight console %}
git clone https://aur.archlinux.org/system76-firmware-daemon.git
cd system76-firmware-daemon
makepkg -sci
sudo systemctl enable --now system76-firmware-daemon.service
{% endhighlight %}

## system76-driver (aur)

This package contains the interface to the firmware daemon and a few other
utilities. You don't really need most of them in Archlinux. They are mostly
support for older hardware.

{% highlight console %}
git clone https://aur.archlinux.org/system76-driver.git
cd system76-driver
makepkg -sci
sudo system76-firmware # So you can tell if you have new firmware
{% endhighlight %}

This is how the firmware UI tool looks like:

<img class="center" src="/assets/firmware.png" />

## system76-power (aur)

This is the utility you want to set up some pre-configured power management
profiles for your machine and to control the Nvidia discrete graphics card.

{% highlight console %}
git clone https://aur.archlinux.org/system76-power.git
cd system76-power
makepkg -sci
sudo systemctl enable --now system76-power.service
{% endhighlight %}

# Managing the discrete graphics card

System76 Pop! OS setup is designed to make you switch between Intel and Nvidia,
reboot and then use that card solely during that session. I don't find that
particularly good since this is a laptop and I want to maximize battery life as
much as I can. I want to run Intel's card and use the discrete card only on
demand.

In order to accomplish this, I use [bumblebee][bumblebee] and
[primus][primus]. I don't use [bbswitch][bbswitch] which is used to control the
discrete graphics card power since it is unstable in this machine (at this point
in time) and we can use system76-power to turn the card on and off more
reliably.

## Setup

{% highlight console %}
sudo pacman -S bumblebee primus
sudo gpasswd -a <username> bumblebee    # need to be part of this group
sudo systemctl enable --now bumblebeed.service
sudo pacman -S nvidia nvidia-settings   # You can use nouveau also.
sudo system76-power graphics intel
{% endhighlight %}

And reboot.

## Turn card on

If you want to use the discrete card:

{% highlight console %}
sudo system76-power graphics power on
sudo modprobe nvidia-drm nvidia-modeset nvidia
primusrun glxinfo | grep -i renderer
OpenGL renderer string: GeForce GTX 1060/PCIe/SSE
{% endhighlight %}

## Turn card off

Run the following:

{% highlight console %}
sudo system76-power graphics power off
{% endhighlight %}

You will probably get the following error message:

{% highlight console %}
system76-power: "device in use"
{% endhighlight %}

You have to unload the graphics drivers first:

{% highlight console %}
sudo rmmod nvidia-drm nvidia-modeset nvidia
sudo system76-power graphics power off
{% endhighlight %}

You can check that it worked with:

{% highlight console %}
lspci | grep -i vga
00:02.0 VGA compatible controller: Intel Corporation Device 3e9b
{% endhighlight %}

## Aliases

In order to make this easier I use the following aliases:

{% highlight console %}
alias nvidia-settings="optirun -b none nvidia-settings -c :8 "
alias nvidia-on="sudo system76-power graphics power on ; sudo modprobe nvidia-drm nvidia-modeset nvidia"
alias nvidia-off="sudo rmmod nvidia-drm nvidia-modeset nvidia ; sudo system76-power graphics power off"
{% endhighlight %}

# Not tested/working

- There seems to be a fingerprint reader in this machine but the default kernel
  doesn't seem to pick it up and I haven't made much of an effort to try to make
  it work. I didn't test it on Pop! OS. If you know which driver it needs please
  let me know.
- Multiple monitors, I don't have an external monitor so I haven't tested
  this. Depending on which graphics card the HDMI/Display ports are wired the
  setup would be different. I'll update the post once I try this.

The webcam, sd card reader, sound, wifi, ethernet work out of the box.

I really hope this guide works for you as it took me a while to figure all this
out (and even had to bring a bring a couple of System76's utilities to our own
AUR).

[popos]:            https://system76.com/pop
[uefi-install]:     https://wiki.archlinux.org/index.php/Installation_guide
[aur]:              https://aur.archlinux.org
[bumblebee]:        https://wiki.archlinux.org/index.php/bumblebee
[primus]:           https://wiki.archlinux.org/index.php/bumblebee#Primusrun
[bbswitch]:         https://wiki.archlinux.org/index.php/bumblebee#Default_power_state_of_NVIDIA_card_using_bbswitch
