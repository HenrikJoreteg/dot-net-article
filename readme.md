# Building a realtime single-page app with Backbone.js

The future of the web is realtime. The reason I can say this with such certainty is because *it's already happening* under our noses.

Facebook, gmail, gtalk, github just to name a few, have all implemented some form of automatic page updating. When they have something new to tell you, they don't wait for you to ask for it. They push it out to you, from the server to the client.

In some cases this is as simple as the page automatically polling to see if there's something new. In other cases it's more advanced where all the data used to build and update the page is coming over an open websocket connection. For our purposes, the transport mechanism is largerly irrelevant, the point is, data comes to you.

This inherently breaks the statelessness of webpages. It used to be that I hit a URL and got back a webpage. I understood that the information on the page was accurate as of the time it was requested. If I wanted a newer state, I'd go ask for it again and got another snapshot in time.

As soon as we make any effort to keep the information on the page in sync with the server, we've now acknowledged that the webpage has "state". The page always had state, but, when we as developers decide we want to do partial updates of the page, the only way we can do so is by knowing what we currently have and comparing it to the server. State duplication has occured and we're now maintaining "state" in some form in the client.

As users get increasingly comfortable with that idea, we'll reach a point where always-current, self-updating information is the expectation rather than a surprise. I *promise* you, this will happen, whether you're on board or not. If you want to stay at the top of your field as a web developer, you'll have to know how to build apps that work this way.

When you duplicate state, you increase complexity. Because now, rather than worrying about just rendering some data correctly, you're now caring about the "realtime" accuracy of the data that you have on the page.

I know what you're probably thinking. Some framework is going to come along that solves this problem for me. You may be right, there are many different approaches to dealing with the problems of duplicated state. There are many frameworks such as Meteor.js, SocketStream, and Derby.js that aim to simplify the process of building apps that work this way. 

The challenge with those frameworks, from where I sit, is that there's a lot of emphasis on trying to share code and share everything between the client and the server when client/server really should be performing fundamentally different roles. Servers are for data, clients are for presentation. To me, this is just basic seperation of concerns when it comes to code. When you try to share too much server code with a browser you've instantly coupled your application to that particular client. This makes it much harder to build other clients, say for example a native iOS app for your app. So while these frameworks are useful for webapps, they may let us down a bit when we want to go beyond that. With more and more talk of "the Internet of things" We have good reason to believe that the breadth of device types that may want to talk to your app will continue to increase.

## Code structure and singe-page apps
If we've decided that we want our server to be able to focus on data we may as well transfer as much of the rendering and presentation of client to the client. It's really complicated to rendering large portions of your page based on server state and then re-rendering other portions in the client. Once we recognize we're building a "thick" client. We may as well render it all there. 

Below is the entirety of the html we send to the browser for our product And Bang:

```html
<!DOCTYPE html>
<script src="/&!.js"></script>
```

Yup, that's it (and yes, omitting `<html>`, `<head>`, and `<body>` is allowed by the HTML specs).

The purpose of this extreme minimalism is primarily aesthetic. But, it also makes it  *abundantly* clear that it's the client's responsibility to render the application and manage everything within it, including the page title and life-cycle of all the page elements.

## MVC or some such acronmym
If you've ever tried to build a complex single-page app. You know that keeping your code clean and tidy is one of the toughest challenges. 

