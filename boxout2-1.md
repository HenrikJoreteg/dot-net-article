# Smart "leave-in-able" clientside logging

Sometimes it'd be really nice to be able to leave log statements in your production code. It's nice to be able to just `console.log("my important thing")` and not worry about polluting your client's log output or worse, in the case of older browsers, downright crashing your app because you accidentally left one and `console` doesn't exist.

Tada! `andlog` to the rescue. Most clientside logging libraries somehow overwrite or modify console object. But, that sucks because often that means you end up logging out the `arguments` object, which is less pretty than the output you get by calling `console.log` directly. So `andlog` creates a dummy console object with normal console methods, but *only* if it finds a "truthy" value in `localStorage.debug`. So, as a result, you can leave log statements in your code, use the native console object when available and still only see the log output if you've explicitly turned it "on" by setting `localStorage.debug = true`.

So you can just do:

```js
var logger = require('andlog');

logger.log('Hello');
```

And it will only log if you've set the flag in localStorage and the browser has a console. And, in addition, it's safe to leave in production.

More info on the readme: [https://github.com/HenrikJoreteg/andlog](https://github.com/HenrikJoreteg/andlog)