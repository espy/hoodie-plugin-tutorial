# Building plugins for Hoodie

## Introduction

Hoodie is a small core that handles data storage, sync and authentication. Everything else is a plugin that can be added to this core. Our goal is to make Hoodie as extensible as possible, while keeping the core tiny.

### What is a Hoodie plugin?

Hoodie plugins have three distinct parts, and you will need at least one of them. They are:

- __A frontend component__ that extends the Hoodie API, written in Javascript
- __A backend component__, written in node.js
- __An admin view__, which is an HTML fragment with associated styles and JS code that appears in Pocket, your Hoodie app's admin panel

### What can a Hoodie plugin do?

In short, anything Hoodie can do. A plugin can work in Hoodie's Node.js backend and manipulate the database or talk to other services, it can extend the Hoodie frontend library's API, and it can appear in Pocket, the admin panel each Hoodie app has, and extend that with new stats, functions and whatever else you can think of.

### Example plugins

You could…

- log special events and send out emails to yourself whenever something catastrophic/wonderful happens
- have node resize any images uploaded to your app, generate a couple of thumbnail versions, save them to S3 and reference them in your database
- securely authenticate with github (or any other service, really) and send data back and forth
- extend Hoodie so signed-up users can send direct messages to each other

## Prerequisites

All you need to write a Hoodie plugin is a running Hoodie app. Your plugin lives directly in the app's `node_modules` directory and must be referenced in its `package.json`, just like any other npm module. You don't have to register and maintain it as an npm module once it is complete, but we'd like to be able to use npm's infrastructure for finding and installing Hoodie plugins, so we'd also like to encourage you to use it as well. See further down for how this needs to look exactly.

### The Hoodie Architecture

If you haven't seen it yet, now is a good time to browse through the explanation of the Hoodie stack, and how it differs from what you might be used to. One of Hoodie's core strengths is that it is offline first, which means the application (and therefore your plugin) should work regardless of the user's connection status. We do this by not letting the frontend send tasks to the backend directly. Instead, the frontend deposits tasks in the database, which, you might remember, is both local and remote and syncs whenever it can. These tasks are then picked up by the backend, which acts upon these tasks. When completed, the database emits corresponding events, which can then in turn be acted upon by the frontend. For this, we provide you with a Plugin API, which handles generating and managing these tasks, as well as a number of other things you'll probably want to do a lot of, like writing stuff to user databases and so on.

### The Plugin API and Tasks

Currently, the only way to get the backend component of a plugin to do anything is with a task. A task is a slightly special object that can be saved into the database from the Hoodie frontend. Your plugin's backend component can listen to the events emitted when a task appears, and then do whatever it is you want it to do. You could create a task to send a private message in the frontend, for example:

    hoodie.task.add("message", {
        "to": "Ricardo",
        "body": "Hello there! How are things? We're hurtling through space! Wish you were here :)"
    });

And in your backend component, listen for that task appearing and act upon it:

    hoodie.task.on('new:message', function (dbName, task) {
        // generate a message object from the change data and
        // store it in both users' databases
    });

But we're getting ahead of ourselves. Let's do this properly and start at the beginning.

## Let's Build a Direct Messaging Plugin

### Where to Start

Any plugins you write live in the `node_modules` directory of your application, with their directory name adhering to the following syntax:

    [your_application]/node_modules/hoodie-plugin-[plugin-name]

So, for example:

    supermessenger/node_modules/hoodie-plugin-direct-messages

Everything related to your plugin goes in there.

### Structuring a Plugin

As stated, your plugin can consist of up to three components: __frontend__, __backend__ and __pocket__. Since it is also ideally a fully qualified npm module, we also require a `package.json` with some information about the plugin.

Assuming you've got all three components, your plugin's directory should look something like this:

    hoodie-plugin-direct-messages
        hoodie.direct-messages.js
        index.js
        /pocket
            index.html
            styles.css
            main.js
        package.json

* `hoodie.direct-messages.js` contains the frontend code
* `index.js` contains the backend code
* `/pocket` contains the admin view
* `package.json` contains the plugin's metadata and dependenceies

Let's look at all four in turn:

#### The Direct Messaging Plugin's Frontend Component

This is where you write any extensions to the client side hoodie object, should you require them. In the same way you can do

    hoodie.store.add('zebra', {"name": "Ricardo"});

from the browser in any Hoodie app, you can use the plugin's frontend component to expose something like

    hoodie.directMessages.add({
        "to": "Ricardo",
        "body": "One of your stripes is wonky"
    });

You've noticed I've used directMessages instead of our plugin's actual name "direct-messages", this is because, well, simply: I can. How and where you extend the hoodie object in the frontend is entirely up to you. Your plugin could even extend Hoodie in multiple places or override existing functionality.