I can tell you from my exprience, that the best way to stay sane (and that's not an exaggeration), is to make darn sure that you cleanly seperate your application state from the DOM.

## Ok, let's build something cool!
Realtime apps suck without an external data source. Basically, what's the point of updating a page if nothing's changed that you didn't change?! So, I'm going to walk you through how to build dashboard app for your team using And Bang's shiny real-time API. If you've seen stuff like the Panic Board you'll know what I'm talking about. When we're done we'll be able to keep this app running and if our team is using And Bang for collaboration we'll be able to se what everybody's up to. 

We'll try to keep the scope fairly narrow, but I'll be sure to throw in some goodies to keep it interesting. Hopefully it'll show you a lot of the tools and techniques needed in order to build these types of applications. The finished app is available at (github.com something, something).

## Basic setup
You'll need node.js 0.8+ and npm installed on your system (these days, npm ships with node). I'm assuming a mac or linux environment, but with very minor tweaks you should be able to follow along just fine if you're on windows.

If you want to follow along from scratch, we'll create a folder structure that looks like this:

```
├── clientapp
│   ├── models - folder for our backbone models
│   ├── views - folder for backbone views
│   ├── apiHandler.js
│   ├── app.js
│   └── router.js (our backbone router)
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
  "name": "my-team-dashboard",
  "version": "0.0.1",
  "dependencies": {
    "backbone": "",
    "underscore": "",
    "async": "",
    "express": "",
    "stitch": "",
    "andbang-express-auth": "",
    "precommit-hook": "",
    "andbang": "",
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
Later we can just that bundle as it were a server. This lets us make changes to modules and not have to worry about restarting or recompiling anything. Stitch handles that for us during development. When it comes time to put this in production you can use stitch to instead just write the file to disk then we can minify it and serve it statically.

```js
app.get('/&!-dashboard.js', clientSideJS.createServer());
``` 

## Breaking it down part 3: Templates for the client, templatizer
I recently wrote a whole post on client-side templating that got a lot of attention on hacker news. The core of it being that sending a whole templating engine to the browser is silly. It's much faster and more efficient to pre-compile templates into vanilla javascript and just send your templates as plain javascript functions to the browser. This may be rather self-evident to you, but, until recently, there haven't been that many template engines that made that pre-compilation step easy to use. 

I wrote a tool called [templatizer](https://github.com/HenrikJoreteg/templatizer) that takes a folder of jade templates and creates a single js file with a function for each template file. That's the approach we're using in the app in the following line pulled from server.js:

```js
templatizer(__dirname + '/clienttemplates', __dirname + '/clientmodules/templates.js');
```

So, when the app executes it creates a "templates" modules and puts it into our "clientModules" directory. Now rendering html becomes as easy as this:

```js
var templates = require('templates');

myHtml = templates.member({name: 'something'});
```


## Building the client
Alright, with the basics covered let's build the client!

I like to start backbone apps by creating one main application controller. This doesn't have to be model, we just create a simple module that we'll attach to the browser window. This will be the only global we create.

The controller looks something like this:


```js
/*global window app */
var MainView = require('views/main'),
    TeamModel = require('models/team'),
    logger = require('andlog'),
    Backbone = require('backbone'),
    cookies = require('cookieReader'),
    _ = require('underscore'),
    API = require('andbang');


module.exports = {
    blastoff: function (spec) {
        var self = this;
        
        this.api = new API();
        this.team = new TeamModel();
        this.view = new MainView({model: this.team});
        
        this.api.on('*', _.bind(this.handleApiEvent, this));

        this.token = cookies('apiToken');
        this.api.validateToken(self.token);

        this.api.once('ready', function (user) {
            self.team.set('id', user.teams[0]);
            self.team.members.fetch();
            self.team.shippedTasks.fetch();
        });

        return this;
    },
};
```

The "blastoff" function you see here is our entry point. It will create our data containers and render the main application view. 

### The launch sequence

Let's break down the code in our blastoff function:

```js
this.api = new API();
this.team = new TeamModel();
this.view = new MainView({model: this.team});
```

First we'll init our api module that we `require`'ed above. It will automatically establish an (unauthenticated) socket.io connection to the api server. Since this piece takes a bit of time we want to start this process as soon as possible in our launch sequence.

We create the "team" object which we'll use as our main container for all of the application data (a.k.a "state"). The “team” object is just a backbone model, with a few collections attached. More on that later. In addition we init our main application view and pass it the team model that becomes its basis for knowing what else to render. The main app view renders itself as soon as the DOM is ready, so we don’t need to worry about that here.

Next, we set up a handler for all of our API events. This is how we’ll keep our client in sync when we get updates generated by activity in the API. We’ll go into this a bit more, later.

```js
this.api.on('*', _.bind(this.handleApiEvent, this));
```

In order to identify ourselves to the API we need to pass it our access token that we got by doing oauth on the server. If you’ll recall from our server.js file, we passed the token to the client along with the html in the form of a token called “apiToken”. So now, using a little cookie reader module we can read it and use it to log into the API:

```js
this.token = cookies('apiToken');
this.api.validateToken(self.token, function () {});
```
When the token is validated the API object emits a “ready” event that calls our callback with a “user” object containing details of who just logged in. This user object looks something like this:

```js
{
    firstName: “henrik”,
    lastName: “joreteg”,
    teams: [“47”]
}
```

As you can see, it includes an array of IDs for teams that we’re apart of. For the purposes of this app we’ll just pick the first one and fetch our initial member and task data.

```js
this.api.once('ready', function (user) {
    self.team.set('id', user.teams[0]);
    self.team.members.fetch();
    self.team.shippedTasks.fetch();
});
```

Generally, it’s good practice to make each model as self-managing as possible. Arguably we could have just had the team know that it should fetch its own data as soon as we knew its “id”. But, for the sake of readability and since it’s part of the application’s loading sequece. It’s nice to be able to read the code as it is written here and be able to see that it’s at this point that we fetch more data.




