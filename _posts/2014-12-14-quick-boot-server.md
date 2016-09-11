---
layout: post
title: "Quick boot server"
cover:
  image: /media/2014-12-14-quick-boot-server/cover.jpg
quote: Ever wanted to boot Linux somewhere, but you don't have a pen at hand? Well, now you can.
comments: 2014-12-14-quick-boot-server
---

Time for a little tutorial!

When fixing computers I'd often like to boot a live-CD so I can freely inspect and play around,
instead of working with the shitty Windows they usually have.

But what if there isn't a pen (or writable CD) at hand? Today we're gonna setup a quick
**boot server** so that computers can boot from us! Wow.

Advantages of this method:

 - No need to find a pen with enough space, you just need an Ethernet cable.
 - Quick to use. Set it up, connect the cable, boot via network and you're done.
 - Also acts as a router, so when the computer boots it can access the Internet
   through your connection.
 - Portable. Start and stop serving whenever you want.

Disadvantages:

 - Can be slower than a pen sometimes.
 - Getting the BIOS to cooperate and boot from network is, in my experience, tricky.


## Preparing

~~~ bash
sudo apt-get install dnsmasq syslinux-common nfs-kernel-server
~~~

Now we have everything we need. But before continuing, edit `/etc/default/dnsmasq`
and set `ENABLED` to `0` to prevent dnsmasq to start automatically. Then, stop it:

~~~ bash
sudo service dnsmasq stop
~~~

And while we're at, you should follow [these steps](http://stackoverflow.com/questions/5321380/disable-network-manager-for-a-particular-interface#6810854)
to make sure NetworkManager won't mess with the wired interface!

## Basic setup

~~~ bash
mkdir myserver && cd myserver
~~~

Okay, the first thing is to setup a simple DHCP server with dnsmasq.
Create `dnsmasq.conf` with this content:

    listen-address=192.168.99.1
    bind-interfaces
    dhcp-range=192.168.99.100,192.168.99.199,12h
    dhcp-authoritative

Let's now start dnsmasq to test the config works:

~~~ bash
sudo ifconfig eth0 up 192.168.99.1/24
sudo dnsmasq -C dnsmasq.conf
~~~

Now connect an Ethernet cable to the other computer and verify it gets an IP as expected.
**Bonus!** Dnsmasq already forwards DNS requests and sets the gateway so you just need to:

~~~ bash
sudo iptables -t nat -A POSTROUTING -s 192.168.99.1/24 -j MASQUERADE
~~~

and now the computer can also access the Internet through the NAT we just created!


## Adding TFTP

Now on to the interesting thing: network booting.

TFTP is frequently used by bootloaders to download the image when booting via network.
Let's create the directory we'll serve over TFTP:

~~~ bash
mkdir tftp
~~~

Now take the ISO of your favorite distribution (I recommend LUbuntu) and *extract* its contents
at the directory we just created:

~~~ bash
mkdir /tmp/iso
sudo mount -t iso9660 /path/to/lubuntu.iso /tmp/iso
cp -r /tmp/iso/* tftp
sudo umount /tmp/iso
~~~

Now let's put in `pxelinux.0`, which will be the invoked by the bootloader:

~~~ bash
cp /usr/lib/syslinux/pxelinux.0 tftp
~~~

And create the configuration files to tell pxelinux where to find the Linux kernel, and the options:

~~~ bash
mkdir tftp/pxelinux.cfg
echo "DEFAULT casper/vmlinuz.efi initrd=casper/initrd.lz boot=casper root=/dev/nfs netboot=nfs nfsroot=192.168.99.1:/absolute/path/to/tftp quiet splash --" > tftp/pxelinux.cfg/default
~~~

Make sure `casper/vmlinux.efi` exists within `tftp` and if not, change it as appropiate.
Same with `casper/initrd.lz`. Also change `/absolute/path/to/tftp` to, well, you know.

Make sure the paths have no spaces or strange characters in them.


## Start it!

Let's export the `tftp` folder through NFS (so that the booted Linux can access it):

~~~ bash
sudo tee -a /etc/exports <<< "/absolute/path/to/tftp   192.168.99.0/24(ro,no_subtree_check,fsid=DDD)"
sudo service nfs-kernel-server restart
~~~

(where DDD is a random number)

And finally, let's tell dnsmasq to serve TFTP! Add this to `dnsmasq.conf`:

    enable-tftp
    dhcp-boot=pxelinux.0
    tftp-root=/absolute/path/to/tftp

We're done! Restart dnsmasq:

~~~ bash
sudo killall dnsmasq
sudo dnsmasq -C dnsmasq.conf
~~~

Now it's time to reboot the other computer, make it boot via network, and pray it'll work.
Note: it takes some time to boot. Be patient.


## Automating everything

Keep in mind this is designed to be temporal: we don't want to turn our laptop into a
permanent boot server, instead we start and stop the server when needed.

Starting and stopping the server is three or two commands; we can create two scripts to
handle that for us.

Create `startserver` next to `dnsmasq.conf`:

{% highlight bash %}
#!/bin/bash
# Start a DHCP, DNS and TFTP boot server, and a NAT.
cd $(dirname $0)

sudo ifconfig eth0 up 192.168.99.1/24
sudo iptables -t nat -A POSTROUTING -s 192.168.99.0/24 -j MASQUERADE
sudo dnsmasq -C dnsmasq.conf
{% endhighlight %}

Create `stopserver` too:

{% highlight bash %}
#!/bin/bash
# Start a DHCP, DNS and TFTP boot server, and a NAT.
cd $(dirname $0)

sudo killall dnsmasq
sudo iptables -t nat -F
{% endhighlight %}

Make them executable:

~~~ bash
chmod +x startserver stopserver
~~~

And that's it! Next time you need to boot Linux on a computer (or need the NAT)
just do `./startserver` and connect the cable, and `./stopserver` when done.


## Conclusion

I find myself using scripts like these much of the time I need to fix problems on
foreign computers, especially these two, and I hope they'll be useful to someone else.

Keep in mind we're not setting any firewalling, i.e. hosts from the "outside" can access
our NAT-ed hosts. Adding firewalling is left as an excercise to the reader.
