---
author: mustafa91
comments: true
date: 2013-11-06 22:11:21+00:00
layout: post
slug: comparing-expressjs-session-stores
title: Comparing ExpressJS Session Stores
---

For my next project, I was exploring the session storages for ExpressJS, and I found [a question on StackOverflow, ](http://stackoverflow.com/questions/8749907/what-is-a-good-session-store-for-a-single-host-node-js-production-app)but I was frustrated with the accepted and 30 upvoted answer, which compares local in memory session store with remote session stores. The answer is very misleading, there is no chance that remote storage can beat local storage, even if the in-memory solution does O(n^9) calculations, it would be faster than going to web.

So I have decided to carry my own test on my local machine, with local installations of Express and MongoDB. The results were interesting. I have tested a simple server that does the following:
    
    
```javascript
    app.get("/", function(req,res){
        if ( req.session && req.session.no){
            req.session.no = req.session.no + 1;
        } else {
            req.session.no = 1;
        }
        res.send("No: " + req.session.no);
    });
````

The results for concurrency levels 1,10,100,500 are:
    

    concurrency: 1
    none       4484.86 [#/sec] 
    memory     2144.15 [#/sec] 
    redis      1891.96 [#/sec] 
    mongo      710.85  [#/sec] 

    Concurrency: 10
    none       5737.21 [#/sec] 
    memory     3336.45 [#/sec] 
    redis      3164.84 [#/sec] 
    mongo      1783.65 [#/sec] 

    Concurrency: 100
    none       5500.41 [#/sec] 
    memory     3274.33 [#/sec] 
    redis      3269.49 [#/sec] 
    mongo      2416.72 [#/sec] 

    Concurrency: 500
    none       5008.14 [#/sec] 
    memory     3137.93 [#/sec] 
    redis      3122.37 [#/sec] 
    mongo      2258.21 [#/sec]

The source code of this is available on the GitHub: [https://github.com/mustafaakin/express-session-store-benchmark](https://github.com/mustafaakin/express-session-store-benchmark)

I hope it will help you to choose a better session storage for your needs. I will go with Redis, because it is almost as fast as in-memory solution, whereas MongoDB s performance cripples down.
