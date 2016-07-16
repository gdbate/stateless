# Stateless #

A mostly unopinionated MVC framework for Node.js and Vue.js designed for portability.

**Note:** This is still ES5 to maximize compatibility, I will port it in a few months.

## Requirements

You must have a **firebase** account, as well as an **Auth0** account for managing secured areas.

## Information ##

Stateless utilizes **Firebase** to manage data related to states, settings, sessions and user specific information. This gives it an element of portability. It is setup with **Webpack** and **Vue** for real-time data updates and single file components. **Auth0** is also required for secured areas.

### Why is this not an NPM?

On top of being a server library that does most of the heavy lifting, Stateless requires developing in a specific way to hook up controllers, enable webpack, allow for customization. The goal is to make an NPM library that does most of the work but there will always be a template for development.

## Setup

### Initialization ###

You might notice the code isn't here yet :) it is coming soon.

```javascript
var Stateless = require('./stateless'),
	stateless = new Stateless();
```

### Configure Website Structure ###

You must send an object with a page structure. This supports any level of hierarchy.

```json
[
  {
    "index": "home",
    "title": "Home",
    "path": "/",
    "nav": true
  },
  {
    "index": "settings",
    "title": "Settings",
    "path": "/settings",
    "nav": true
  }
]
```

```javascript
stateless.pages(__dirname + '/pages.json');
```

This maps all the pages to Vue components, and takes care of Vue-router and express routing. If you have a page with children, a 'pages' property will be available in the page components data object, allowing you to draw navigation for child pages.

**Note:** this is designed to be a website micro service, so entire sites will use authentication or not.

### Configure Firebase ###

You must create a service account for your firebase database. This is for storing session data and updating users in real-time. 

Obtain a service account credentials JSON under **Permissions -> Service Account** and the other values under "**project settings**" (click "add Firebase to your web app).

```javascript
var options = {
	serviceAccountCredentials: __dirname + '/firebase-credentials.json',
	apiKey: 'api-key',
	authDomain: 'myaccount-#####.firebaseapp.com',
	databaseURL: 'https://myaccount-#####.firebaseio.com'
};
stateless.firebase(options);
```

These Firebase rules are required for security:

```json
{
  "rules": {
    "sessions": {
      ".read": "false",
      ".write": "false"
    },
    "states": {
      ".read": "auth != null",
      ".write": "false"
    },
    "users": {
			"$uid": {
        "client": {
          ".read": "$uid === auth.uid",
          ".write": "$uid === auth.uid"
        },
        "server": {
          ".read": "$uid === auth.uid",
          ".write": "false"
        }
      }
    }
  }
}
```

### Configure Auth0 ###

Auth0 is a great authentication service that is extremely fair priced. You can make it work social network connections, internal database users, and apply things like two-factor authentication or logging in as a user easily. When you setup the client for this site, also add a **Firebase addon**. Pull out the information from the Firebase JSON credentials file (including the email address) and enter it into Auth0.

```javascript
var options = {
	domain: 'mydomain.auth0.com',
	clientId: 'XXXXXXXXXXXXXXXXXXXXXX',
	clientSecret: 'XXXXXXXXXXXXXXXXXXXXXXXX_-XXXXXXXXXXXXXXXXXX'
};
stateless.auth0(options);
``` 

### Configure Reusable Vue Components ###

If you use Vue reusable components, You must describe them so they get included by webpack.

```javascript
stateless.vue('component', ['form', 'line', 'markdown', 'table', 'tabs']);
stateless.vue('field', ['boolean', 'checkbox', 'email', 'list', 'markdown', 'number', 'password', 'radio', 'select', 'text', 'textarea', 'toggle', 'url', 'date']);
stateless.vue('cell', ['hidden', 'template', 'field', 'currency', 'if-null', 'boolean', 'number']);
```

**Note:** Currently They must be prefixed/categorized. The .vue single file components will be found in the named folders, as well as referenced like this:

```html
<component-line></component-line>
```

### Configure Web Server ###

```javascript
var options = {
	host: '127.0.0.1',
	port: 8080,
	jsPort: 9201, //also package.json "npm run frontend"
	sessionSecret: 'chicken',
	urlLogout: '/'
};
stateless.http(options);
```

---

## Anatomy of a Controller

The bulk of your business logic and data handling should be contained to simple controllers. They are meant to be accessed on request from the client or used inside the application.  Controllers are categorized by a name. Under each name you can chose to have subfolders if you wish.  **Recommended:** Name your controllers after the data model the data belongs to (or the primary query table). This will ensure a good natural organization.

The parameters of a controller is always a linear name->value object. This ensures the best access by code, http/ajax requests and socket requests.

### Simple Controller

```javascript
;(function(){
	module.exports = function(cb, params){
		cb(null, 'hello world');
	}
}());
```

**Note:** The first variable is always an error object (if applicable).

### Controller Calling Another

You can call controllers from others. The controller method is available in every controllers scope:

```javascript
;(function(){
	module.exports = function(cb, params){
		this.controller('message', 'hello-world', function(err, message){
			cb(null, message + ' ' + params.name);
		});
	}
}());
```

### From The Main App

You can call controllers from others. The controller method is available in every controllers scope:

```javascript
stateless.controller('message');
```

## Appending to Controller Scope

Stateless strives to be as unopinionated as possible. One way is allowing you to use any libraries you want.

### Appending

```javascript
function date(){
	var y=new Date().getFullYear().toString();
	var m=(new Date().getMonth()+1).toString();
	var d=new Date().getDate().toString();
	return y+'-'+(m[1]?m:'0'+m[0])+'-'+(d[1]?d:'0'+d[0]);
}
stateless.scope('date', date);
```

### Reading Scope

```javascript
	var ymd = this.date();
```

**Note:** This is optional you can also require dependencies inside a controller.

---

## State Management

Stateless weirdly is very good at managing states. Everything from site-wide variables to user specific information.

### Mapping to Controllers

When a server starts it must know how to populate state information to send to Firebase (and every user). You must provide a name of a state, name of a controller, and the path. Optionally you can specify parameters for the controller.

```
stateless.stateMap('countries', 'country', 'list');
```

Now the users will have a list of countries always available to them.

### Accessing on the client

Accessing the (real-time) information in a Vue component is easy. In the `<template>`:

```html
<select>
	<option v-for="(iso, country) in countries" track-by="$index" :value="iso">{{ country }}</option>
</select>
```

And in the `<script>`:

```javascript
  export default {
    data: function(){return{
      countries: Store.countries
    }}
  };
```

### Updating Manually

Sometimes you might not want to map a state name to a controller. This is how you manually specify the value:

```
//inside a controller:
this.stateSet('restart', this.datetime());

//in the main app code:
stateless.stateSet('restart', stateless.datetime());
```

## Event Handling

You can hook into events for logging, error handling purposes.

### Types of events

* **start** when the HTTP server starts running
* **request** when a user requests a path (there should only be 1 per user)
* **authenticated** when a user logs in or signs up
* **error** when stateless encounters an error

### Watching for an Event

```javascript
stateless.on('err', function(err){
	console.log('darn it all ', err.message);
})
```
*Better documentation coming soon*

### Triggering an Event 

You may want to trigger an event yourself so you have one source for events:

```javascript
//inside a controller:
this.trigger('error', err);

//in the main app code:
stateless.trigger('error', err);
```

## Logging

### User Specific


## Todo

* Support for state lists (chat messages)
* Custom events
