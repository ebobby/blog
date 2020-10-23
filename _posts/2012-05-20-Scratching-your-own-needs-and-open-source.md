---
layout: post
title: Scratching your own needs and open source
date:  2012-05-20
tags: [tech]
---

<img style="float: left" src="../assets/beanie.png" />

I’ve been thinking recently about how hard it feels to contribute to an
open source project and by that I don’t mean that open source makes it
hard for you, *au contraire*, some of them try to make it as easy for
you to do it as possible, sites like [github](http://www.github.com),
[codeplex](http://www.codeplex.com), [google project
hosting](http://code.google.com/hosting/), etc., give you and developers
the tools to build an open source project and try to foster a community
around it.

I think one of the main motivations people have to contribute for
creating and contributing to open source projects is having a need. You
need something and you simply do not find something that solves your
need in the market (or you can’t pay for it) and create an open source
project, or maybe there’s a project somewhere that almost solves it and
it just needs a little push so you contribute. You need to use it to be
able to be an effective contributor, and most of the time, you use it
because you need it.

You can find a lot of examples of this but the canonical example is
[Linus Torvalds](http://en.wikipedia.org/wiki/Linus_Torvalds) starting
[Linux](http://en.wikipedia.org/wiki/Linux) because he wanted to run
Unix on his 386 and his only option at the time was
[MINIX](http://www.minix3.org/) and he was frustrated by its licensing.

Nowadays of course, most people don’t have a need to build an OS from
scratch (do [they](http://www.loper-os.org/?\)), so they build other
fancy things like [Redis](http://redis.io) or [Ruby On
Rails](http://rubyonrails.org/) which again, were started to scratch a
need and people have been adopting in en masse because they had that
pain too.

Funny thing is, we look at these examples and think that a successful
contribution or a successful project has to be a complex piece of
software that solves the needs of millions, or contributing to a project
involves being a genius, nothing far from the truth, lots of open source
projects thrive with the contributions of every day developers like you
and me caring enough to research a bit and be interested in helping.

When I was in college, a few years ago, I used
[FreeBSD](http://www.freebsd.org) on my personal computer which was a
bit of a problem because the school machines and my professors used
Windows machines. So when I was developing something (mostly in C back
then) it had to compile and be able to run in Windows. For the most part
this isn’t very hard to do if you stick to [ANSI
C](http://en.wikipedia.org/wiki/ANSI_C) which suffices for most of your
average school projects. That is, until you try to do something fancy…

As part of my AI class, we analyzed several [combinatorial
optimization](http://en.wikipedia.org/wiki/Combinatorial_optimization)
algorithms and tried to solve different problems with them, one of the
problems of choice was the [8 queen
puzzle](http://en.wikipedia.org/wiki/Eight_queens_puzzle) which
basically consists on coming up with arrangement of 8 queens in a
regular chess board so that no two queens can attack each other.

At the end of the semester I decided to present as a final project a
version of all the algorithms we saw trying to solve this problem but
instead of having only print outs of numbers or strings as a solution,
to actually have a 3D chess board and the queens moving from place to
place as the algorithms worked on the problem so it would be more visual
and more [interesting](http://franciscosoto.net/jsqueens) to look at. I
coded this using C and [OpenGL](http://www.opengl.org/) under FreeBSD
and since I needed to compile it under Windows too I used
[GLUT](http://www.opengl.org/resources/libraries/glut/) to handle my
windowing and input needs.

The way I went was using the [Mesa 3D](http://www.mesa3d.org/) library
which at the time was at its 5.\* version, but in FreeBSD’s
[ports](http://www.freebsd.org/ports/) the available version was 3.\*
(called Mesa3) which for some reason would not work in my machine, the
window opened and the program ran and I got output on my terminal but
the display would be always black and I struggled with this for a while
but it really pissed me off that with a hand compiled current (at the
time\!) Mesa 5.\* version it would work flawlessly.
The reason it pissed me off is because I have never liked manually
installing things in my \*nix systems, if you want to remove them you
have to look for the files, read the Makefile, etc. I always try to use
the package manager or I feel my system as “unclean”.

So, in order to be able to use Mesa5 and still feel somewhat clean was
to make it a package, so I did, I copied the Mesa3 port directory to a
new Mesa5 directory and changed the Makefile (and I do not remember if I
had to change the patch files a little bit too) to work with Mesa 5.\*.
I got my project running, able to compile correctly under FreeBSD and
Windows and got an A+ on my AI class.

I also sent the files to FreeBSD (I do not recall what medium I used), I
thought they were going to ignore them and I thought they did because I
never actually got an answer.

Here’s the kicker, years later, as I was googling around my old email
address for reasons I do not remember, I found out a series of emails
about my commit\!, they actually
[received](http://lists.freebsd.org/pipermail/cvs-ports/2003-October/014219.html)
it and considering making me the
[maintainer](http://lists.freebsd.org/pipermail/cvs-ports/2003-October/014238.html)
of a newly created **mesagl** port (mine was called **Mesa5** I guess
mesagl was created by another guy with my same problem) until another
guy came with another
[update](http://lists.freebsd.org/pipermail/cvs-ports/2003-October/014242.html)
that was better. I never found out when it was actually happening
because I wasn’t in the mailing list (which is one of the first things
they ask you to do if you are going to commit stuff, my mistake).

So that’s the way I *almost* became a FreeBSD port maintainer. Maybe if
I go back to FreeBSD at some point (using archlinux now) Ill try to get
to maintain a port.
