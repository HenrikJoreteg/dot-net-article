# Building a realtime team dashboard app with Backbone.js

The future of the web is realtime. The reason I can say this with such certainty is because *it's already happening* under our noses.

Facebook, gmail, gtalk, github just to name a few, have all implemented some form of automatic page updating. When they have something new to tell you, they don't wait for you to ask for it. They push it out to you, from the server to the client.

In some cases this is as simple as the page automatically polling to see if there's something new. In other cases it's more advanced where all the data used to build and update the page is coming over an open websocket connection. For our purposes, the transport mechanism is largerly irrelevant, the point is, data comes to you.

This inherently breaks the statelessness of webpages. It used to be that I hit a URL and got back a webpage. As a user I understood that the information on the page was accurate as of the time it was requested. If I wanted to check for something new, I'd go ask for it again and got another snapshot in time.

As soon as we make any effort to keep the information on the page in sync with the server, we've now acknowledged that the webpage has "state". The page always had state, but, when we as developers decide we want to do partial updates of the page, the only way we can do so is by knowing what we currently have and comparing it to the server. State duplication has occured and we're now maintaining "state" in some form in the client.

As users get increasingly comfortable with that idea, we'll reach a point where always-current, self-updating information is the expectation rather than a surprise. I *promise* you, this will happen, whether you're on board or not. If you want to stay at the top of your field as a web developer, you'll have to know how to build apps that work this way.

When you duplicate state, you increase complexity. Because now, rather than worrying about just rendering some data correctly, you're now caring about staleness, caching, merge conflicts.

At this point, if we step back a bit and look at what we're doing, we start to realize that we're actually building a distributed system and we'll have all the same challenges that come with building distributed sytems.

I know what you're probably thinking. Some framework is going to come along that solves this problem for me. You may be right, there are many different approaches to dealing with the problems of duplicated state. There are many frameworks such as Meteor.js, SocketStream, and Derby.js that aim to simplify the process of building apps that work this way. 

The challenge with those frameworks, from where I sit, is that there's a lot of emphasis on trying to share code and logic between the client and the server when client/server really should be performing fundamentally different roles. Servers are for data, clients are for presentation. To me, this is just basic seperation of concerns when it comes to code. From everything I've read and heard, the way you solve complex problems is by not solving complex problems. Instead, complex problems are solved by building small, maintainable pieces that tackle a small portion of the complex problem. 

Why bring the complexity of the server to the client and vice/versa. In addition, when you try to share too much server code with a browser you've instantly coupled your application to that particular client. This makes it much harder to build other clients, say for example a native iOS app for your app. So while these frameworks are useful for webapps, they may let us down a bit when we want to go beyond that. With more and more talk of "the Internet of things" We have good reason to believe that the breadth of device types that may want to talk to your app will continue to increase.

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

We create the "team" object which we'll use as our main container for all of the application data (a.k.a "state"). The "team" object is just a backbone model, with a few collections attached. More on that later. In addition we init our main application view and pass it the team model that becomes its basis for knowing what else to render. The main app view renders itself as soon as the DOM is ready, so we don't need to worry about that here.

Next, we set up a handler for all of our API events. This is how we'll keep our client in sync when we get updates generated by activity in the API. We'll go into this a bit more, later.

```js
this.api.on('*', _.bind(this.handleApiEvent, this));
```

In order to identify ourselves to the API we need to pass it our access token that we got by doing oauth on the server. If you'll recall from our server.js file, we passed the token to the client along with the html in the form of a token called "apiToken". So now, using a little cookie reader module we can read it and use it to log into the API:

```js
this.token = cookies('apiToken');
this.api.validateToken(self.token);
```
When the token is validated the API object emits a "ready" event that calls our callback with a "user" object containing details of who just logged in. This user object looks something like this:

```js
{
    firstName: "henrik",
    lastName: "joreteg",
    teams: ["47"]
}
```

