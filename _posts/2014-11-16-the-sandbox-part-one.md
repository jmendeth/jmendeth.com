---
layout: post
title: "The Sandbox: part I"
cover:
  image: /media/2014-11-16-the-sandbox-part-one/cover.jpg
  source: http://www.tripwiremagazine.com/2012/12/sand-texture.html
dark: yes
quote: "If you're like me, working in an environment where you don't have admin rights was probably frustrating, but no more!"
comments: 2014-11-16-the-sandbox-part-one
---

Now's my first year at university.

We're given credentials so we can login into the computers around the campus, which run Ubuntu 12.04 LTS and ---surprise!--- have *not* been updated in a whole lot of time.

After using them for some time, the unevitable happened: It lacked my essential set of programs (Chromium or Node.JS or KmPlot for example) and I really
missed my laptop where I have admin rights and I can just `apt-get install` what I need. These computers boot via PXE, so I thought about bringing a Pi
to make them boot whatever I wanted, but it'd be a questionalbly legal approach which could get me into trouble.

I don't want that. (Not yet.)

So I'm making this series of posts explaining the steps I did to get a bit of **freedom** within my restricted user, in the hope they'll be useful for someone else.  
**Spoiler:** I could finally run my own, sandboxed Debian install inside my user, with great results! This is explained in the next parts.


## First steps

Let's start with the basics. You don't have root access so you can't install binaries at `/usr/bin`; you need a directory that is always in `PATH`, where you can drop programs and run them as if they were system-wide commands.

    mkdir -p ~/.local/bin

In case you didn't know, there's a convention that `.local` is like the `/usr/local` prefix but for individual users (`/usr/local/share` is equivalent to `~/.local/share`, same with `bin`, `lib`, ...). We'll be following this convention.

Now, edit `~/.profile` and append the code:

    export PATH=$PATH:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:$HOME/.local/bin

This will add our directory to `PATH` as well as the `sbin` directories where commands like `ifconfig` are stored.

> **Important:** add it to `.profile`, not `.bashrc`. The earlier is session-wide, the latter only applies to your interactive Bash shells.

And while we're at it, what about having our own directory for *libraries* as well? Maybe our programs depend on libraries which are not installed. So let's do it:

    mkdir -p ~/.local/lib

Edit `~/.profile` and append:

    export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/.local/lib

Now the linker will also load libraries from our local directory.

**Great!** Logout and login again to make changes effective.


## Installing packages

Now to something more interesting: installing `.deb` packages. Let's suppose we want to install the `tree` utility.

`apt-get install tree` won't work of course, but we can instead do `apt-get download tree`, which will just fetch the `.deb` package from the repos.

Then let's extract the contents of the package in some directory:

~~~ bash
mkdir test
dpkg-deb -x tree_xxxxx_xxxx.deb test
# repeat for every downloaded package
~~~

It'll create a bunch of files in `test/usr/share` (documentation and manual pages), and the binary that we want at `test/usr/bin/tree`.
So we can take that binary and drop it in `.local/bin` and we should be able to run `tree` without problems now.

#### What if I can't?

That's probably because, in order for `tree` to run, you must install some dependencies too.
Try to simulate how `apt-get` would install that package:

    apt-get -s install tree

Note the dependencies `apt-get` installs, and repeat the procedure above for each dependency.
Remember to copy any library you see to `.local/lib`!


## Compiling programs

Now, what if we want to install a program that isn't on the repos, or you want to install a newer version?
I'd recommend you do something very similar to what we did above.

Compile it as you'd do on your computer (i.e. `./configure && make` or whatever it needs) but don't do the final `make install` since we don't have root access.

Instead, create an empty directory and tell `make` to install the files there:

    mkdir test
    DESTDIR=test make install

Examine the files it installed, and copy what you need to the local directories as we did in the previous section.

Even if the program doesn't use `make` as buildsystem, you should still try to invoke it with the `DESTDIR` variable set, as we did before.
It's a pretty standard convention. If it still doesn't work, just copy the files yourself and see if it works.


## Conclusion

We're still limited, but we can at least install our own programs now.  
But there must be a better way...

This comes at part II!
