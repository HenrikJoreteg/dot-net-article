# Building a realtime team dashboard app with node.js and Backbone.js
## Part 1 of 2: Introduction and setup

The future of the web is realtime. The reason I can say this with such certainty is because *it's already happening* under our noses.

Facebook, gmail, gtalk, github just to name a few, have all implemented some form of automatic page updating. When they have something new to tell you, they don't wait for you to ask for it. They push it out to you, from the server to the client.

In some cases this is as simple as the page automatically polling to see if there's something new. In other cases it's more advanced where all the data used to build and update the page is coming over an open websocket connection. For our purposes, the transport mechanism is largely irrelevant, the point is, data comes to you.

This inherently breaks the statelessness of webpages. It used to be that I hit a URL and got back a webpage. As a user I understood that the information on the page was accurate as of the time it was requested. If I wanted to check for something new, I'd go ask for it again and got another snapshot in time.

As soon as we make any effort to keep the information on the page in sync with the server, we've now acknowledged that the webpage has "state". The page always had state, but, when we as developers decide we want to do partial updates of the page, the only way we can do so is by knowing what we currently have and comparing it to the server. State duplication has occurred and we're now maintaining "state" in some form in the client.

As users get increasingly comfortable with that idea, we'll reach a point where always-current, self-updating information is the expectation rather than a surprise. I *promise* you, this will happen, whether you're on board or not. If you want to stay at the top of your field as a web developer, you'll have to know how to build apps that work this way.

When you duplicate state, you increase complexity. Because now, rather than worrying about just rendering some data correctly, you're now caring about staleness, caching, and merge conflicts.

If we step back a bit and look at what we're actually doing, we start to realize that we're actually building is a distributed system and we'll have all the same challenges that come with building distributed systems.

I know what you're probably thinking. Some framework is going to come along that solves this problem for me. You may be right, there are many different approaches to dealing with the problems of duplicated state. There are many frameworks such as Meteor.js, SocketStream, and Derby.js that aim to simplify the process of building apps that work this way. 

The challenge with those frameworks, from where I sit, is that there's a lot of emphasis on trying to share code and logic between the client and the server when client/server really should be performing fundamentally different roles. Servers are for data, clients are for presentation. To me, this is just basic principle of seperation of concerns. From everything I've read and heard, the way you solve complex problems is by *not* solving complex problems. Instead, complex problems are solved by building small, maintainable pieces that tackle a small portion of the complex problem. 

Why bring the complexity of the server to the client and vice/versa? In addition, when you try to share too much server code with a browser it's very easy to fall into the trap of tightly coupling your application to that particular client. This makes it much harder to build other clients, say for example a native iOS app for your app. So while these frameworks are useful for webapps, they may let us down a bit when we want to go beyond that. With more and more talk of "the Internet of things" we have good reason to believe that the breadth of device types that may want to talk to your app will continue to increase.

## Code structure and singe-page apps
If we've decided that we want our server to be able to focus on data we may as well transfer as much of the rendering and presentation of client to the client. Some devs are advocating partiall rendering on the server, etc. To me, once we recognize we're building a "thick" client. We may as well render it all there. Othewise if part our page state is rendered on the server we have to somehow re-translate the HTML into state information. Again, I prefer an explicit separation of concerns.

As a result, below is the entirety of the html we send to the browser for our product And Bang:

```html
<!DOCTYPE html>
<script src="/&!.js"></script>
```

Yup, that's it (and yes, omitting `<html>`, `<head>`, and `<body>` is allowed by the HTML specs).

The purpose of this extreme minimalism is primarily aesthetic. But, it also makes it  *abundantly* clear that it's the client's responsibility to render the application and manage everything within it, including the page title and life-cycle of all the page elements.

## MVC or some such acronmym
If you've ever tried to build a complex single-page app. You know that keeping your code clean and tidy is one of the toughest challenges.