As you can see, it includes an array of IDs for teams that we're apart of. For the purposes of this app we'll just pick the first one and fetch our initial member and task data.

```js
this.api.once('ready', function (user) {
    self.team.set('id', user.teams[0]);
    self.team.members.fetch();
    self.team.shippedTasks.fetch();
});
```

Generally, it's good practice to make each model as self-managing as possible. Arguably we could have just had the team know that it should fetch its own data as soon as we knew its "id". But, for the sake of readability and since it's part of the application's loading sequece. It's nice to be able to read the code as it is written here and be able to see that it's at this point that we fetch more data.


## The Team Model
Much of the data management is handled by the team model. Let's take a look at what it does.

```js
var Backbone = require('backbone'),
    _ = require('underscore'),
    Members = require('models/members'),
    ShippedTasks = require('models/shippedTasks');


module.exports = Backbone.Model.extend({
    initialize: function () {
        this.members = new Members();
        this.members.on('change:presence change:activeTaskTitle reset add', this.updateOrder, this);
        this.shippedTasks = new ShippedTasks();
        this.shippedTasks.on('add reset', this.updateShippedTotals, this);
        ...
    },
    updateOrder: function () {
        var sorted = this.members.sortBy(function (member) {
                var online = member.get('presence') === 'online' ? 2 : 0,
                    working = !!member.get('activeTask') ? 1 : 0;
                return -(online + working);
            });

        sorted.forEach(function (member, index) {
            member.set('order', index);
        });
    },
    updateShippedTotals: function () {
        var totals = _.countBy(this.shippedTasks.models, function (task) {
                return task.get('shippedBy');
            });

        app.team.members.each(function (member) {
            member.set('shippedCount', totals[member.id] || 0);
        });
    },
    ...
});
```

The "team" model contains two child collections. "members" which contains "member" models for each person on our team and the "shippedTasks" collection which stores, you guessed it, models of tasks that have been completed or "shipped".

The goal is to have our app automatically sort team members visually according to their status. If they're online and working they should go at the top of the list, if they're online but not working they're next, and of they're offline, then we put them at the bottom. In addition, any time that order changes, rather than just having them "jump" around we want to smoothly slide them around and reposition them into a grid on the page. To accomplish this effect, we'll position the cards with "position: absolute" apply CSS3 transitions and then use JS to calculate and set their "top" and "left" values to shuffle them around. 

