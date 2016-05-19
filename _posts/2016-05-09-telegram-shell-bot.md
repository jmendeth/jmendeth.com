---
layout: post
title: Telegram shell bot
cover:
  image: /media/2016-05-09-telegram-shell-bot/cover.jpg
  source: https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/res/drawable-hdpi/background_hd.jpg
quote: This handy bot runs commands on demand and sends the live output, allowing you to interact at any time. It can even run graphical apps!
comments: 2016-05-09-telegram-shell-bot
---

If you use Telegram regularly, you've probably heard of [bots](https://core.telegram.org/bots). They're like regular accounts, you can talk to them or have them in groups and they'll do all sorts of useful things for you.

In this post, I present you the [shell bot](https://github.com/jmendeth/node-botgram/tree/master/examples/shell)! Unlike other bots, this one is self-hosted: you use it by running your own instance of the bot in your server.

This handy bot runs commands on demand and sends the live output, allowing you to interact at any time. It can even run graphical apps!

{% include image.html url="/media/2016-05-09-telegram-shell-bot/example-1.jpg" description="The shell bot running some simple commands." %}

It was made as an example for the [botgram](https://github.com/jmendeth/node-botgram) framework, but I've been using it for some time and it's proven to be very useful. When you're on mobile and just want to issue a few commands, talking to a bot is probably more convenient than opening an SSH app and connecting to the server.

{% include image.html url="/media/2016-05-09-telegram-shell-bot/example-2.jpg" description="Graphical application example, alsamixer in this case." %}

It has creative uses as well: I once was troubleshooting a server problem with some friends, so we added the bot to the group and started playing around. Everyone could see what was going on, and suggest possible causes or things to try.

If you want to have your own shell bot, this post will guide you through the process of creating the Telegram bot itself, and installing the necessary software.

## Create your bot

First of all, you need to register a Telegram bot. Don't worry, it only takes some seconds. [Click here](https://telegram.me/BotFather) to talk to the BotFather, then say `/newbot` and you'll be asked some things (name, username, etc.). At the end, you'll be given an **authentication token** that looks like the following:

    123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11

## Preparations

The following should be done on the computer you want to run commands on. First of all, make sure you have [Node.JS](https://nodejs.org) installed (you can verify by running `npm -v`). You'll also need a working compiler and build tools:

~~~ bash
sudo apt-get install build-essential
~~~

Clone the project and install dependencies:

~~~ bash
git clone https://github.com/jmendeth/node-botgram.git
cd node-botgram
npm install
~~~

Now, before running the shell bot, you need to know your numeric **user ID**. To do this, run:

~~~ bash
node examples/print <auth token>
~~~

Replacing `<auth token>` with what you got from the BotFather. Then say something to your bot, and it'll print something like:

    Got a message from user 97438879 (Xavier Mendez):
    
    [...]

Indicating your numeric ID (mine is 97438879).

## Run it!

Now that we have everything set up, we can run the shell bot:

~~~ bash
cd examples/shell
npm install
npm start <auth token> <your ID>
~~~

If you receive a message saying `Bot ready` then you have a working shell bot! Try saying `/run uname -a` for example, or say `/help` to learn about the available commands.

## Autostart

Once you've played around, you may want the bot to start automatically on boot, and respawn if it crashes. For that, let's install [forever](https://github.com/foreverjs/forever):

~~~ bash
sudo npm install -g forever
~~~

Then, from your `/etc/rc.local` or an init script, call:

~~~
forever start /path/to/node-botgram/examples/shell/server.js <auth token> <your ID>
~~~

Also, it's a good idea to talk to the BotFather and say `/setcommands` to define a list of commands your bot accepts. You'll find this list in [`commands.txt`](https://github.com/jmendeth/node-botgram/blob/master/examples/shell/commands.txt); just paste the contents when asked. You may also want to change the bot's profile photo and description.

Did you find the bot useful? Got any suggestions? Feel free to tell me about it in the comments!
