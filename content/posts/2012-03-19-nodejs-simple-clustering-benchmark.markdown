---
author: mustafa91
comments: true
date: 2012-03-19 18:29:21+00:00
layout: post
slug: nodejs-simple-clustering-benchmark
title: NodeJS Simple Clustering Benchmark
---

If you are interested in node, you know that NodeJS uses an event driven I/O model and it is single threaded, so it is not meant to be working multi-core. However, by using the clustering support, you can bump your applications speed. I will give a a plain hello example, however consider that you will never send just hello world to anyone in the world without any background processing like database connections, cache connections, file operations or just simply I/O.

The server configuration is: Intel(R) Xeon(R) CPU  E5606 @ 2.13GHz (8 Cores), 16 GB RAM.


## Clustering Support with 8 instances of Node

```javascript
var express = require('express');
var cluster = require("cluster");
var os = require("os");

var app = module.exports = express.createServer();

app.get('/', function(req,res){
    res.send("Hello World");
});

if (cluster.isMaster) {
    console.log("CPUS: " + os.cpus().length);
    for (var i = 0; i < os.cpus().length / 2; i++) {
        var worker = cluster.fork();
    }
} else {
    app.listen(3000);
}
```

## Without any Clustering Support

```javascript
var express = require('express');
var os = require("os");

var app = module.exports = express.createServer();

app.get('/', function(req,res){
    res.send("Hello World");
});

app.listen(3000);
```

I have used ab to test the following system with 100.000 reuests at concurrency level 1000.

* **With Clustering:** 11612.01 requets / sec
* **No Clustering:** 3497.04 requests / sec

A little tweak can be helpful to increase your througput. It will utilize all the cores of the server, if you seperate your databse and server (you should), it would absolutely give a better result.
