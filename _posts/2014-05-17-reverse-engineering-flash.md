---
layout: post
title: Reverse-engineering Flash
cover:
  image: /media/2014-05-17-reverse-engineering-flash/cover.jpg
quote: A complete introduction to reverse-engineering SWFs on Linux, using open-source tools.
comments: 2014-05-17-reverse-engineering-flash
theme_color: 912303
---

Flash has always intimidated me. Websites usually use it to evade inspection
(together with minified JS) or to make use of specific features (clipboard, memory, ...).

Turns out, in practice Flash helps in reverse-engineering. This is because there
are few Flash obfuscators and people don't think anyone is ever going to look
inside their SWFs, so they don't use them. Sometimes I even find additional
**debug info**, like the complete filename of each source file, line numbers, etc.

Flash is high-level assembly, like Java. You get function names, parameter names,
class names, field names and the assembly is easy to understand once you're
accostumed to it. That, plus the fact it runs in a sandboxed environment (just
like Java applets) makes it really easy to deal with.

There's open-source, high quality software out there that allows for precise
manipulation of SWFs. But before we dive in, let's talk briefly about the SWF.


## Small Web Format

I don't know much about the format, but every SWF consists of a **header**
(indicating, among other things, Flash version and compression) and then
a series of **tags**. A tag can contain other tags, text, controls, multimedia,
vector paths, compiled ActionScript or arbitrary binary content, to name a few.

If you have never programmed in ActionScript, there's an important thing to
note. In Flash, classes "reference" objects on the SWF if the name matches
and they extend the correct class.

For example, if the SWF has a button named `example.Submit` and the ActionScript
declares a class named `Submit` on package `example` that extends
`flash.display.Button`, then adding event listeners on that class will add them
onto the original button, and so on.

Similarly for binary tags, declaring a class named `Payload` that extends
`flash.utils.BinaryArray` allows ActionScript to access the binary content of
a binary tag with the same name, that could be a hidden resource or a compressed
asset.


## ActionScript ByteCode (ABC)

ActionScript source is compiled to bytecode, that is run by the ActionScript
Virtual Machine. I strongly recommend you to read an
<a href="{{ site.assets_url }}/media/2014-05-17-reverse-engineering-flash/avm2overview.pdf" target="_blank">overview of the AVM</a>
now, to be able to understand the assembly better.

ActionScript bytecode is placed into a `DoABC` tag on the SWF. An SWF can
contain multiple `DoABC` tags. When such a tag is found, the player loads the
bytecode, verifies it[^1] and runs it.

[^1]: "Verification" means the code is checked for overflows, invalid jumps or other illegal operations. At any point is the SWF checked for a signature from the publisher, which can be done in Java.


## Setting up

We're going to install the software that will allow us to see inside SWFs.

### Basic things

