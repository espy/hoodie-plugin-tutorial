# Very Important:

__This document describes functionality and features that don't exist yet.__ Much of it is dreamcode and represents the ideal state we want to work towards. Significant parts might change in the near future. However, feel free to file issues if you have any suggestions or ideas about the plugins API and tasks. 

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

All you need to write a Hoodie plugin is a running Hoodie app. Your plugin lives directly in the app's `node_modules` directory and must be referenced in its `package.json`, just like any other npm module. You don't have to register and maintain it as an npm module once it is complete, but we'd like to be able to use npm's infrastructure for finding and installing Hoodie plugins, so we'd also like to encourage you to use it as well. We'll explain how to do this at the end of this document.

### The Hoodie Architecture

If you haven't seen it yet, now is a good time to browse through [the explanation of the Hoodie stack](http://hood.ie/intro.html), and how it differs from what you might be used to. One of Hoodie's core strengths is that it is offline first, which means the application (and therefore your plugin) should work regardless of the user's connection status. We do this by not letting the frontend send tasks to the backend directly. Instead, the frontend deposits tasks in the database, which, you might remember, is both local and remote and syncs whenever it can. These tasks are then picked up by the backend, which acts upon these tasks. When completed, the database emits corresponding events, which can then in turn be acted upon by the frontend. For this, we provide you with a Plugin API, which handles generating and managing these tasks, as well as a number of other things you'll probably want to do a lot of, like writing stuff to user databases and so on.

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

### How Will this Work?

Here's what we want the Hoodie app to be able to do with the plugin, which we'll call `direct-messages`:

* Logged in users can send a direct message to any other logged in user
* Recipient users will see a new message appear in near real time

In the frontend, we need:

* a `directMessage.send()` method in the Hoodie API to add a task that sends the message
* a `directMessage.on()` method to listen for events that are fired, for example when a new message appears in the recipient account

In the backend, we need to:

1. check that the recipient exists
2. save the new message to the recipient's database
3. mark the original task as completed
4. if anything goes wrong, update the task accordingly

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
* `package.json` contains the plugin's metadata and dependencies

Let's look at all four in turn:

#### The Direct Messaging Plugin's Frontend Component

This is where you write any extensions to the client side hoodie object, should you require them. In the same way you can do

    hoodie.store.add('zebra', {"name": "Ricardo"});

from the browser in any Hoodie app, you can use the plugin's frontend component to expose something like

    hoodie.directMessages.send({
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
      function send( messageData ) {
        var defer = hoodie.defer();

        hoodie.task.add('direct-message', messageData)
        .done( function(message) {
          hoodie.task.on('remove:direct-message:'+message.id, defer.resolve);
          hoodie.task.on('error:direct-message:'+message.id, defer.reject);
        })
        .fail( defer.reject );

        return defer.promise();
      };

      function on( eventName, callback ) {
        hoodie.task.on( eventName + ':direct-message', callback);
      };

      hoodie.directMessages = {
        send: send,
        on: on
      };
    });

