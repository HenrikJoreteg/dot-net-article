# .net article - Build a realtime web app with Backbone.js for voting on where to go to lunch

1. Introduction
    1. "realtime"
        1. What does it mean in context of the web?
        1. Maintaining "state"
            1. what is it?
            1. avoiding duplication
            1. Interfaces as slaves to app state
            1. Binding state all the way to the server
    1. tools
        1. Backbone.js
            1. Why backbone, versus alternatives
            1. What does it give us
        1. Node.js
        1. Socket.io
        1. Stitch
        1. Jade / Templatizer
            1. Doing fast clientside templating
        1. And Bang API 
            1. Gives us a server-side
    1. What we're going to build: "Food Fight!" - a realtime voting app to help a team know where to go for lunch
        1. Since we're focusing on the browser code we'll use And Bang as an exiting realtime api.
1. The structure of the client-side code
    1. CommonJS modules
    1. Linting standands
1. The structure of our server-side code
1. Our model files
1. Rending the basic app layout
    1. Binding model state to the view
    1. Modifying model state in console to see view updates
1. Getting the data
    1. Setting up our connection 
    1. handling initial state
    1. handling updates
1. Running it
1. Wrapping it up
    1. learning more
    1. links to resources/projects
    1. other realtime mechanisms
    1. Follow me on twitter