I can tell you from my exprience, that the best way to stay sane (and that's not an exaggeration), is to make darn sure that you cleanly seperate your application state from the DOM. This lets your DOM simply be a reflection of current model state.

## Ok, let's build something cool!
Realtime apps suck without an external data source. Basically, what's the point of updating a page if nothing's changed that you didn't change?! So, I'm going to walk you through how to build dashboard app for your team using And Bang's shiny new real-time API. If you've seen stuff like the Panic Board you'll know what I'm talking about. When we're done we'll be able to keep this app running and if our team is using And Bang for collaboration we'll be able to se what everybody's up to. 

We'll try to keep the scope fairly narrow, but I'll be sure to throw in some goodies to keep it interesting. Hopefully it'll show you a lot of the tools and techniques needed in order to build these types of applications. The finished app is available at: [https://github.com/HenrikJoreteg/sample-dashboard.andbang.com](https://github.com/HenrikJoreteg/sample-dashboard.andbang.com).

## Give ‘em the dashboard!
Our final app will look like this:

!["Team dashboard app screenshot"](http://f.cl.ly/items/2w010C3Q0G1W1c1N3H2R/Screen%20Shot%202012-11-12%20at%2012.22.00%20AM.png)

There's a "card" for each member on the team. We'll live-sort and animate these member cards based on who is online and if they're working or not. We can also see how many items they've "shipped" that day based on how many little pink rockets there are on their card. Plus, recent shipped items for the whole team appear in the list on the right-hand side. Our app will only render shipped items that happened that day and will be reset at 4:00am each night.

## Basic setup
You'll need node.js 0.8+ and npm installed on your system (these days, npm ships with node). I'm assuming a mac or linux environment, but with very minor tweaks you should be able to follow along just fine if you're on windows.

If you want to follow along from scratch, we'll create a folder structure that looks like this:

```
├── clientapp
│   ├── models - folder for our backbone models
│   ├── views - folder for backbone views
│   └── app.js
├── clientmodules - flat folder of all our single-file modules like backbone.js, underscore, etc.
├── clienttemplates - folder of our clientside jade templates
├── public - folder of our public, static resources
│
├── app.html - main app html
├── dev_config.json - our config file
├── package.json - package description
├── README.md
└── server.js - Main application file
```

First we know we'll need to install some stuff. Create a new folder for your project and create a package.json file in it that looks something like this:

```json
{
  "name": "sample-dashboard.andbang.com",
  "version": "0.0.1",
  "homepage": "https://sample-dashboard.andbang.com",
  "repository": {
    "type": "git",
    "url": "git@github.com:henrikjoreteg/sample-dashboard.andbang.com.git"
  },
  "description": "Mind-meldification for teams",
  "dependencies": {
    "backbone": "",
    "underscore": "",
    "express": "",
    "stitch": "",
    "andbang-express-auth": "",
    "precommit-hook": "",
    "clientmodules": "",
    "templatizer": "",
    "andlog": "",
    "getconfig": ""
  },
  "clientmodules": ["andlog", "backbone", "underscore"],
  "main": "server.js",
  "scripts": {
    "postinstall": "node node_modules/clientmodules/install.js"
  }
}

```

Next change into your new project folder and type:

```
npm install .
```

This should install everything you need. 

Next we need to build a server.js file. This will set up our server application and handle everything we need to do oAuth with andbang.com so we have access to our team data. 

See the comments inline:

```js
/*
server.js

This is the *main* application file. It will create and configure our server
and begin listening for requests.
*/

// First we import the modules we'll need:
var express = require('express'),
    stitch = require('stitch'),
    andbangAuth = require('andbang-express-auth'),
    templatizer = require('templatizer');

// build our client templates into vanilla javascript
templatizer(__dirname + '/clienttemplates', __dirname + '/clientmodules/templates.js');

// Then we define the files that will make up our clientside application. 
// We use Stitch (https://github.com/sstephenson/stitch) to allow us to 
// organize our browser.js using the same CommonJS module pattern we use 
// on the server.
var clientSideJS = stitch.createPackage({
        paths: [__dirname + '/clientmodules', __dirname + '/clientapp'],
        dependencies: [
            __dirname + '/public/jquery.js',
            __dirname + '/public/sugar-1.3.6-dates.js',
            __dirname + '/public/socket.io.js',
            __dirname + '/public/init.js'
        ]
    });

// Now create and configure our express.js application
// docs at: http://expressjs.com
var app = express();

app.use(express.static(__dirname + '/public'));
app.use(express.cookieParser());
app.use(express.session({ secret: 'neat-o bandit-o' }));
app.use(andbangAuth.middleware({
    app: app,
    clientId: 'andbang-client',
    clientSecret: 'dev-client-secret',
    defaultRedirect: '/',
    loginPageUrl: '/auth'
}));

// We can now use our application to specify which url we'll use to server the
// clientisde application. Stitch turns it into a single file for us and while we're
// in development it can also watch for changes and update itself so we don't have
// to re-start the server each time we change any of our client files.

// We wouldn't want to serve our application this way in production. For that, we can 
// write the file to disk, minify it and just serve it as a static javascript file.
app.get('/&!-dashboard.js', clientSideJS.createServer());

// we could just
app.get('/', andbangAuth.secure(), function (req, res) {
    res.sendfile(__dirname + '/app.html');
});

// a helper for getting our token
app.get('/token', andbangAuth.secure(), function (req, res) {
    res.send(req.session.accessToken);
});

// start listening for requests
app.listen(3003);

// write some helpful log output
console.log('Dashboard is running at: http://localhost:3003. Yep. That\'s pretty awesome.');
```

## Breaking it down part 1: Auth
You'll see we're using some middleware to handle oauth with And Bang for us. For this to work, you'll need to enter you client ID and secret.

```js
app.use(andbangAuth.middleware({
    app: app,
    clientId: '<< YOUR CLIENT APP ID >>',
    clientSecret: '<< YOUR APP SECRET >>',
    defaultRedirect: '/',
    loginPageUrl: '/auth'
}));
```

Then, by securing the main route with the auth middleware, you'll immediately be re-directed to log in via andbang. You can see that in action here. Note the `andbangAuth.secure()` middleware:
```
app.get('/', andbangAuth.secure(), function (req, res) {
    res.cookie('apiToken', req.session.accessToken, {expires: new Date(Date.now() + 30000)});
    res.sendfile(__dirname + '/app.html');
});
```

## Breaking it down part 2: Stitch and the clientside app bundle
We use [stitch](https://github.com/sstephenson/stitch) written by [Sam Stephenson](https://twitter.com/sstephenson) of 37signals. This lets us keep our clientside code cleanly seperated into modules that only do one thing. 

If you're unfamiliar with CommonJS modules, it's a patterns for structuring modules where you explicitly `require` other modules you want to use and `export` functions or objects that you want to make available to other modules. 

Nothing has done more for the cleanliness of my clientside applications that switching to this approach. Since node.js also uses this module system it also has the added benefit of letting me re-use a module on the server or client.

In addition it lets you structure your code into as many files and folders as make sense to you, and not worry about what that does to the various `<script>` tags in your html.

Stitch simply takes your folder structure and builds and concatenates all of it into a single JS file. 

It also supports using regular JS libraries that are not written in the CommonJS style, like jQuery for example.

For other files that we want to include, we simply list them out as dependencies, in the order we want them like so: 

```js
var clientSideJS = stitch.createPackage({
        paths: [__dirname + '/clientmodules', __dirname + '/clientapp'],
        dependencies: [
            __dirname + '/public/jquery.js',
            __dirname + '/public/sugar-1.3.6-dates.js',
            __dirname + '/public/socket.io.js',
            __dirname + '/public/init.js'
        ]
    });

```

During development we just that bundle as it were a server. This lets us make changes to modules and not have to worry about restarting or recompiling anything, Stitch just handles that for us by creating a little server that watches our directories for us like so:

```js
app.get('/&!-dashboard.js', clientSideJS.createServer());
```

When it comes time to put this in production you'll want to just write the file to disk, minify it and just serve it statically. Stitch makes that easy too, with its `compile()` function. Assuming `clientSideJS` was our stitch package from above we could minify and write it to disk like this:

```js
// This script builds the &!.js and public.js files for production use
var uglifyjs = require('uglify-js'),
    fs = require('fs');

function uglify(code) {
    var ast = uglifyjs.parser.parse(code);
    ast = uglifyjs.uglify.ast_mangle(ast);
    ast = uglifyjs.uglify.ast_squeeze(ast);
    return uglifyjs.uglify.gen_code(ast);
}

clientSideJS.appPackage.compile(function (err, source) {
    fs.writeFileSync('build/&!.js', uglify(source));
});
```

## Next Month, the client.
Now we're fully set up and ready to start building the clientside app. We've got our app server, authentication, client templating, and client package management. Next month we'll dig into backbone and really start to see what it's like to build a realtime app with backbone.

Hope you had fun and learned something. I'm @HenrikJoreteg on twitter. Be sure to ping me on twitter with any thoughts, questions or feedback. I'd love to hear from you.