Let's go through this line by line:

    Hoodie.extend(function(hoodie) {

Here we extend the hoodie object we use in the browser and also pass a reference to that object back in, so your frontend component can actually use the rest of the Hoodie API.

    function send( messageData ) {

Here's our first API method: sending a private message. This method  requires an object with each message's data, just like in the example above. It could require all sorts of things though, after all, it's your plugin, not ours :) Note that it won't actually be available for use in the Hoodie API at this point, but we'll get to that later.

    var defer = hoodie.defer();

Now it gets a little tricky. We want your plugin API to be able to handle promises, such as

    hoodie.directMessages.add( messageData )
        .then( onMessageSent, onMessageError );

`hoodie.defer()` basically gives you the promises that were chained behind the actual API call, so they don't get lost anywhere and you can call them later. Remember, you're building an API that might get used by people other than yourself, and for consistency, it would be nice if it also worked with promises, just like the rest of the Hoodie frontend API. Let's look at the next line:

    hoodie.task.add('direct-message', messageData)
    .done( function(messageTask) {
      hoodie.task.on('remove:direct-message:'+messageTask.id, defer.resolve);
      hoodie.task.on('error:direct-message:'+messageTask.id, defer.reject);
    })
    .fail( defer.reject );

The is a big one, but if you've used Hoodie before, it will look familiar. We're adding a new task and passing it a type `direct-message`, as well as the payload from the `hoodie.directMessage.send()` call. If this succeeds, we register two event listeners, one for the removal of the task, which we'll do once the plugin's backend component has completed it, and a second one for when something goes wrong and the backend returns an error. `messageTask` is simply the task object that gets returned when `hoodie.task.add()` succeeds.

__Note:__ the `hoodie.task.on()` listener accepts three different object selectors after the event type, just like `hoodie.store.on` does in the hoodie.js frontend library:

* none, which means any object type: `'success'`
* a specific object type: `'success:direct-message'`
* a specific individual object `'success:direct-message:a1b2c3'`

The latter is what we're doing in the current line: listening for the `success` and `error events of the specific `direct-message` object with the id of the relevant message task. We pass through the promises that were attached to the original API call to handle the events (`defer.resolve` corresponds to `onMessageSent`, `defer.reject` to `onMessageError`).

##### An Important Note on Task Events

There's a crucial detail about task event types and where they can occur. __The frontend component__ can listen for:

`add`, `change`, `success` and `error`

The __backend component__, however, can only listen for:

`add` and `change`

It can _call_ `task.success()` and `task.error()` and through this _emit_ these events, but it can't _listen_ for them. This makes sense, because the backend component is where the tasks succeed or fail, so it already always knows when that happens. It's just the frontend component that has to be notified of these events. However, tasks can be added and changed on both sides, which is why both sides can listen for these events. If you're doing something that can succeed or fail on the client side, you don't need tasks to do it.

While we're at it, there's a second backend limitation concerning events: __you can't listen for events from a specific individual object in the backend component of a plugin.__ This will work in the frontend component, but not in the back:

    task.on('success:direct-message:a1b2c3', defer.resolve);

Allowing this behaviour in the backend would generate potentially enormous amounts of callbacks sitting in memory on the server and possibly never being called and removed. We're thinking that the backend components, which are essentially node.js workers, shouldn't be storing state like this anyway. That's what we've got the whole task system for in the first place. So, ideally, this limitation shouldn't be a problem for you.

But for now, back to the original example:

Lastly, if the `task.add()` fails outright before it even reaches the database, we also call `defer.reject` from the `fail()` promise of the `add()` method.

Then comes the final part of the `send()` method:

    return defer.promise();

Which simply passes through the original API call's entire promise.

In order to listen for incoming messages, we also expose an `on()` method, with which we can subscribe to events related to `direct-message` tasks.

    function on( eventName, callback ) {
      hoodie.task.on( eventName + ':direct-message', callback);
    };

Since the whole `extend()` construct is essentially a module, we'll have to explicitly make our API methods publically available so they can actually be called from the outside, and that's what happens at the very end:

    hoodie.directMessages = {
      send: send,
      on: on
    };

Now `hoodie.directMessages.send()` and `hoodie.directMessages.on()` actually exist. If you've ever seen the revealing module pattern, you know what this is.

That's your frontend component dealt with! Remember, your plugin can consist of only this component, should you just want to encapsulate some more complex abstract frontend code in some convenience functions, for example.

But we have lots of ground to cover, so onward! to the second part:

#### The Direct Messaging Plugin's Backend Component

For reference while reading along, here's the [current state of the Plugin API](https://github.com/hoodiehq/hoodie-plugins-api/blob/master/README.md) as documented on gitHub.

By default, the backend component lives inside a `index.js` file in your plugin's root directory. It can be left there for simplicity, but Hoodie will prefer the following, if present:

* Whatever you reference under `main` in the plugin's `package.json`
* Whatever you get when you `require()` the plugin root directory

We didn't want to be too opinionated here.

__First things first__: this component will be written in node.js, and node in general tends to be in favor of callbacks and opposed to promises. We respect that and want everyone to feel at home on their turf, which is why all of our backend code is stylistically quite different from the frontend code.

Let's look at the whole thing first:

    module.exports = function(hoodie) {
      hoodie.task.on('add:direct-message', handleNewMessage);

      function handleNewMessage(originDb, message) {
        var recipient = message.to;

        hoodie.account.find('user', recipient, function(error, user) {
          if (error) {
            return hoodie.task.error(originDb, message, error);
          };

          var targetDb = "user/" + user.ownerHash;
          hoodie.database(targetDb).add('message', message, addMessageCallback);
          hoodie.task.success(originDb, message, handleError);
        });
      };

      function addMessageCallback(error, object) {};
    };

Again, let's go through line by line.

    module.exports = function(hoodie) {

Essentially a boilerplate container for the actual backend component code. Again, we're passing the hoodie object so we can use the API inside the component.

    hoodie.task.on('add:direct-message', handleNewMessage);

Remember when we did `hoodie.task.add('direct-message', messageData)` in the frontend component? This is the corresponding part of the backend, listening to the event emitted by the `task.add()`. We call `handleNewMessage()` when it gets fired:

    function handleNewMessage(originDb, message) {

Now we're getting into databases. Remember: every user in Hoodie has their own isolated database, and `task.on()` passes through the name of the database where the event originated.

    var recipient = message.to;
    hoodie.account.find('user', recipient, function(error, user) {

We also need to find the recipient's database, so we can write the message to it. Our `hoodie.directMessages.send()` took a message object with a `to`key for the recipient, and that's what were using here. We're assuming that users are adressing each other by their actual Hoodie usernames and not some other name.

    if (error) {
      return hoodie.task.error(originDb, message, error);
    };

The sender may have made a mistake and the recipient may not exist. In this case, we call `task.error()` and pass in the message and the error so we can deal with the problem where neccessary. Remember, this will emit an event that you can listen for both in the front- _and/or_ backend with `task.on()`. In our case, we were just passing them through our plugin's frontend component to let the app author deal with it. Internally, Hoodie knows which task the error refers to through the `message` object and its unique id.

    var targetDb = "user/" + user.ownerHash;

We still haven't got the recipient's database, which is what we do here. In CouchDB, database names consist of a type prefix (in this case: `user`), a slash, and an id. We'd recommend using Futon to find out what individual objects and databases are called. Now we get to the main point:

    hoodie.database(targetDb).add('message', message, addMessageCallback);

This works a lot like adding an object with the Hoodie frontend API, except we use callbacks instead of promises here. We've added the message data as a `message` object in the recipient's database, and if we're listening to the corresponding `new` event in the frontend, we can make it show up in near realtime.

__Note__: You'll probably be thinking: "wait a second, what if another plugin generates `message` objects too?" and that's very prescient of you. We're not dealing with namespacing here for simplicity's sake, but prefixing your object type names with your plugin name seems like an excellent idea. In that case, this line should read

    hoodie.database(targetDb).add('direct-messages-message', message, addMessageCallback);

Right, we're nearly there, we just have to clean up after ourselves:

    hoodie.task.success(originDb, message, handleError);

Again, Hoodie knows which task `success` refers to through the `message`object and the unique id therein. Once you've called `success` on a task, it will be marked as deleted, and the frontend component, which is listening for `'remove:direct-message:'+message.id'`, will trigger the original API call's success promise. The task's life cycle is complete.

#### Additional Notes on the Frontend Component and Application Frontend

All that's left to do now is display the message in the recipient user's app view as soon as it is saved to their database. In the application's frontend code, we'd just

    hoodie.store.on('add:message'), function(messageObject){
      // Show the new message somewhere
    });

Really basic Hoodie stuff. You can also call `hoodie.store.findAll('message').done(displayAllMessages)`or any of the other `hoodie.store` methods to work with the new messages objects.

As a plugin author, you could wrap your own methods around these, so app authors can stay within your plugin API's scope even when listening for native Hoodie store events or using the Hoodie core API. For example, in the plugin's frontend component, have:

    function findAll(){
      var defer = hoodie.defer();
      hoodie.store.findAll('message')
      .done(defer.resolve)
      .fail(defer.reject)

      return defer.promise();
    }

    hoodie.directMessages = {
      send: send,
      findAll: findAll,
      on: on
    };

This could then be called by the app author as

    hoodie.directMessages.findAll().done(displayAllMessages)

Now you know how to create and complete tasks, make your plugin promise-friendly, emit and listen for task events, build up your frontend API, write objects to user databases and a couple of other things. I'd say your're well on your way. All other features of the plugin API are waiting in the docs section for you to discover them.

There's more, though: we can build an admin panel for the `direct-messages` plugin.

#### Extending Pocket with your Plugin's own Admin Panel

For this example, let's have an admin panel which

* can send users direct messages
* has a configurable config setting for maximum message length (because it's working for Twitter, why shouldn't it work for us?)

To do this, you must provide a `/pocket` directory in your plugin's root directory, and this should contain a small HTML page, or, more precisely, an HTML fragment. This means an HTML document without `<html>`, `<head>` or `<body>` tags. You can have `<script>`and `<link>` tags in there to load JS and CSS, and Hoodie will automatically inject three more things for your convenience:

1. The hoodie.js frontend library (and jQuery)
2. A special API called hoodie.admin.js
3. Pocket's CSS stylesheet

Let's start with the easy bit:

##### Styling your Plugin's Admin Panel

As noted, your admin panel will have Pocket's styles applied by default. Pocket is built with Bootstrap (currently 2.3.2), so all plugin developers can rely on a set of components they know will be sensibly styled by default. You're completely free to override these styles, should you want to do something spectacular.

We're working on a helper plugin called `hoodie-plugin-elements` which is simply a page filled with all elements and components from Bootstrap, in the Pocket style. We'll be using it to test our CSS, plugin developers can use it as a copy and paste snippet library for quickly building their own plugin backends without having to worry about consistency and styling.

##### Sending Messages from Pocket

Pocket has a special version of Hoodie, called HoodieAdmin. It offers several APIs by default, like `hoodieAdmin.signIn(password)`, `hoodie.users.findAll`, and [more](https://github.com/hoodiehq/hoodie.admin.js).

It can be extended just like the standard Hoodie library:

    HoodieAdmin.extend(function(hoodieAdmin) {
      function send( messageData ) {
        var defer = hoodieAdmin.defer();

        hoodieAdmin.task.add('direct-message', messageData)
        .done( function(message) {
          hoodieAdmin.task.on('remove:direct-message:'+message.id, defer.resolve);
          hoodieAdmin.task.on('error:direct-message:'+message.id, defer.reject);
        })
        .fail( defer.reject );

        return defer.promise();
      };

      function on( eventName, callback ) {
        hoodieAdmin.task.on( eventName + ':direct-message', callback);
      };

      hoodieAdmin.directMessages = {
        send: send,
        on: on
      };
    });

Now `hoodie.directMessages.send` can be used the same way by the admin in pocket as it can be used by the users of the app. The only difference is that other users cannot send messages to the admin, as it's a special kind of account.

##### Getting and Setting Plugin Configurations

To get / set a plugin's config, you can use `hoodieAdmin.plugin.getConfig('direct-messages')` & `hoodieAdmin.plugin.updateConfig('direct-messages', config)`. `config` is a simple object with key/value settings.

For example, let's say you'd like to limit the message length to 140 characters. You'd build a corresponding form in your `pocket/index.html` with an input for a number (let's say 140), and bind this to the submit event:

    hoodieAdmin.plugin.updateConfig('direct-messages',
      { maxLength: valueFromInputField }
    );

Then in the backend, you could check for the setting and reject messages that are longer:

    if (message.body.length > hoodie.config('maxLength')) {
      var error = {
        error: 'invalid',
        message: 'Message is too long (hoodie.config('maxLength') + ' characters maximum).'
      };
      return hoodie.task.error(originDb, message, error);
    }

#### The package.json

The package.json is required by node.js. For our plugin, it will look like this:

    {
      "name": "hoodie-plugin-direct-messages",
      "description": "Allows users to send direct messages to each other",
      "version": "1.0.0",
      "main": "worker.js"
    }

#### Deploying your Plugin to NPM

It's as simple as `npm publish`.


#### Installing your plugin:

Inside your Hoodie application, simply run `npm install hoodie-plugin-direct-messages`.