The result is a smoothly flowing re-arrangement when the order changes. So, in process it may look something like this: ![Team member grid animation screenshot](http://f.cl.ly/items/0Z0s3J2L190j43193A0T/Screen%20Shot%202012-11-12%20at%2012.30.18%20AM.png)

We're not going to try to maintain a certain order within the collection itself. But rather, we'll run a `updateOrder` function that will do this:
- calculate what position each member should be in
- set that position as the "order" property of that member

Then, each member view can listen for changes to its "order" property and position itself accordingly.

So, within this team object, any time "presence", "activeTaskTitle" changes on a member, or we add, remove or reset the collection we want to re-run the `updateOrder` function which creates a sorted array of members, based on whether they're online or have an active task. To get the exact order that we want we assign a different point value and return that from the `sortBy` function. We calculate a point value, we give two points if they're online, one point if they have an active task, then sort those point totals from highest to lowest. 

That way, we always have this order:
1. Online with active task
2. Online but no active task
3. Offline but with an active task
4. Offline with no active task

Then we loop through the array we just created and assign an "order" value for each. Here's the whole function:

```js
updateOrder: function () {
    var sorted = this.members.sortBy(function (member) {
            var online = member.get('presence') === 'online' ? 2 : 0,
                working = !!member.get('activeTask') ? 1 : 0;
            return -(online + working);
        });

    sorted.forEach(function (member, index) {
        member.set('order', index);
    });
},
```

Similarly, we also want to maintain a count for each member of how many things they've shipped that day. So, we've also registered a use `updateShippedTotals` handler that calculates that value for each member, each time something new gets added to the "shippedTasks" collection or the collection gets reset. 

In addition, we create a function that gets called at a regular interval. It's job is to set a string such as "thursday" as the "day" attribute of team. That way, if it changes we reset our collection of shipped tasks. For the purposes of this app, we'll consider anything before 4:00AM to be the same day. Since, for many up-late-at-night coders, midnight is when they start hitting their stride.

### A quick note on handling dates
Date math is hard, seriously. It's a giant pain when you try to do anything complex and you're just using vanilla JS. I'd strongly suggest using some sort of date utility. My personal favorite is the "dates" module of Sugar.js. I don't necessarily like to extend native objects too much, but to me, the pain of manipulating, comparing and parsing dates in JS makes this worth it. For example, we do `Date.create().addHours(-4).format('{weekday}')` to create a current date, subtract four hours and turn the resulting date into a string that represents the current weekday in a nice, legible way. This alone is perhaps not worth using a library for, but being able to do stuff like `Date.create().isAfter('March 1st')` is priceless. This is also great for parsing user input. To see what else you can do check out [http://sugarjs.com/dates](http://sugarjs.com/dates).You can create a custom sugar.js build that only includes the date functionality. It works in node.js too. I highly recommend it.

## The Member model
Our member models are pretty straight forward:

```js
var Backbone = require('backbone'),
    _ = require('underscore');

module.exports = Backbone.Model.extend({
    defaults: {
        shippedCount: 0
    },
    initialize: function () {
        this.on('change:activeTask', this.handleActiveTaskChange, this);
        this.handleActiveTaskChange();     
    },
    handleActiveTaskChange: function (model, val) {
        var self = this,
            activeTaskId = this.get('activeTask') || '';
        
        if (activeTaskId === '') {
            self.set('activeTaskTitle', '');
        } else {
            app.api.getTask(app.team.id, activeTaskId, function (err, task) {
                if (task) self.set('activeTaskTitle', task.title);
            });
        }
    },
    fullName: function () {
        return this.get('firstName') + ' ' + this.get('lastName');
    },
    picUrl: function () {
        return 'https://api.andbang.com/teams/' + app.team.id + '/members/' + this.id + '/image';
    }
});
```

We start by establishing a default value for number of things shipped. We also want to maintain a task title for each member who is currently working on something. But, when we initially requested the member data from the API we only get a "activeTask" attribute as an ID. So, we register a handler that fetches the task title each time our "activeTask" value changes. It then sets it as a property directly on the model itself. This way, we should always be able to just render the "activeTaskTitle".

In addition we have a couple of convenience methods for retrieving a URL we can use to get the users avatar and their full name.

## The main view

It's the main view's job to render everything in the `<body>` tag. In our app we hand it the "team" model as its root model.

```js
var BaseView = require('views/base'),
    _ = require('underscore'),
    templates = require('templates'),
    MemberView = require('views/member');
    
module.exports = BaseView.extend({
    initialize: function () {    
        this.model.shippedTasks.on('add', this.handleNewTask, this);
        this.model.shippedTasks.on('reset', this.handleTasksReset, this);
        this.model.members.on('add', this.handleNewMember, this);
        this.model.members.on('reset', this.handleMembersReset, this);
        $(_.bind(this.render, this));
    },
    render: function () {
        this.setElement($('body')[0]);
        this.$el.html(templates.app());
    },
    handleNewTask: function (model) {
        this.$('.shippedContainer').prepend(templates.shipped({task: model}));
    },
    handleTasksReset: function () {
        this.$('.shippedContainer').empty();
    },
    handleMembersReset: function () {
        this.model.members.each(this.handleNewMember, this);
    },
    handleNewMember: function (member) {
        var peopleContainer = this.$('.people'),
            view = new MemberView({model: member});
        peopleContainer.append(view.render().el);
    }
});
```

You'll notice that we start by registering some handlers for "add" and "reset" events for both "shippedTasks" and "members". Shipped tasks are basically just a log. When we get new ones, we want to render a template for them and add it to the "shippedContainer" element. Handling new members is a bit more complex, because we want to be able to store some additional logic in each member view to handle changes to the member objects. So for each of these we actually create a new "MemberView" and pass it the "member" object it represents. Then we render that view and append it to its container.

## The Member View

The member view contains a lot of what makes the app fun. 

It has a few specific tasks it needs to perform:
- Maintain it's own physical width and position on the page.
- Maintain a class of "online" or "offline" based on the user's presence.
- Add/remove an "active" class on its container based on whether a user has an active task or not.
- Draw one pink rocket for each "shipped" item.
- Remove itself from the DOM if the member is removed.


```js
var BaseView = require('views/base'),
    _ = require('underscore'),
    templates = require('templates');

module.exports = BaseView.extend({
    contentBindings: {
        activeTaskTitle: '.activeTask'
    },
    classBindings: {
        presence: ''
    },
    render: function () {
        this.setElement(templates.member({member: this.model}));
        this.handleBindings();
        this.model.on('remove', this.destroy, this);
        this.model.on('change:order', this.handleChangeOrder, this);
        this.model.on('change:activeTask', this.handleActiveTaskChange, this);
        this.model.on('change:shippedCount', this.handleShippedCountChange, this);
        $(window).on('resize', _.bind(this.handleChangeOrder, this));

        this.handleChangeOrder();
        this.handleActiveTaskChange();
        this.handleShippedCountChange();

        return this;
    },
    handleChangeOrder: function () {
        var order = this.model.get('order'),
            windowWidth = window.innerWidth,
            minimumWidth = 290,
            numberOfColumns = Math.floor((windowWidth - minimumWidth) / minimumWidth),
            columnWidth = (windowWidth / (numberOfColumns + 1)) - 10,
            row = Math.floor(order / numberOfColumns),
            column = order % numberOfColumns,
            rowHeight = 150;
        
        this.$el.css({
            top: (row * rowHeight) + 'px',
            left: (column * columnWidth) + 'px',
            width: (columnWidth - 10) + 'px'
        });

        $('.shippedContainer').css('width', columnWidth + 'px');
    },
    handleActiveTaskChange: function () {
        this.$el[this.model.get('activeTask') ? 'addClass' : 'removeClass']('active');
    },
    handleShippedCountChange: function () {
        var i = this.model.get('shippedCount'),
            container = this.$('.shippedCount');
        
        container.empty();
        while (i--) {
            container.append(templates.rocket());
        }
    },
    destroy: function () {
        this.model.off();
        this.remove();
    }
});
```

We start with two declarative bindings. These are not part of backbone.js but if you look at the "clientapp/views/base.js" file you'll see what they do. 

- `contentBindings` keep the property you enter bound to the selector you provide.
- `classBindings` bind the property you provide as a class on whatever element selector you give it. In this case, doing empty string means maintaining a class on `this.el` itself.

It's really in `handleChangeOrder` that we get fancy. Anytime this model's "order" attribute is set or the window is resized we recalculate it's physical size and position. This setup allows us to create an animated responsive layout. Which is handy for a dashbaord app that could be rendered on a 73" LED tv or someone's little laptop. The math here will calculate an appropriate amount of columns and cell widths and set it directly on the element itself. By doing this, we get the animation and tweening for free by simply setting a transition in our CSS using CSS3 transitions.

We also bind a `destroy` method to our model that gets called if a model is removed. That way our view unbinds any handlers and calls backbone's `remove` method to remove this element from the DOM.

## Wiring it all up to receive events from the API
The big piece that's left is actually making sure that we properly handle updates we receive from the API. Luckily, it's actually quite simple. This is where we make use of that wildcard event handler we talked about in our controller.

Look at the bottom of that controller and you'll see our handler. It looks like this:

```js
handleApiEvent: function (eventtype, payload) {
    if (payload && payload.crud && payload.category === 'team' && payload.instance === app.team.id) {
        payload.crud.forEach(function (item) {
            var type = item.type,
                model,
                collection;
            
            if (type === 'update') {
                logger.log('got an update', item);
                model = app.findModel(item.object, item.id);
                if (model) {
                    model.set(item.data);
                }
                if (eventtype === 'shipTask' && item.object === 'task') {
                     app.team.shippedTasks.add(item.data);
                }
                
            } else if (type === 'create' && item.object === 'member') {
                logger.log('got a create');
                collection = app.findCollection(item.object);
                if (collection) {
                    collection.add(item.data);
                }
            } else if (type === 'delete') {
                logger.log('got a delete');
                model = app.findModel(item.object, item.id);
                if (model) {
                        model.collection.remove(model);
                }
            }
        });
    }
}
```

This is all that is required to keep all of our models in sync with what's going on, in the rest of the team.

This is due to the fact that each event type that involves a modification of state for the team you're logged in as will include a "crud" attribute. 

A typical event will look something like this:

```json
{
  "action": {
    "when": "1352711565839",
    "presence": "offline",
    "who": "4"
  },
  "category": "team",
  "instance": "1",
  "crud": [
    {
      "id": "4",
      "type": "update",
      "object": "member",
      "data": {
        "presence": "offline"
      }
    }
  ],
  "eventNumber": 19277
}
```

There is some various meta data, etc. But for our purposes, the only thing we care about are the "crud" portions of the event. As you probably suspected, "crud" stands for "Create", "Read", "Update" and "Delete". A given action, by another member of your team could result in multiple data changes that you need to make in order to keep your app in sync. 

Crud events have the following attributes:
1. type - "create", "update", "delete"
2. object - The type of object being referred to. For example "task", "member", or "team".
3. id - The id of the object being referred to.
4. data - The new properties in the cases of "create" and "update".

Now for each incoming event we loop through each item in the "crud" array and handle it. 

For "create" we use the object type to look up the corresponding backbone collection and simply add the contents of the "data" attribute to the collection. Assuming your app is configured properly this will create a model and add it to the collection. 

For "delete" we use the object type and ID to look up the model and remove it from its collection.

For "update" we use the object type and ID to look up the model and then simply call "set" on that model with the contents of the "data" attributes.

Since by default you are subscribed to each team that you're on, you'll notice we do a bit of checking to make sure that payload "category" and "instance" attributes match the team we're trying to update. Another quirk for our particular usecase here is that we only care about "shipped" tasks. But, shipping a task is not a "create" as far as the API is concerned because the task already existed. So we have a special case inside our "update" to handle event type "shipTask" as a "create" event.

But, the neat thing is, that with these twenty-some lines of code any changes that any other team members make will be reflected in our local model state.

## Bam!
Hope you had fun and learned something. Now you know how to build a simple, but very useful realtime, singlepage app in backbone.js.

I'm @HenrikJoreteg on twitter. Be sure to ping me on twitter with any thoughts, questions or feedback. I'd love to hear from you.

## Resources
- Dashboard application source code: [https://github.com/HenrikJoreteg/dotnet-dashboard.andbang.com](https://github.com/HenrikJoreteg/dotnet-dashboard.andbang.com)
- Hosted version of this app: (tbd)
- Backbone.js docs: [http://backbonejs.org](http://backbonejs.org)
- Andbang API docs: [https://docs.andbang.com](https://docs.andbang.com)
- Sugar.js Date docs: [http://sugarjs.com/dates](http://sugarjs.com/dates)
- andyet.com blog posts related to JS: [http://blog.andyet.com/tag/javascript/](http://blog.andyet.com/tag/javascript/)