# Building a realtime single-page app with Backbone.js

The future of the web is realtime. The reason I can say this with such certainty is because *it’s already happening* under our noses.

Facebook, gmail, gtalk, github just to name a few, have all implemented some form of automatic page updating. When they have something new to tell you, they don’t wait for you to ask for it. They push it out to you, from the server to the client.

In some cases this is as simple as the page automatically polling to see if there’s something new. In other cases it’s more advanced where all the data used to build and update the page is coming over an open websocket connection. For our purposes, the transport mechanism is largerly irrelevant, the point is, data comes to you.

This inherently breaks the statelessness of webpages. It used to be that I hit a URL and got back a webpage. I understood that the information on the page was accurate as of the time it was requested. If I wanted a newer state, I’d go ask for it again and got another snapshot in time.

As soon as we make any effort to keep the information on the page in sync with the server, we’ve now acknowledged that the webpage has “state”. The page always had state, but, when we as developers decide we want to do partial updates of the page, the only way we can do so is by knowing what we currently have and comparing it to the server. State duplication has occured and we’re now maintaining “state” in some form in the client.

As users get increasingly comfortable with that idea, we’ll reach a point where always-current, self-updating information is the expectation rather than a surprise. I *promise* you, this will happen, whether you’re on board or not. If you want to stay at the top of your field as a web developer, you’ll have to know how to build apps that work this way.

When you duplicate state, you increase complexity. Because now, rather than worrying about just rendering some data correctly, you’re now caring about the “realtime” accuracy of the data that you have on the page.

I know what you’re probably thinking. Some framework is going to come along that solves this problem for me. You may be right, there are many different approaches to dealing with the problems of duplicated state. There are many frameworks such as Meteor.js, SocketStream, and Derby.js that aim to simplify the process of building apps that work this way. 

The challenge with those frameworks, from where I sit, is that there’s a lot of emphasis on trying to share code and share everything between the client and the server when client/server really should be performing fundamentally different roles. Servers are for data, clients are for presentation. To me, this is just basic seperation of concerns when it comes to code. When you try to share too much server code with a browser you’ve instantly coupled your application to that particular client. This makes it much harder to build other clients, say for example a native iOS app for your app.

## Code structure and singe-page apps
If we’ve decided that we want our server to be able to focus on data we may as well transfer as much of the rendering and presentation of client to the client. It’s really complicated to rendering large portions of your page based on server state and then re-rendering other portions in the client. Once we recognize we’re building a “thick” client. We may as well render it all there. 

Below is the entirety of the html we send to the browser for our product And Bang:

```html
<!DOCTYPE html>
<script src="/&!.js"></script>
```

Yup, that’s it (and yes, omitting <html>, <head>, and <body> is allowed by the HTML specs).

The purpose of this extreme minimalism is primarily aesthetic. But, it also makes it  *abundantly* clear that it’s the client’s responsibility to render the application and manage everything within it.

## MVC or some such acronmym
If you’ve ever tried to build a complex single-page app. You know that keeping your code clean and tidy is one of the toughest challenges. 

I can tell you from my exprience, that the best way to stay sane (and that’s not an exaggeration), is to make darn sure that you cleanly seperate your application state from the DOM.

## Ok, let’s build something cool!
Realtime apps suck without an external data source. Basically, what’s the point of updating a page if nothing’s changed that you didn’t change?! So, I’m going to walk you through how to build dashboard app for your team using And Bang’s shiny real-time API. If you’ve seen stuff like the Panic Board you’ll know what I’m talking about. When we’re done we’ll be able to keep this app running and if our team is using And Bang for collaboration we’ll be able to se what everybody’s up to. 

We’ll try to keep the scope fairly narrow, but I’ll be sure to throw in some goodies to keep it interesting. Hopefully it’ll show you a lot of the tools and techniques needed in order to build these types of applications. The finished app is available at (github.com something, something).

## Basic setup
You’ll need node.js 0.8+ and npm installed on your system (these days, npm ships with node). I’m assuming a mac or linux environment, but with very minor tweaks you should be able to follow along just fine if you’re on windows.

First we know we’ll need to install some stuff. Create a new folder for your project and create a package.json file in it that looks something like this:

```json
{
  "name": “my-team-dashboard”,
  "version": "0.0.1",
  "dependencies": {
    "backbone": "",
    "underscore": "",
    "async": "",
    "express": "",
    "stitch": "",
    "andbang-express-auth": "",
    "precommit-hook": "",
    "andbang": “”,
    "clientmodules": "",
    "templatizer": "",
    "getconfig": ""
  },
  "clientmodules": ["andbang", "backbone", "underscore", "async"],
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
    config = require('getconfig'),
    templatizer = require('templatizer'),
    yetify = require('yetify');

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
            __dirname + '/public/d3.v2.js',
            __dirname + '/public/sugar-1.2.1-dates.js',
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
    clientId: config.oauth.clientId,
    clientSecret: config.oauth.secret,
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
app.listen(config.http.port);

// write some helpful log output
console.log('Dashboard is running at: http://localhost:' + config.http.port + ' Yep. That\'s pretty awesome.');
```