We need a working D compiler. Better [download it from the official site](http://dlang.org/download.html), since the
APT version often causes trouble. Then, install it:

~~~ bash
sudo dpkg -i <path to the .deb>
~~~

Make sure `flashplugin-installer` is installed (not `adobe-flashplugin`):

~~~ bash
sudo apt-get install flashplugin-installer
~~~

Git, the JDK, and LZMA development files are also needed:

~~~ bash
sudo apt-get install git openjdk-7-jdk liblzma-dev
~~~

### [RABCDAsm](https://github.com/CyberShadow/RABCDAsm)

~~~ bash
git clone https://github.com/CyberShadow/RABCDAsm.git && cd RABCDAsm
rdmd build_rabcdasm.d
sudo install {abc,swfbin}{export,replace} rabc{d,}asm swf{7z,lzma,de}compress /usr/local/bin
~~~

RABCDAsm contains utilities for:

  * Extracting ABC blocks from an SWF file (`abcexport`), and replacing them
    (`abcreplace`).
  * Disassembling the ABC blocks into a well structured assembly language
    (`rabcdasm`) and assembling them back (`rabcasm`).
  * Extracting binary tags from an SWF file (`swfbinexport`), and replacing them
    (`swfbinreplace`). We've said earlier that these tags can contain any data,
    and are often used to hide resources or whole SWFs.
  * Manual compression and decompression of an SWF file. All the other utilities
    can deal with compressed SWF ---there's no need to decompress them first---
    but these are provided for debugging and manual inspecting of SWFs.

The code also allows for programmatic parsing and manipulation of SWFs and their
tags, as well as deep parsing and manipulation of ActionScript blocks. The
disassembler can be easily tuned to modify the formatting of the disassembly.

RABCDAsm is fast and resistent to any obfuscations applied to the bytecode.
It's typically used like this:

~~~ bash
abcexport file.swf # will create file-0.abc, file-1.abc, file-2.abc, etc.
rabcdasm file-0.abc # disassemble!
rabcdasm file-1.abc
rabcdasm file-2.abc
~~~

Which disassembles each block in the directories `file-0`, `file-1`, `file-2`,
... After editing, to assemble the ABC and update the SWF:

~~~ bash
rabcasm file-0/file-0.asasm # assemble!
rabcasm file-1/file-1.asasm
rabcasm file-2/file-2.asasm
abcreplace file.swf 0 file-0/file-0.asasm # replace the blocks
abcreplace file.swf 1 file-1/file-1.asasm
abcreplace file.swf 2 file-2/file-2.asasm
~~~

### [redasm-abc](https://github.com/jmendeth/redasm-abc)

~~~ bash
git clone --recurse https://github.com/jmendeth/redasm-abc.git && cd redasm-abc
rdmd build_redasm.d
sudo install redasm /usr/local/bin
~~~

redasm-abc is a simple assistant to RABCDAsm. It aims to remove the tedious
workflow you just saw. To use redasm-abc, put the SWF in an empty directory,
then just run:

~~~ bash
redasm
~~~

And it will disassemble all the blocks at `block-0`, `block-1`, `block-2`, ...
When you have made changes and want to update the SWF, run again:

~~~ bash
redasm
~~~

And it will reassemble the files that have changed. It will work from everywhere
inside the directory of the SWF. It also creates a backup of the SWF, just in
case.

redasm-abc is [especially useful in SWFs with lots of blocks](http://showterm.io/f8e8416a2968828a75353),
and it doesn't create intermediate files so it's more comfortable to use.
Sometimes though, RABCDAsm utilities need to be used directly.

### [Flash Player debugger](http://www.adobe.com/support/flashplayer/downloads.html)

~~~ bash
wget http://fpdownload.macromedia.com/pub/flashplayer/updaters/11/flashplayer_11_plugin_debug.i386.tar.gz
tar xzOf flashplayer_11_plugin_debug.i386.tar.gz libflashplayer.so > libflashplayer-debug.so
sudo install -m 644 libflashplayer-debug.so /usr/lib/flashplugin-installer
sudo update-alternatives --install "/usr/lib/mozilla/plugins/flashplugin-alternative.so" "mozilla-flashplugin" /usr/lib/flashplugin-installer/libflashplayer-debug.so 50

# only if you are running 64-bit
sudo apt-get install libnss3:i386 nspluginwrapper
sudo nspluginwrapper -i /usr/lib/flashplugin-installer/libflashplayer-debug.so
sudo mv /usr/lib/mozilla/plugins/npwrapper.libflashplayer-debug.so /usr/lib/flashplugin-installer/libflashplayer-debug.so
~~~

The Flash Player content debugger is essential if you're going to modify your
SWF. You get a nice error box showing the error instead of the player stopping
abruptly.

To switch between the regular Flash player and the debugger, do:

~~~ bash
sudo update-alternatives --config mozilla-flashplugin
~~~

And restart the web browser to use it. **Edit:** Chromium recently dropped support for NSAPI,
so the flash debugger won't work in it. Use another browser instead. If someone knows a way to
debug with PepperFlash, please post a comment!

Visit `about:plugins` to verify that the correct plugin has loaded.

### [Vizzy](http://code.google.com/p/flash-tracer)

> To install, [download](http://code.google.com/p/flash-tracer/downloads) the ZIP for Linux and extract it.

Vizzy is a small tool to display the Flash Player logs. You just run the JAR
and it shows highlighted real-time logs, allowing you to filter by keywords.

This is handy when you want to get some values from the SWF at runtime.
To see them in the logs, just `trace()` them:

~~~ asasm
findproperty        QName(PackageNamespace(""), "trace")
  ; (push the value to the stack)
callpropvoid        QName(PackageNamespace(""), "trace"), 1
~~~

### [SWFTools](http://swftools.org) (optional)

~~~ bash
sudo apt-get install swftools
~~~

They have some interesting utilities, namely:

  * `swfdump` parses the SWF and outputs a dump of its structure.
    You can see which tags, sprites, IDs, are there, and at which offset
    they're found.
  * `swfextract` extracts specific assets from an SWF (images, streams or whole
    frames). You need to lookup their IDs through `swfdump` first.
  * `swfstrings` extracts strings out of an SWF.

I won't go into their usage, that's out of the scope of this post.
But the dump should be minimally intuitive to read, especially if
you have worked with Flash before.

### Intercepting proxy

Requests made by Flash aren't usually logged on the Developer Tools console (even though
they're cached by the browser) so you'll often need a good MITM proxy to save SWF files,
see what other SWFs are being loaded and serve the reassembled copy instead.

I've been using MITMProxy (which works with HTTPS out of the box, and with IPTables you
can do transparent proxying) together with a hand-written Node proxy server, but I find
that too low-level.

Fiddler also has an alpha build for Linux that looks promising, but it isn't open-source.

### Other software

There are some other open-source utilities for SWFs, but I don't consider them
to be of much use in reverse-engineering.
The [Ming library](http://libming.org), [swfmill](http://swfmill.org/),
`swfc` (part of SWFTools), the [Flex toolkit](http://flex.apache.org),
[JPEXS](https://github.com/jindrapetrik/jpexs-decompiler) ---that one might be
useful, but I haven't tried it against obfuscated files---
[Flasm](http://nowrap.de/flasm.html), [MTASC](http://mtasc.org).


## Some tips

Put the SWF in his own directory and add the files to a Git repository
just after disassembling it:

~~~ bash
redasm
git init && git add -A
git commit -m "disassemble SWF"
~~~

Always run these commands when getting on an SWF, even if you're only planning
to read the assembly. You'll thank me later.

Save [this page](http://www.anotherbigidea.com/javaswf/avm2/AVM2Instructions.html)
as a reference for the AVM instructions.
Also, the syntax used in the disassembly is explained in [the README](https://github.com/CyberShadow/RABCDAsm#syntax).


## Conclusion

While it's a bit tedious to read the disassembly, these tools really give us
a lot of control over the SWF, and the fact they're open-source gives you the
ability to tune them or build on top of them (like I did with redasm-abc).
