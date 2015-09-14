---
layout: post
title: Tutorial - Data Channels with WebRTC in Node.js
categories:
- blog
---

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
channel. Then a peer can click "Send Data," which will send some text over to the other peer.

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

# User Interface Functionality

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


{% highlight javascript %}
{% endhighlight %}
{% highlight javascript %}
{% endhighlight %}
