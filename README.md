express-brute
=============
[![Build Status](https://travis-ci.org/AdamPflug/express-brute.png?branch=master)](https://travis-ci.org/AdamPflug/express-brute)
[![NPM version](https://badge.fury.io/js/express-brute.png)](http://badge.fury.io/js/express-brute)

A brute-force protection middleware for express routes that rate limits incoming requests, increaseing the delay with each request in a fibonacci-like sequence.

Installation
------------
  via npm:

      $ npm install express-brute

A Simple Example
----------------
``` js
var ExpressBrute = require('express-brute'),

var store = new ExpressBrute.MemoryStore(); // stores state locally, don't use this in production
var bruteforce = new ExpressBrute(store);

app.post('/auth',
	bruteforce.prevent, // error 403 if we hit this route too often
	function (req, res, next) {
		res.send('Success!');
	}
);
```

Options
-------
### ExpressBrute(store, options)
- `store` An instance of `ExpressBrute.MemoryStore` or `ExpressBrute.MemcachedStore`
- `options`
	- `freeRetries`  The number of retires the user has before they need to start waiting (default: 2)
	- `minWait`      The initial wait time (in milliseconds) after the user runs out of retries (default: 500 milliseconds)
	- `maxWait`      The maximum amount of time (in milliseconds) between requests the user needs to wait (default: 15 minutes). The wait for a given request is determined by adding the time the user needed to wait for the previous two requests.
	- `lifetime`     The length of time (in seconds since the last request) to remember the number of requests that have been made by an IP. By default it will be set to `maxWait * the number of attempts before you hit maxWait` to discourage simply waiting for the lifetime to expire before resuming an attack. With default values this is about 6 hours.
	- `failCallback` gets called with (`req`, `resp`, `next`, `nextValidRequestDate`) when a request is rejected (default: ExpressBrute.FailForbidden)

### ExpressBrute.MemcachedStore(hosts, options)
- `hosts` Memcached servers locations, can by string, array, or hash.
- `options`
	- `prefix`       An optional prefix for each memcache key, in case you are sharing 
	                 your memcached servers with something generating its own keys.
	- ...            The rest of the options will be passed directly to the node-memcached constructor.

For details see [node-memcached](http://github.com/3rd-Eden/node-memcached).

Instance Methods
----------------
- `protect(req, res, next)` Middleware that will bounce requests that happen faster than
                            the current wait time by calling `failCallback`. Equivilent to `getMiddleware(null)`
- `getMiddleware(key)`      Generates middleware that will bounce requests with the same `key` and IP address
                            that happen faster than the current wait time by calling `failCallback`.
                            `key` can be a string. Alternatively, key can be a `function(req, res, next)`
                            that returns a string or calls `next`, passing a string as the first parameter.
                            Also attaches a function at `req.brute.reset` that can be called to reset the
                            counter for the current ip and key. This functions the the `reset` instance method,
                            but without the need to explicitly pass the `ip` and `key` paramters
- `reset(ip, key, next)`    Resets the wait time between requests back to its initial value. You can pass `null`
                            for `key` if you want to reset a request protected by `protect`.

Built-in Failure Callbacks
---------------------------
There are some built-in callbacks that come with BruteExpress that handle some common use cases.
		- `ExpressBrute.FailForbidden` Terminates the request and responds with a 403 and json error message
		- `ExpressBrute.FailMark` Sets res.nextValidRequestDate and the res.status=403, then calls next() to pass the request on to the appropriate routes

A More Complex Example
----------------------
``` js
require('connect-flash');
var ExpressBrute = require('express-brute'),
	moment = require('moment'),
    store;

if (config.environment == 'development'){
	store = new ExpressBrute.MemoryStore(); // stores state locally, don't use this in production
} else {
	// stores state with memcached
	store = new ExpressBrute.MemcachedStore(['127.0.0.1'], {
		prefix: 'NoConflicts'
	});
}

var failCallback = function (req, res, next, nextValidRequestDate) {
	req.flash('error', "You've made too many failed attempts in a short period of time, please try again "+moment(nextValidRequestDate).fromNow());
	res.redirect('/login'); // brute force protection triggered, send them back to the login page
};
var userBruteforce = new ExpressBrute(store, {
	freeRetries: 5,
	winWait: 5*60*1000, // 5 minutes
	maxWait: 60*60*1000, // 1 hour,
	failCallback: failCallback
});
// No more than 1000 login attempts per day per IP
var globalBruteforce = new ExpressBrute(store, {
	freeRetries: 1000,
	winWait: 25*60*60*1000, // 1 day 1 hour (should never reach this wait time)
	maxWait: 25*60*60*1000, // 1 day 1 hour (should never reach this wait time)
	lifetime: 24*60*60*1000, // 1 day
	failCallback: failCallback
});

app.post('/auth',
	globalBruteforce.prevent,
	userBruteforce.getMiddleware(function(req, res, next) {
		// prevent too many attempts for the same username
		return req.body.username;
	}),
	function (req, res, next) {
		if (User.isValidLogin(req.body.username, req.body.password)) { // omitted for the sake of conciseness
		 	// reset the failure counter so next time they log in they get 5 tries again before the delays kick in
			req.brute.reset(function () {
				res.redirect('/'); // logged in, send them to the home page
			});
		} else {
			res.flash('error', "Invalid username or password")
			res.redirect('/login'); // bad username/password, send them back to the login page
		}
	}
);
```

Changelog
---------
### v0.3.0
* NEW: Support for using custom keys to group requests further (e.g. grouping login requests by username)
* NEW: Support for middleware from multiple instances of `BruteForce` on the same route.
* NEW: Tracking `lifetime` now has a reasonable default derived from the other settings for that instance of `ExpressBrute`
* NEW: Keys are now hashed before saving to a store, to prevent really long key names and reduce the possibility of collisions.
* CHANGED: Tracking `lifetime` is now specified on `ExpressBrute` instead of `MemcachedStore`. This also means lifetime is now supported by MemoryStore.
* IMPROVED: Efficiency for large values of `freeRetries`.
* BUG: Removed a small chance of incorrectly triggering brute force protection.