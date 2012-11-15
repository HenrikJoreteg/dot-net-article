# Fast, precompiled client-side templates

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

From my rather unscientific tests, this was roughly 6-10 times faster than mustache.js and certainly resulted in faster more cacheable page loads.