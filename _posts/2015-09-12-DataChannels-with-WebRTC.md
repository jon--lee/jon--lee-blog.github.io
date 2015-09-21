---
layout: post
title: Tutorial - Data Channels with WebRTC in Node.js
categories:
- blog
---

(IN PROGRESS)

So, you're interested in working with data channels and WebRTC? Well so am I!
Hopefully you already have an understanding of what WebRTC is and some if its cool
features. Maybe you've even built a simple video streaming application. In this tutorial,
    I want to go over how to use data channels to share text and files.

The reason I'm writing this is that I had a hard time finding a solid, comprehensive
guide for data channels with WebRTC. However, there's a great tutorial on building a video chat application out there on the [Twilio
Blog by Phil Nash.](https://www.twilio.com/blog/2014/12/set-phasers-to-stunturn-getting-started-with-webrtc-using-node-js-socket-io-and-twilios-nat-traversal-service.html)
I definitely recommend it if you're interested in video streaming or just want a basic understanding
of WebRTC. Phil provides great explanations, walks you through the signalling process, and introduces STUN/TURN so that 
your application can go live.

In this guide, I want to borrow from and spring off of Phil's post so that together we can all understand
the data channel as it is a little different from video streaming.

---

# A Brief History of Real-Time

The RTC of WebRTC stands for real-time communications, which is more of a blanket term for sharing information (video, audio, files) instantaneously.
The difference between the old RTC and the new RTC is that the former often required a lot of overhead such as plugin-dependency.
The mission of WebRTC is to eliminate this need for a "server middle-man" and standardize communications across all platforms.

Google released WebRTC as an open source project a few years ago and it's already becoming a standard. I've heard that Facebook uses WebRTC somewhere
in the Messenger system.


# First steps

By the end of this tutorial we will have:

- set up a Node.js application locally with express.
- implemented a signalling system with websockets.
- established a connection between peers.
- created a data channel.
- shared some text and data via the data channel.

You may have noticed the word "websockets" and wondered why we need websockets when we can just use WebRTC.
One thing that you should keep in mind is that we actually do need servers to facilitate WebRTC.
For example, **presence detection** and **signalling** both involve sending information
between to and from a server before we can establish our peer connection and data channel. Websockets tend
to be pretty straightforward for these tasks.

Again I'd just like to note that I borrow a lot from [Phil Nash's Twilio Blog post](https://www.twilio.com/blog/2014/12/set-phasers-to-stunturn-getting-started-with-webrtc-using-node-js-socket-io-and-twilios-nat-traversal-service.html), 
so credit goes to him for an awesome guide. A lot of code at the beginning will take directly from his post.

## Dependencies

There are some Node tools that we want to get to make our lives easier.

- [Express](http://expressjs.com/) as our web framework so we can serve the pages and 
"abstract away" all the nitty gritty details.
- [Socket.io](http://socket.io) as our solution to presence detection and signalling.

You can use other application frameworks and communication channels, but I find these to be fairly intuitive and fun.

## Set up

Make a new directory called "channels" and and initialize a Node application there. Fill out the information for which `npm init` prompts.
If you don't know the answers to some prompts, just press enter. It doesn't really matter.

{% highlight bash %}
$ mkdir channels
$ cd channels
$ npm init
{% endhighlight %}

You'll end up with a file `package.json` in your directory that should look like kind of like this

{% highlight json %}
{
  "name": "channels",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
{% endhighlight %}

Now we want to install our dependencies so just run

{% highlight bash %}
$ npm install express socket.io --save
{% endhighlight %}

You'll see a directory called `node_modules` which contains the dependencies
and if you look in `package.json` you should now see an addition `dependencies` element.

{% highlight json %}
{
    ...
    "dependencies": {
        "express": "^4.13.3",
        "socket.io": "^1.3.6"
    }
}
{% endhighlight %}

Let's make some files in our application. We'll start with the backend. Remember how you're `package.json`
has `"main": "index.js"`? Now we have to make that file.

{% highlight bash %}
$ touch index.js
{% endhighlight %}

Now let's create a directory called `public` which will contain all our front end files.

{% highlight bash %}
$ mkdir public
$ cd public
$ touch app.js index.html
{% endhighlight %}

We'll also need an `adapter.js`, which is a ["shim to insulate apps from spec changes and prefix differences"](https://github.com/webrtc/adapter)
You can get the file by running

{% highlight bash %}
$ curl https://webrtc.googlecode.com/svn-history/r4259/trunk/samples/js/base/adapter.js \
        > adapter.js
{% endhighlight %}

Don't worry too much about the adapter. You won't have to edit it directly.

To recap, you're file structure should look like this
{% highlight bash %}
channels/
    node_modules/
        express/
        socket.io/
    package.json
    index.js
    public/
        adapter.js
        app.js
        index.html
{% endhighlight %}

The first file that we're going to edit is the `public/index.html` just to set up the interface.
For now it's nothing fancy.

{% highlight html %}
<!-- public/index.html -->
<html>
    <head>
        <title>Data Channel Tutorial</title>
    </head>
    <body>
        <h1>Data Channel</h1>
        <button id='ready'>Ready</button>
        <button id='connect' disabled>Connect</button>
        <button id='send-data' disabled>Send Data</button>
        
        <script src='/socket.io/socket.io.js'></script>
        <script src='/adapter.js'></script>    
        <script src='/app.js'></script>
    </body>

</html>
{% endhighlight %}

If you open this raw HTML file, you'll find a header and three buttons. Essentially what will happen here is that
when the user clicks the "Ready" button, they'll make their presence noticed by the server. When we have two
peers who are noticed by the server, the server will enable either to click "Connect," which will establish the data
channel. Then a peer can click "Send Data," which will send some text over to the other peer. "Connect" and "Send Data" are
disabled for now since the peer will go through stages to establish a data channel.

We want this to be running from Express, so we should probably set that up now. Open up `index.js`
and add the following lines of code.

{% highlight javascript %}
// index.js
var express = require('express');
var app = express();
var http = requir('http').Server(app);

var socketio = require('socket.io');
var io = socketio(http);
{% endhighlight %}

We've just imported the Node modules here and created instances of Express and Socket.io.
Now we want to serve our `index.html` when a GET request is made to the homepage. We also
want to tell our app where to find static files.

{% highlight javascript %}
// index.js
...
app.get('/', function(res, req){
    res.sendFile(__dirname + '/public/index.html');
});
app.use(express.static(__dirname + '/public'));

{% endhighlight %}

Then we must add the code that will actually start the server when we run this file. You can
change the port number if you would like.

{% highlight javascript %}
// index.js
...
port = process.env.PORT || 5000;
http.listen(port, function() {
    console.log("running on http://localhost:" + port);
});
{% endhighlight %}

Let's get this server running locally by running the following command in the `channels` directory
{% highlight bash %}
$ node index.js
{% endhighlight %}

You should see a printed statement that says `running on http://localhost:5000`.
Now if you point your browser to `http://localhost:5000/`, you should see that same HTML page. Nice!

# User interface functionality

If you tried to click those buttons, you would realize that they don't do anything. Let's
give this some functionaliy. Just as Phil Nash did in his guide, we will create a Channel object
inside `public/app.js` that will act as a comprehensive collection of functions and variables. We'll 
add our buttons to that object and give them some event listeners, which will correspond to functions
already in the object

{% highlight javascript %}
// public/app.js
var Channel = {
    getReady: function(){
        console.log('ready button pressed!');
    },
    establishConnection: function(){
        console.log('connect button pressed!');
    },
    sendData: function(){
        console.log('send data button pressed!');
    }
};
Channel.readyButton = document.getElementById('ready');
Channel.readyButton.addEventListener('click', Channel.getReady, false);

Channel.connectButton = document.getElementById('connect');
Channel.connectButton.addEventListener('click', Channel.establishConnection, false);

Channel.sendButton = document.getElementById('send-data');
Channel.sendButton.addEventListener('click', Channel.sendData, false);
{% endhighlight %}

If you load the page again, and click the ready button, you should see that little message
in the console. You can verify that the other buttons are also working by taking out the `disabled`
attribute and reloading the page.

# Sending messages to the server

Displaying a message in the console isn't all that exciting. We really want the ready button
to notify the server that this peer is present and ready to set up a data channel. This process
can be broken down into three tasks:

- Send a ready message to the server over a socket.
- Get a notification that the server recognizes us.
- Get a notification that the other peer is ready and recognized as well.

Step 1 will be handled on the client side. We just need to add the following code to connect
and cause the server to fire an event when we're ready. This means that we need to call `io()` which
connects over websockets to the server. Then we `emit` a message to join a "room." Don't worry too much
if you're not familiar with how Socket.io works. You can think of it as the client joining a chat room 
called "room" and now the client can send messages to room; however, in our case, these messages will be received
by the server and the server will fire event handlers upon reception and send messages back.

{% highlight javascript %}
// public/app.js
var Channel = {
    socket: io(),
    ...
    getReady: function(){
        Channel.socket.emit('join', 'room');       
    },
    ...
};
...
{% endhighlight %}

Let's head back over to the server so we can receive this message.

{% highlight javascript %}
// index.js
...
io.on('connection', function(socket){
    socket.on('join', function(room){
        // It's empty now. We'll implement it in just a second.
    });    
});
{% endhighlight %}

The new lines are basically the receptors of the message. When we called `io()` on the client side we fired the first line `io.on('connection'...);`
which sets us up for handling receptions of subsequent messages.

The second line defines our handler for a 'join' message, which is what we are sending in the 
`getReady` function. `join` has a parameter `room` which is just the name of the room that I talked
about earlier. You may want to use different room names if you have lots of users.

{% highlight javascript %}
// index.js
...
io.on('connection', function(socket){
    socket.on('join', function(room){
        var clients = io.sockets.adapter.rooms[room];
        var numClients = (typeof clients !== 'undefined') ? Object.keys(clients).length : 0;
        if (numClients == 0){
            socket.join(room);
            socket.emit('join', room);
        }
        else if (numClients == 1) {
            socket.join(room);
            socket.emit('join', room);
            socket.emit('ready', room);
            socket.broadcast.emit('ready', room);
        }
        else {
            socket.emit('full', room);
        }
    });    
});
{% endhighlight %}

Whoa! A lot happened right there! This is drawn straight from Phil Nash's article.
I don't really want to go too into the weeds here, but the gist is that we're
getting the number of clients connected to `room`. If there no one else is in the room,
we have to wait for the other peer, but we can notify the client that we've connected
successfully. If there is another client in the room, we can tell the new client that they've
joined and we can tell both clients that we're ready for the next phase.

If there are already two clients there, then we want to let the new client know that
the room is filled :(

# Receiving messages on the client side

I mentioned that the server is going to send messages back to the client. The client needs some
functionality to handle those messages. But first we need to tell the client what kinds
of messages we're expecting. Remember that the server could send back `'join'`, `'ready'` or `'full'`.
Just before we emit to the server, we should add handlers to the socket.

{% highlight javascript %}
// public/app.js
var Channel = {
    ...
    getReady: function(){
        Channel.socket.on('join', Channel.onJoin);
        Channel.socket.on('ready', Channel.onReady);
        Channel.socket.on('full', Channel.onFull);
        Channel.socket.emit('join', 'room');
    },
    ...
};
...
{% endhighlight %}

Of course, we have to implement `onJoin`, `onReady`, and `onFull`. For now we will just spit some output
to the console to ensure that everything is working.

{% highlight javascript %}
// public/app.js
var Channel = {
    ...
    onJoin: function(){
        console.log('server says we have joined!');
    },
    onReady: function(){
        console.log('server says it is ready!');
    },
    onFull: function(){
        console.log('boo :( room is full');
    }
};
...
{% endhighlight %}

Restart your server and point your browser to the localhost. If you click the ready button
you should see a message in your browser console that says it has joined.

While keeping this tab open, open a new tab and point it to the same localhost. Click the ready
button again and you should see an additional message in each tab's console that says the server
is ready.

If you open a third tab and do the same, the message should say that the room is full.

Below is a screenshot showing the ready message:

![Ready example](/ready.png)

And one showing the full message:

![Full example](/full.png)

### [Checkpoint 1](https://github.com/jon--lee/webrtc-datachannel-post/tree/3d250cc8dfd35dadf6b48331496220e23a9c3a0d)
Just to ensure that you are on the right track, I posted a [checkpoint on github](https://github.com/jon--lee/webrtc-datachannel-post/tree/3d250cc8dfd35dadf6b48331496220e23a9c3a0d).
It contains the source code up to this point in case you are stuck.

We want to disable the option of clicking "Ready" again and enable the option of clicking
"Connect" when the server is ready. Replace the console messages with the following:

{% highlight javascript %}
// public/app.js
var Channel = {
    ...
    onJoin: function(){
        Channel.readyButton.setAttribute('disabled', 'disabled');        
    },
    onReady: function(){
        Channel.connectButton.removeAttribute('disabled');
    },
    ...
};
...
{% endhighlight %}





{% highlight javascript %}
{% endhighlight %}
{% highlight javascript %}
{% endhighlight %}
{% highlight javascript %}
{% endhighlight %}


