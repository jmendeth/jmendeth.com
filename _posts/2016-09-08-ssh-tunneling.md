---
layout: post
title: SSH tunnels done right
cover:
  image: /media/2016-09-08-ssh-tunneling/cover.jpg
  source: https://unsplash.com/photos/xNdPWGJ6UCQ
quote: There's a better way to use SSH tunnels than the standard port redirection we all know and love.
comments: 2016-09-08-ssh-tunneling
theme_color: 6B5951
---

In case you've never used them, SSH tunnels are freaking useful.
They turn SSH servers into temporary *proxies* so you can connect to hosts
you can't reach directly. Keep reading.

The traditional way to create tunnels is by using the `-L` option of `ssh`.
Let's say you want to manage your home's AP. You usually do this by browsing
to `https://192.168.1.4`.

If you're outside home, but are able to SSH into your home's router, you'd type
something like:

~~~ bash
ssh <router public IP> -L 8080:192.168.1.4:443
~~~

When it logs in, you'd browse to `https://localhost:8080` and see your AP's
administration portal.

## The problem

All seems good, but your browser has no way of knowing that
`localhost:8080` is actually going to `https://192.168.1.4`, which means that:

 - Static assets will need to be redownloaded and cached again.
 - If the website uses absolute URLs to load its resources, it won't work.
 - Each time the website redirects you to `https://192.168.1.4/something`, you'll
   have to correct the URL manually.
 - You'll have to login again, because the session cookies for `192.168.1.4`
   will not be in effect.
 - Any previously saved passwords won't apply.
 - URL autocomplete won't be useful either.
 - The AP will not receive `Host: 192.168.1.4` as usual, but a confusing
   `Host: localhost:8080`, which can give problems with virtual hosts.

Fortunately, there's a way out of all this.


## The solution

Instead of using `-L`, use `-D <port>`. This instructs `ssh` to create a SOCKS
proxy that'll tunnel every incoming connection through your SSH session.

So, staying with our AP example, you'd run something like:

~~~ bash
ssh <router public IP> -D 49000
~~~

And set your browser to use the SOCKS proxy at `localhost` port 49000.
Then, just browse to `https://192.168.1.4` as you'd do at home!

Think about it. No more bashing against the wall until you realize you typed
`-L 8080:192.168.1.4:80` instead of `-L 8080:192.168.1.4:443`. No more seing
`ERR_CONNECTION_CLOSED` until you type in the «https://» before «localhost».
URL completion works. Redirects work. Absolute URLs work. You can browse just
as if you were at home.

It's also more natural. You can first ssh into the router, *then* think of
what you want to access. If you then want to manage another device,
no need to kill ssh and run it again changing the `-L` option.

There's one more perk: names are resolved from your router too, so if browsing
to `https://ap.lan` would work from home, it'll also work with the SOCKS proxy.

**Isn't this the way you always wanted tunneling to work?**


## Tunneling other things

We can see that using `-D` plays well with browsers. But not all tunnels are
made to access web portals. Another frequent use for SSH tunnels is SSH itself
(i.e. to SSH into an otherwise unreachable host).

I made a helper program that adds proxy support to SSH. Once installed (see
below) you can do:

~~~ bash
ssh <router public IP> -D 49000

# on another terminal
export all_proxy="socks://localhost:49000"
ssh root@ap.lan
~~~

Again, compare with:

~~~ bash
ssh <router public IP> -L 8080:ap.lan:22

# on another terminal
ssh -p 8080 root@localhost
~~~

When you do `ssh root@ap.lan`, ssh won't ask you «Are you sure you want to
continue connecting (yes/no)?» since it knows it's connecting to your AP.
And options in your `.ssh/config` set for your AP will be honored. This is
impossible with `-L`.


## ssh-from

To make it even easier, I made a command called [`ssh-from`](https://gist.github.com/jmendeth/346f2233310d8292efe7595d60aa3659).
You invoke it just like `ssh`, but instead of getting a remote
shell to issue commands in, you get a local shell "tunnelled" through that
host. When you're finished, type `exit` to terminate the proxy.

Inside this "tunnelled" shell, `ssh` and derivatives like `scp`, `sftp`,
`sshfs` all work out of the box:

{% include image.html url="/media/2016-09-08-ssh-tunneling/example-ap-ssh.jpg" description="Using ssh-from to SSH into the AP from outside." %}

Chromium will also work if you launch it with
`chromium-browser --temp-profile --proxy-server=$all_proxy`:

{% include image.html url="/media/2016-09-08-ssh-tunneling/example-ap-web.jpg" description="Using ssh-from to manage the AP through its website." %}

`ssh-from` is just `ssh`, so it'll also work! This makes it really easy to
create double (or triple) SSH tunnels:

{% include image.html url="/media/2016-09-08-ssh-tunneling/example-double-ssh.jpg" description="Connecting to a laptop behind two routers, by nesting ssh-from sessions." %}

Some commands like `curl` will also work out of the box, but for commands
with no proxy support, `ssh-from` integrates with tsocks so just prefix[^1] them
by `tsocks`:

{% include image.html url="/media/2016-09-08-ssh-tunneling/example-telnet.jpg" description="Connecting by telnet to a host through ssh-from." %}


## Installation

Want to have `ssh-from` too? To install, make sure you have a compiler,
python, lsof and tsocks:

~~~ bash
sudo apt-get install build-essential python lsof tsocks
~~~

Then:

~~~ bash
sudo wget https://gist.github.com/jmendeth/346f2233310d8292efe7595d60aa3659/raw/ssh-from.py -O /usr/local/bin/ssh-from && sudo chmod a+rx /usr/local/bin/ssh-from
wget https://gist.github.com/jmendeth/9b3c50226aa82a292a452107b34aca79/raw/ssh-proxy-dialer.c && cc ssh-proxy-dialer.c && sudo install a.out /usr/local/bin/ssh-proxy-dialer && rm ssh-proxy-dialer.c
~~~

Finally, put this at the end of `/etc/ssh/ssh_config`:

~~~ conf
Match exec "ssh-proxy-dialer test"
ProxyCommand ssh-proxy-dialer dial '%h' '%p'
ProxyUseFdpass yes
~~~

Was `ssh-from` useful to you? Any suggestions?
Please let me know in the comments!


<!-- TODO: mac support -->



[^1]: A limitation of tsocks is that name resolution happens locally, not remotely, so you should specify IPs instead of internal names if using it.
