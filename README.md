
# node-crawler
[![npm package](https://nodei.co/npm/crawler.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/crawler/)

[![build status](https://secure.travis-ci.org/bda-research/node-crawler.png)](https://travis-ci.org/bda-research/node-crawler)
[![Dependency Status](https://david-dm.org/bda-research/node-crawler/status.svg)](https://david-dm.org/bda-research/node-crawler)

Best crawling/scraping package for Node, 1.0.0 is released, happy hacking :)

Features:
 * server-side DOM & automatic jQuery insertion with Cheerio (default) or JSDOM
 * Configurable pool size and retries
 * Control rate limit
 * Priority queue of requests
 * forceUTF8 mode to let crawler deal for you with charset detection and conversion

Here is the [CHANGELOG](https://github.com/bda-research/node-crawler/blob/master/CHANGELOG.md)

# How to install


    $ npm install crawler

# Crash course


```javascript
var Crawler = require("crawler");
var url = require('url');

var c = new Crawler({
    maxConnections : 10,
    // This will be called for each crawled page
    callback : function (error, result, $) {
        // $ is Cheerio by default
        //a lean implementation of core jQuery designed specifically for the server
		if(error){
			console.log(error);
		}else{
			console.log($("title").text());
		}
    }
});

// Queue just one URL, with default callback
c.queue('http://www.amazon.com');

// Queue a list of URLs
c.queue(['http://www.google.com/','http://www.yahoo.com']);

// Queue URLs with custom callbacks & parameters
c.queue([{
    uri: 'http://parishackers.org/',
    jQuery: false,

    // The global callback won't be called
    callback: function (error, result) {
		if(error){
			console.log(error);
		}else{
			console.log('Grabbed', result.body.length, 'bytes');
		}
    }
}]);

// Queue some HTML code directly without grabbing (mostly for tests)
c.queue([{
    html: '<p>This is a <strong>test</strong></p>'
}]);
```

# Work with `bottleneck`

Control rate limits for each connection, usually used with proxy.

```javascript
var Crawler = require("crawler");

var c = new Crawler({
    maxConnections : 3,
    rateLimits:2000,
    callback : function (error, result, $) {
		if(error){
			console.error(error);
		}else{
			console.log($('title').text());
		}
    }
});

c.queue({
    uri:"http://www.google.com",
    limiter:"key1",// for connection of 'key1'
    proxy:"http://user:pass@127.0.0.1:8080"
});

c.queue({
    uri:"http://www.google.com",
    limiter:"key2", // for connection of 'key2'
    proxy:"http://user:pass@127.0.0.1:8082"
});

c.queue({
    uri:"http://www.google.com",
    limiter:"key3", // for connection of 'key3'
    proxy:"http://user:pass@127.0.0.1:8081"
});

```

# Options reference


You can pass these options to the Crawler() constructor if you want them to be global or as
items in the queue() calls if you want them to be specific to that item (overwriting global options)

This options list is a strict superset of [mikeal's request options](https://github.com/mikeal/request#requestoptions-callback) and will be directly passed to
the request() method.

Basic request options:

 * `uri`: String, the URL you want to crawl
 * `timeout` : Number, in milliseconds        (Default 15000)
 * [All mikeal's requests options are accepted](https://github.com/mikeal/request#requestoptions-callback)

Callbacks:

 * `callback(error, result, $, done)`: Function that will be excuted after a request was completed
     * `error`: Error Messages (Default null)
     * `result`: Result of this task
         * `result.statusCode`: HTTP status code,`200` for example
         * `result.body`: HTTP response content,`<html><body>content</body></html>` for example
         * `result.headers`: HTTP response headers
         * `result.request`: Detail request information
             * `result.request.uri`: HTTP request entity of parsed url,[click](https://nodejs.org/api/url.html#url_url_strings_and_url_objects) for detail description
             * `result.request.method`: HTTP request method,`GET` for example
             * `result.request.headers`: HTTP request headers
         * `result.options`: [Options](#options-reference) of this task
     * `$`: DOM selector of result html page, [click](https://api.jquery.com/category/selectors/) for detail description
     * `done`: Function that must be called when you complete your task

Pool options:

 * `maxConnections`: Number, Size of the worker pool (Default 10),
 * `priorityRange`: Number, Range of acceptable priorities starting from 0 (Default 10),
 * `priority`: Number, Priority of this request (Default 5),

Retry options:

 * `retries`: Number of retries if the request fails (Default 3),
 * `retryTimeout`: Number of milliseconds to wait before retrying (Default 10000),

Server-side DOM options:

 * `jQuery`: true, false, "whacko" or ConfObject (Default true). Crawler will use 

Charset encoding:

 * `forceUTF8`: Boolean, if true will get charset from HTTP headers or meta tag in html and convert it to UTF8 if necessary. Never worry about encoding anymore! (Default true),
 * `incomingEncoding`: String, with forceUTF8: true to set encoding manually (Default null)
     `incomingEncoding : 'windows-1255'` for example

Cache:

 * `skipDuplicates`: Boolean, if true skips URIs that were already crawled, without even calling callback() (Default false). This is not recommended, it's better to handle outside `Crawler` use [seenreq](https://github.com/mike442144/seenreq)

Other:
 * `rotateUA`: Boolean, if true, `userAgent` should be an array, and rotate it (Default false) 
 * `userAgent`: String or Array, if `rotateUA` is false, but `userAgent` is array, will use first one. 
 * `referer`: String, if truthy sets the HTTP referer header
 * `rateLimits`: Number of milliseconds to delay between each requests (Default 0) 


 
# Class:Crawler

## Event: 'limiterChange'
 * `options` [Options](#options-reference)
 * `limiter` String

Emitted when limiter has been changed.

## Event: 'request'
 * `options` [Options](#options-reference)

Emitted when crawler is ready to send a request.

If you are going to modify options at last stage before requesting, just listen on it.

```
crawler.on('request',function(options){
    options.qs.timestamp = new Date().getTime();
});
```

## Event: 'drain'

Emitted when queue is empty.

```
crawler.on('drain',function(){
    // For example, release a connection to database.
    db.end();// close connection to MySQL
});
```

## crawler.queue(uri|options)
 * `uri` String
 * `options` [Options](#options-reference)

Enqueue a task and wait for it to be excuted.

## crawler.queueSize
 * Number

Size of queue, read-only

 
# Working with Cheerio or JSDOM


Crawler by default use [Cheerio](https://github.com/cheeriojs/cheerio) instead of [Jsdom](https://github.com/tmpvar/jsdom). Jsdom is more robust but can be hard to install (espacially on windows) because of [contextify](https://github.com/tmpvar/jsdom#contextify).
Which is why, if you want to use jsdom you will have to build it, and `require('jsdom')` in your own script before passing it to crawler. This is to avoid cheerio crawler user to build jsdom when installing crawler.

## Working with Cheerio
```javascript
jQuery: true //(default)
//OR
jQuery: 'cheerio'
//OR
jQuery: {
    name: 'cheerio',
    options: {
        normalizeWhitespace: true,
        xmlMode: true
    }
}
```
These parsing options are taken directly from [htmlparser2](https://github.com/fb55/htmlparser2/wiki/Parser-options), therefore any options that can be used in `htmlparser2` are valid in cheerio as well. The default options are:

```js
{
    normalizeWhitespace: false,
    xmlMode: false,
    decodeEntities: true
}
```

For a full list of options and their effects, see [this](https://github.com/fb55/DomHandler) and
[htmlparser2's options](https://github.com/fb55/htmlparser2/wiki/Parser-options).
[source](https://github.com/cheeriojs/cheerio#loading)

## Working with JSDOM

In order to work with JSDOM you will have to install it in your project folder `npm install jsdom`, deal with [compiling C++](https://github.com/tmpvar/jsdom#contextify) and pass it to crawler.
```javascript
var jsdom = require('jsdom');
var Crawler = require('crawler');

var c = new Crawler({
    jQuery: jsdom
});
```

# How to test



## Install and run Httpbin

crawler use a local httpbin for testing purpose. You can install httpbin as a library from PyPI and run it as a WSGI app. For example, using Gunicorn:

    $ pip install httpbin
    // launch httpbin as a daemon with 6 worker on localhost
    $ gunicorn httpbin:app -b 127.0.0.1:8000 -w 6 --daemon

    // Finally
    $ npm install && npm test

## Alternative: Docker

After [installing Docker](http://docs.docker.com/), you can run:

    // Builds the local test environment
    $ docker build -t node-crawler .

    // Runs tests
    $ docker run node-crawler sh -c "gunicorn httpbin:app -b 127.0.0.1:8000 -w 6 --daemon && cd /usr/local/lib/node_modules/crawler && npm install && npm test"

    // You can also ssh into the container for easier debugging
    $ docker run -i -t node-crawler bash

    
[![build status](https://secure.travis-ci.org/bda-research/node-crawler.png)](https://travis-ci.org/bda-research/node-crawler)

# Rough todolist

 * Introducing zombie to deal with page with complex ajax
 * Refactoring the code to be more maintenable, it's spaghetti code in there !
 * Proxy feature
 * This issue: https://github.com/sylvinus/node-crawler/issues/118
 * Make Sizzle tests pass (jsdom bug? https://github.com/tmpvar/jsdom/issues#issue/81)
 * More crawling tests
 * Document the API more (+ the result object)


# ChangeLog

See [CHANGELOG](https://github.com/bda-research/node-crawler/blob/master/CHANGELOG.md)
