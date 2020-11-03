---
layout: post
title: Improving Linux battery life and thermal efficiency
date:  2018-09-01
tags: [linux]
---

I recently switched from MacOS to Linux for my personal/work machine. As much as
I like my new laptop I really missed all the battery life that my previous
Macbook Pro was capable of.

I decided to do something about it and improve battery life of my setup as much
as I could. This post is part me saving it for future reference and part sharing
it with you, the curious reader.

## Setup

I have a [System76's Oryx Pro 4 (2018)][oryxpro4] with the following characteristics:

- CPU: Intel i7-8750H, 2.2 up to 4.1 GHz, 6 cores
- Display: 15.6" 1920x1080 144hz
- Discrete GPU: NVIDIA GeForce GTX 1060
- Memory: 32 GB dual-channel DDR4 @ 2400 MHz
- Storage: 500G Samsung NVME 970 EVO, 1TB Samsung SSD 860 EVO
- Communications: Gigabit Ethernet, Intel Wireless-AC WiFi, Bluetooth
- Camera: 1080p HD Webcam
- Battery: Embedded 4 Cells Polymer battery pack – 55Wh

And I run the following software (at the time of this writing):

{% highlight console %}
-------------
OS: Arch Linux x86_64
Host: Oryx Pro oryp4
Kernel: 4.18.5-arch1-1-ARCH
Packages: 900 (pacman)
Shell: zsh 5.5.1
WM: i3
------------------
{% endhighlight %}

## Pop OS!

[System76][system76] has their own Linux distribution [Pop! OS][popos]. By
default uses Gnome and comes preloaded with some good stuff and uses an older
version of the kernel. It must be the LTS kernel since they have to support it
and it would be crazy to support every version out there.

When I tested this machine with Pop OS! I got a discharge rate of about ~11-10W
idle, just powertop running. I do not remember if I had the keyboard backlight
on and what the screen backlight level was so it may not be a fair comparison
but it's a good baseline.

## Archlinux

I use [Archlinux][arch] and i3-gaps which are way lighter than Ubuntu (Pop! OS base) and
Gnome. With my current setup I get ~8.5 W discharge rate while idling, just
running powertop.

This is before I run anything related to power savings so it comes to show how
much Linux has improved.

## Power savings configuration

### Kernel command line

I boot the kernel adding the following to the command line:

{% highlight console %}
nowatchdog
{% endhighlight %}

The interesting bit for our current discussion is the nowatchdog paramater which
[turns off][kernel-cli] lockup detectors in the kernel which saves a tiny fraction of power
and performance.

### Kernel module configuration

I have the following kernel module configuration file:

{% highlight conf %}
# /etc/modprobe.d/power-management.conf
# Wifi power saving
options iwlwifi power_save=1 d0i3_disable=0 uapsd_disable=0
options iwldvm force_cam=0

# USB autosuspend
options usbcore autosuspend=5

# Sound card power save
options snd_hda_intel power_save=1

# Intel GPU
options i915 enable_dc=2 enable_fbc=1
{% endhighlight %}

These are here for reference only and to show the concept of doing some research
on what drivers your machine is using and what do they need to have increased
power savings. You don't want to copy these as-is because it may turn your
machine unusable.

### TLP – Linux Advanced Power Management

From their project page:

> TLP brings you the benefits of advanced power management for Linux without the
> need to understand every technical detail. TLP comes with a default
> configuration already optimized for battery life, so you may just install and
> forget it. Nevertheless TLP is highly customizable to fulfil your specific
> requirements.

[TLP][tlp] is a daemon that turns on power saving flags in hardware, configures the
CPU, configures kernel parameters, etc, that are good for power savings. As the
project says, the defaults are pretty good but you can always tweak it a little
bit more.

My own configuration:

{% highlight conf %}
TLP_ENABLE=1
TLP_DEFAULT_MODE=AC
TLP_PERSISTENT_DEFAULT=0
DISK_IDLE_SECS_ON_AC=0
DISK_IDLE_SECS_ON_BAT=5
MAX_LOST_WORK_SECS_ON_AC=15
MAX_LOST_WORK_SECS_ON_BAT=60
CPU_SCALING_GOVERNOR_ON_AC=powersave
CPU_SCALING_GOVERNOR_ON_BAT=powersave
CPU_HWP_ON_AC=balance_performance
CPU_HWP_ON_BAT=balance_power
SCHED_POWERSAVE_ON_AC=0
SCHED_POWERSAVE_ON_BAT=1
NMI_WATCHDOG=0
ENERGY_PERF_POLICY_ON_AC=performance
ENERGY_PERF_POLICY_ON_BAT=power
DISK_IOSCHED="keep"
SATA_LINKPWR_ON_AC="max_performance"
SATA_LINKPWR_ON_BAT="min_power"
AHCI_RUNTIME_PM_TIMEOUT=15
PCIE_ASPM_ON_AC=performance
PCIE_ASPM_ON_BAT=powersave
RADEON_POWER_PROFILE_ON_AC=high
RADEON_POWER_PROFILE_ON_BAT=low
RADEON_DPM_STATE_ON_AC=performance
RADEON_DPM_STATE_ON_BAT=battery
RADEON_DPM_PERF_LEVEL_ON_AC=auto
RADEON_DPM_PERF_LEVEL_ON_BAT=auto
WIFI_PWR_ON_AC=off
WIFI_PWR_ON_BAT=on
WOL_DISABLE=Y
SOUND_POWER_SAVE_ON_AC=0
SOUND_POWER_SAVE_ON_BAT=1
SOUND_POWER_SAVE_CONTROLLER=Y
BAY_POWEROFF_ON_AC=0
BAY_POWEROFF_ON_BAT=0
BAY_DEVICE="sr0"
RUNTIME_PM_ON_AC=on
RUNTIME_PM_ON_BAT=auto
USB_AUTOSUSPEND=1
USB_BLACKLIST_BTUSB=0
USB_BLACKLIST_PHONE=0
USB_BLACKLIST_PRINTER=1
USB_BLACKLIST_WWAN=1
RESTORE_DEVICE_STATE_ON_STARTUP=0
{% endhighlight %}

There is some fine tweaking possible for newer CPUs with TLP to manage cpu power
hints, pstates, turbo settings, etc. I don't use them since I run System76's
system76-power(github) to handle the CPU profiles because it is easier.

#### Archlinux

{% highlight console %}
sudo pacman -S tlp
sudo systemctl enable --now tlp.service
sudo tlp-stat
{% endhighlight %}

### fancontrol

`fancontrol` is a daemon, that as its name says, controls fans in your
computer. That is, if the computer maker decided to give you that power. In
Linux, such control is done through the PWM (Pulse Width Modulator) interface.

PWM allows you to control the power that goes into an electrical line (I am
being intentionally simplistic here). If you have fan PWM lines available you
can control how much power goes into them, and in turn, how fast they spin.

`fancontrol` is part of the `lm_sensors` package in archlinux and ubuntu.

`fancontrol` is a tool to control a fan speed based on some temperature
thresholds. This is useful because usually the auto control in the fans is
overly aggressive, turning fans on very rapidly when the CPU activity increases
a tiny bit. And, the faster a fan spins, the more power it consumes.

You have to take several things into account here, the temperature limits of the
hardware you want to control the fan of, the ambient temperature of where you
live. Say, if the CPU wants to be at 30C by default but your ambient temperature
is 40C those fans are going to spin forever. You want to be careful with
this. You most likely don't want to fry your electronics.

If you install the `lm_sensors` package and run the sensors utility you can see
which hardware temperature monitors you have available and what are their
temperature limits. Plus, if you have fans around.

Mine looks like this:

{% highlight console %}
$ sensors
coretemp-isa-0000
Adapter: ISA adapter
Package id 0:  +39.0°C  (high = +100.0°C, crit = +100.0°C)
Core 0:        +37.0°C  (high = +100.0°C, crit = +100.0°C)
Core 1:        +38.0°C  (high = +100.0°C, crit = +100.0°C)
Core 2:        +37.0°C  (high = +100.0°C, crit = +100.0°C)
Core 3:        +38.0°C  (high = +100.0°C, crit = +100.0°C)
Core 4:        +36.0°C  (high = +100.0°C, crit = +100.0°C)
Core 5:        +37.0°C  (high = +100.0°C, crit = +100.0°C)

system76-isa-0000
Adapter: ISA adapter
CPU fan:         2533 RPM
GPU fan:         4083 RPM
CPU temperature:  +38.0°C
GPU temperature:   +0.0°C
{% endhighlight %}

Now, that is one I care about, the system76-isa-0000 has a fan for the CPU, and
turns out it is very aggressive and since I live in a very warm place to begin
with, I want to tone it down so I can save a bit of juice.

Run the `pwmconfig` utility which will create an `/etc/fancontrol` file which
looks like this:

{% highlight conf %}
# Configuration file generated by pwmconfig, changes will be lost
INTERVAL=10
DEVPATH=hwmon1=devices/platform/system76
DEVNAME=hwmon1=system76
FCTEMPS= hwmon1/pwm1=hwmon1/temp1_input
FCFANS= hwmon1/pwm1=hwmon1/fan1_input
MINTEMP= hwmon1/pwm1=40
MAXTEMP= hwmon1/pwm1=75
MINSTART= hwmon1/pwm1=14
MINSTOP= hwmon1/pwm1=12
{% endhighlight %}

(Don't copy this. This is very hardware dependant, run pwmconfig)

Enable the `fancontrol.service` and voila.

#### Archlinux

{% highlight console %}
sudo pacman -S lm_sensors
sudo pwmconfig
sudo systemctl enable --now fancontrol.service
{% endhighlight %}

Fans are now under control.

### thermald

[Linux thermal daemon][thermald] (thermald) monitors and controls temperature in
laptops, tablets PC with the latest Intel sandy bridge and latest Intel CPU
releases. Once the system temperature reaches a certain threshold, the Linux
daemon activates various cooling methods to try to cool the system.

Now, the difference between the fans and thermald is that thermald uses passive
cooling methods built into these CPUs.

`thermald` is a last barrier. When the CPU is under heavy load and not even the
fan can keep it cool thermald uses several passive cooling features built into
newer cpus (Intel's, not sure about AMD) to minimize power and temperature.

This is my current configuration:

{% highlight xml %}
<!-- /etc/thermald/thermal-conf.xml -->
<?xml version="1.0"?>
<ThermalConfiguration>
  <Platform>
    <Name>Override CPU Passive</Name>
    <ProductName>*</ProductName>
    <Preference>QUIET</Preference>
    <ThermalZones>
      <ThermalZone>
        <Type>cpu</Type>
        <TripPoints>
          <TripPoint>
            <Temperature>82000</Temperature>
            <type>passive</type>
            <ControlType>SEQUENTIAL</ControlType>
          </TripPoint>
        </TripPoints>
      </ThermalZone>
    </ThermalZones>
  </Platform>
</ThermalConfiguration>
{% endhighlight %}

Which pretty much means that if the CPU goes higher than 82C it should use all
matters of passive cooling at its disposal to cool it down. I've tested this and
it's amazing how even using all the cores heavily it stays at 82C.

After you configure it you just enable the daemon and that's it. It will do it's
thing when it must.

#### Archlinux

{% highlight console %}
sudo pacman -S thermald
sudo systemctl enable --now thermald.service
{% endhighlight %}

### Undervolting

Note: This is experimental.

Undervolting is similar to overclocking, except instead of increasing speed, you
are trying to keep your current speed by using less voltage.

This works because due to tiny differences in manufacturing processes CPU
voltage and power specifications are not the same between units. They have
ranges and tolerances. This is what lets you overclock and undervolt.

When a CPU maker builds a CPU they make sure to feed it voltage which can
guarantee they meet the specified clock speed. But it is very likely any given
CPU can work with less voltage and still work within the specified speeds and
stability.

What this accomplishes for us is using less power to feed the CPU and that saves
juice. Less voltage also mean less heat, which in turn means less fan noise and
spinning, which also saves juice.

I use [intel-undervolt][intel-undervolt] to set the voltages, but there are
other scripts and tools available.

You have to find the specific voltages and tolerances that your CPU can work
with, you do this by decreasing the voltage just a tiny bit, do some heavy
stress testing for a good chunk of time. If it works, you decrease it a bit
more, and you continue doing it till you find a set of voltages that work for
your machine.

This is how my (current) `/etc/intel-undervolt.conf` looks like:

{% highlight conf %}
# CPU Undervolting
# Usage: apply ${index} ${display_name} ${undervolt_value}
# Example: apply 2 'CPU Cache' -25.84

apply 0 'CPU' -170
#apply 1 'GPU' 0
apply 2 'CPU Cache' -170
#apply 3 'System Agent' 0
#apply 4 'Analog I/O' 0

# TDP Alteration
# Usage: tdp ${short_term} ${long_term}
# Usage: tdp ${short_term}/${time_window} ${long_term}/${time_window}
# Example: tdp 45/0.002 35/28

# Critical Temperature Offset Alteration
# Usage: tjoffset ${temperature_offset}
# Example: tjoffset -20

# Daemon Update Inverval
# Usage: interval ${interval_in_milliseconds}

interval 5000
{% endhighlight %}

The reason the GPU is commented out is because apparently it is already
undervolted by quite a bit by default. I recommend you use `sudo intel-undervolt
read` to check your default values. You may find something you don't want to
change.

#### Archlinux

{% highlight console %}
git clone https://aur.archlinux.org/intel-undervolt.git ; cd intel-undervolt
makepkg -sci
sudo systemctl enable --now intel-undervolt.service
{% endhighlight %}

## Results

With all of this configuration, my idle power usage doesn't really change very
much. It starts showing when you actually put some load and start using the
system. My overall battery life went from about ~4 hours to about ~5.5
hours. Which is pretty good. It still sucks compared with the MBP but this
machine is a power house. I didn't really expect to get 10 hours of battery with
it.

Thermals improved quite a bit too, and that's pretty important to me because I
live in a very hot place.

That is it. If you have any suggestions how to improve this guide, or how to
increase battery life please do share!

[oryxpro4]:   https://system76.com/laptops/oryx
[system76]:   https://system76.com/
[popos]:      https://pop.system76.com/
[arch]:       https://archlinux.org/
[kernel-cli]: https://01.org/linuxgraphics/gfx-docs/drm/admin-guide/kernel-parameters.html
[tlp]:        https://linrunner.de/en/tlp/tlp.html
[thermald]:   https://01.org/linux-thermal-daemon/documentation/introduction-thermal-daemon
[intel-undervolt]: https://github.com/mihic/linux-intel-undervolt
