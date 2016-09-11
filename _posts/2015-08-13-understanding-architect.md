---
layout: post
title: Understanding Architect
cover:
  image: /media/2015-08-13-understanding-architect/cover.jpg
quote: This micro-framework is more than a loader, it presents an entirely new way to build Node apps.
comments: 2015-08-13-understanding-architect
---

If you have worked with [Cloud9](https://c9.io), you may have heard about
[Architect](https://github.com/c9/architect):

> Architect is a simple but powerful structure for Node.js applications.

I have to admit, after reading the whole readme for the first time, I
still wasn't seeing the potential that such a simple system hides. In
my opinion, the best way to present the benefits of a plugin-based app
will be with a practical example. Without further ado, meet...

# The Logger™

It's not needed at all, but I suggest you to read [Low Power WiFi
Datalogger](https://learn.adafruit.com/low-power-wifi-datalogging). It covers
the creation of a battery-powered Arduino, that connects to your WiFi every
few minutes, and reports some measures.

Here, we'll build the other part of the system&mdash;the server that collects
and accumulates measures from a series of these dataloggers.

So here's the thing: we have ten dataloggers, which will regularly open a
TCP connection to us and report their readings (currently one). Our server
has to store all readings for the last ten hours and present them in a
nice website.


## Planning

We can see there are three basic parts in our application:

 - The module that listens for connections from an Arduino, identifies it
   and decodes the reading. We'll call this the **backend**.
 - The module that stores the readings, associated with each logger.
   This will be the **storage**.
 - The module that presents the readings in a nice UI.
   This will be the **frontend**.

We can also include a **logger** module where all modules can push messages
whenever something does not work as expected. This is our dependency graph,
in glorious ASCII art:

    +---------+             +---------+             +----------+
    | backend |----------->>| storage |<<-----------| frontend |
    +---------+    save     +---------+     query   +----------+
        |                        |                        |
        | log                    | log                    | log
        |        +--------+      |                        |
        `------>>| logger |<<----+------------------------'
                 +--------+

Alright, time to start coding!

~~~ bash
mkdir the-logger && cd the-logger
echo "Datalogger server" > README.md
wget http://jmendeth.mit-license.org -O LICENSE
npm install architect
~~~

We'll start with `backend` module, or rather, plugin. Which takes us to...


## Our first plugin

"Plugin" may sound like a bold word, but fear not! A plugin is simply
a `package.json` with some metadata and a JavaScript file with code.
That's why they need to be in their own folders:

~~~ bash
mkdir backend && cd backend
~~~

The `package.json` will say: «This is called `backend`, it's a plugin, and it
wants to use things from `storage`»

{% highlight json %}
{
  "name": "backend",
  "plugin": {
    "consumes": ["storage"]
  }
}
{% endhighlight %}

As always, you could go on and include a `description`, `version`, `keywords`,
`author`, etc. but as long as it has a `name` and a `plugin` section, it's good.

We'll now create `index.js` next to it, with the following boilerplate:

{% highlight js %}
module.exports = function(options, imports, register) {

}
{% endhighlight %}

As you can see, `index.js` exports a single function, the constructor, which
creates a new instance of that plugin. It takes three parameters. The
first has user-provided preferences for this plugin. The second contains
the dependencies we asked for (in this case, only `storage`). And instead
of returning the result, it calls `register` with it. Example of how our
constructor may get called:

{% highlight js %}
constructor({ port: 1940, verbose: true }, { storage: <API> }, result_callback);
function result_callback(error, result) {
  if (error)
    console.log("Plugin failed to initalize!");
  else
    // continue...
}
{% endhighlight %}

This plugin is really simple, it just creates a server and, whenever
a measure arrives, call a method of our `storage` plugin to store it.

{% highlight js %}
module.exports = function(options, imports, register) {
  var net = require("server");
  // Retrieve our storage dependency
  var storage = imports.storage;

  // Create our server
  net.createServer(function(client) {
    client.on("end", function() {
      // Parse what it sent us
      var parsed = client.read().match(/^(\d+)\n$/);
      var measure = parseInt(parsed[1]);

      // Call the storage plugin to save the measure
      storage.storeMeasure(client.remoteAddress, measure);
    });
  }).listen(options.port || 8000);

  // Tell whoever called us we're ready
  register(null, {});
}
{% endhighlight %}

This plugin does not provide any API for other plugins to use,
that's why `register` is called with an empty object.


## Our second plugin

Let's now create the storage plugin. Because we're just testing that
the whole thing works, and don't want to bring in any database yet,
the plugin will just store the measures locally, in memory.

~~~ bash
cd ..
mkdir local-storage && cd local-storage
~~~

This time, the `package.json` looks a bit different. This doesn't depend on
any plugin, but it provides the `storage` API for other plugins to use.

{% highlight json %}
{
  "name": "local-storage",
  "plugin": {
    "provides": ["storage"]
  }
}
{% endhighlight %}

And `index.js`:

{% highlight js %}
module.exports = function(options, imports, register) {
  var database = {};

  // This is the API object that will be passed to other plugins
  var api = {};

  // Plugins will call this function to store a measure
  api.storeMeasure = function(ip, measure) {
    if (!(ip in database)) database[ip] = [];
    database[ip].push({time: Date.now(), value: measure});
  };

  // Plugins will call this function to query all stored
  // measures for a given IP, sorted by time
  api.queryMeasures = function(ip) {
    return database[ip];
  };

  // Plugins will call this function to query all IPs we know
  api.listIPs = function() {
    return Object.keys(database);
  };

  // Periodically clean old measures from the database
  setInterval(function() {
    var now = Date.now(), limit = options.limit || (10 * 3600 * 1000);
    database.map(function(entry) {
      return entry.filter(function(m) { return (now - m.time) > limit });
    });
  }, options.cleanInterval || (600 * 1000));

  // Done! Export our storage API for other plugins to use
  register(null, { storage: api });
}
{% endhighlight %}

Halt. There's something important to note here. `backend` is *not* depending on
`local-storage`, it just wants us to give him a `storage` API to push measures
to. And this plugin we just wrote is a candidate of providing that API (the
only one, currently). This is called a **soft dependency**, and is the key
advantage of plugin-based apps. More on that later.


## The third plugin

Let's use Express for the webserver, and let's make it quick.

~~~ bash
cd ..
mkdir web-frontend && cd web-frontend
~~~

The `package.json` (by now you should know what's coming):

{% highlight json %}
{
  "name": "web-frontend",
  "plugin": {
    "consumes": ["storage"]
  }
}
{% endhighlight %}

And the `index.js`:

{% highlight js %}
module.exports = function(options, imports, register) {
  var express = require("express");
  // Retrieve our dependency
  var storage = imports.storage;

  var app = express();
  app.set("views", __dirname + "/views");
  app.set("view engine", "your favorite view engine");

  app.get("/", function(req, res) {
    var ips = storage.listIPs();
    res.render("list", {ips: ips});
  });

  app.get("/:ip", function(req, res, next) {
    var measures = storage.queryMeasures(req.params.ip);
    if (!measures) return next();
    res.render("measures", {ip: req.params.ip, measures: measures});
  });

  // This plugin doesn't export any APIs either
  register(null, {});
}
{% endhighlight %}

Writing the views and choosing the view engine is out of the scope
of this article.


## Launching the application

At this point we have all the plugins we need to start the first version
of The Logger™. But there's nothing we can actually run yet; something
has to call the create an instance of each plugin and put them together.

For that purpose let's create `run.js` on the project root, next to the readme:

{% highlight js %}
// Create storage, it's needed by the other plugins
require("./local-storage")({}, {}, function(error, result) {
  if (error) process.exit(1);

  // Create backend
  require("./backend")({}, { storage: result.storage }, function(error, result) {
    if (error) process.exit(1);
    console.log("Backend ready");
  });

  // Create frontend
  require("./web-frontend")({}, { storage: result.storage }, function(error, result) {
    if (error) process.exit(1);
    console.log("Frontend ready");
  });
});
{% endhighlight %}

And that's it, we have our app running! But it felt a bit tedious to write
that launcher, didn't it? **That's where Architect comes in.** Since we have
all the dependencies written in each `package.json`, we can replace the above
code with:

{% highlight js %}
var architect = require("architect");
var config = architect.loadConfig(__dirname + "/config.js");
architect.createApp(config);
{% endhighlight %}

And `config.js` just exports a list of plugins to load:

{% highlight js %}
module.exports = [
  "./backend",
  "./local-storage",
  "./web-frontend",
]
{% endhighlight %}

Then when we run `run.js`, Architect will figure out the right order in which
to create each plugin, making sure to satisfy all dependencies, avoiding
dependency cycles, etc. You only need to modify `config.js` as needed.

Now for some little practice: would you be able to add in a logger plugin,
and use it throughout the app? The plugin should just log messages to the
console.


## Why do you do this to me

Now you are probably thinking: couldn't we have just used modules and be done?
What do we gain by structuring our application in plugins?

**Scalability.**

Okay, now you want to take this application into a production environment.
Write a `mongodb-storage` plugin which saves the data to a MongoDB database.

Then you use `local-storage` or `mongodb-storage` depending on the environment
you're on. As long as both export the same API, nothing changes for the rest
of the code.

Oh, your server is not very powerful and you don't want to run any frontend
in it? Fine, disable the plugin. Or maybe you'd prefer the frontend to be
a REST API to query it from a big, central server? Write a REST frontend.

And what if you're a user and don't maintain the code, but just want
to hook up support for your custom datalogger? No problem.

Some plugins can be in other modules. Heck, you could have a freaking repo for
every plugin if you wanted. Anything `require()` and NPM can fetch works.
Here, the team that writes the Arduino code could also mantain the `backend`
plugin there.

Oh, and that also works the other way: other people can require our backend
plugin for their own purposes; as long as they provide a storage API.

This is just a subset of the advantages. I don't want to make it seem like
plugins are a magic bullet for Node.JS applications. It's far from that,
but they have allowed me to build complex designs very fast. I totally
recommend trying them with a more real example.

Happy coding!

PS: Want to know how to pass options to plugins? Register your own error
handler when a plugin fails? Curious to see the full Architect API? This and
more, in their [repository](https://github.com/c9/architect).