All of your plugin's frontend code must live inside a file named according to the following convention:

    hoodie.[plugin_name].js

In our case, this would be

    hoodie.direct-messages.js

The code inside this is relatively straightforward:

    Hoodie.extend(function(hoodie) {
      function add( messageData ) {
        var defer = hoodie.defer();

        hoodie.task.add('direct-message', messageData)
        .done( function(message) {
          hoodie.task.on('remove:direct-message:'+message.id, defer.resolve)
          hoodie.task.on('error:direct-message:'+message.id, defer.reject)
        })
        .fail( defer.reject )

        return defer.promise()
      }

      function on( eventName, callback ) {
        hoodie.task.on( eventName + ':direct-message', callback)
      }

      hoodie.directMessages = {
        add: add,
        on: on
      };
    });

Let's go through this line by line:

    Hoodie.extend(function(hoodie) {

Here we extend the hoodie object we use in the browser and also pass a reference to that object back in, so your frontend component can actually use the rest of the Hoodie API.

    function add( messageData ) {

Here's our first API method: adding a private message. This method  requires an object with each message's data, just like in the example above. It could require all sorts of things though, after all, it's your plugin, not ours :) Note that it won't actually be available for use in the Hoodie API at this point, but we'll get to that later.

    var defer = hoodie.defer();

Now it gets a little tricky. We want your plugin API to be able to handle promises, such as

    hoodie.directMessages.add( messageData )
        .then( onMessageSent, onMessageError )

hoodie.defer() basically gives you the promises that were chained behind the actual API call, so they don't get lost anywhere and you can call them later. Remember, you're building an API that might get used by people other than yourself, and for consistency, it would be nice if it also worked with promises, just like the rest of the Hoodie frontend API. Let's look at the next line:

    hoodie.task.add('direct-message', messageData)
    .done( function(messageTask) {
      hoodie.task.on('remove:direct-message:'+messageTask.id, defer.resolve)
      hoodie.task.on('error:direct-message:'+messageTask.id, defer.reject)
    })
    .fail( defer.reject )

The is a big one, but if you've used Hoodie before, it will look familiar. We're adding a new task and passing it a type `direct-message`, as well as the payload from the `hoodie.directMessage.add()` call. If this succeeds, we register two event listeners, one for the removal of the task, which we'll do once the plugin's backend component has completed it, and a second one for when something goes wrong and the backend returns an error. `messageTask` is simply the task object that gets returned when `hoodie.task.add()` succeeds.

Note that the `hoodie.task.on()` listener accepts three different object selectors after the event type:

* none, which means any object type: `'remove'`
* a specific object type: `'remove:direct-message'`
* a specific individual object `'remove:direct-message:A1B2C3'`

The latter is what we're doing in the current line: listening for the remove and error events of the specific `direct-message` object with the id of the relevant message task. We pass through the promises that were attached to the original API call to handle the events (`defer.resolve` corresponds to `onMessageSent`, `defer.reject` to `onMessageError`).

Lastly, if the `task.add()` fails outright before it even reaches the database, we also call `defer.reject` from the `fail()` promise of the `add()` method.

Then comes the final part of the `add()` method:

    return defer.promise()

Which simply passes through the original API call's entire promise.

In order to listen for incoming messages, we also expose an `on()` method, with which we can subscribe to events related to `direct-message` tasks.

    function on( eventName, callback ) {
      hoodie.task.on( eventName + ':direct-message', callback)
    }

Since the whole `extend()` construct is essentially a module, we'll have to explicitly make our API methods publically available so they can actually be called from the outside, and that's what happens at the very end:

    hoodie.directMessages = {
      add: add,
      on: on
    };

Now `hoodie.directMessages.add()` and `hoodie.directMessages.on()` actually exist. If you've ever seen the revealing module pattern, you know what this is.

That's your frontend component dealt with! Remember, your plugin can consist of only this component, should you just want to encapsulate some more complex abstract frontend code in some convenience functions, for example.

But we have lots of ground to cover, so onward! to the second part:

#### The Direct Messaging Plugin's Backend Component

By default, the backend component lives inside a `index.js` file in your plugin's root directory. It can be left there for simplicity, but Hoodie will prefer the following, if present:

* Whatever you reference under `main` in the plugin's `package.json`
* Whatever you get when you `require()` the plugin root directory

We didn't want to be too opinionated here.

__First things first__: this component will be written in node.js, and node in general tends to be in favor of callbacks and opposed to promises. We respect that and want everyone to feel at home on their turf, which is why all of our backend code is stylistically quite different from the frontend code.

###

Use Gregor's backend code for this, once I've completely understood it:

https://gist.github.com/gr2m/6148091

###
