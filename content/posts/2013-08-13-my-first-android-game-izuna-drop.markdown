---
author: mustafa91
comments: true
date: 2013-08-13 21:33:01+00:00
layout: post
slug: my-first-android-game-izuna-drop
title: 'My First Android Game: Izuna Drop'
---

As a part of my CS-319 Object-Oriented Software Engineering course,  I developed a computer game with Nail Akıncı and Naime Nur Çadırcı, called Izuna Drop. It is a simple space shooter clone. As the design was more imporant in that course, the implementation was not very efficient, it was written on Java, and due to our bugs, it requried approximately 1 GB of memory. If you wonder what that looked like, it is avaialable on [GitHub/Izuna](https://github.com/mustafaakin/Izuna).

In this summer, I bought a Turkish book, [Android Oyun Programlama](http://www.kitapyurdu.com/kitap/default.asp?id=629585)(Android Game Programming) It focuses on [AndEngine](https://github.com/nicolasgramlich/AndEngine) which is a Java Game engine for Android. Book is nice, but it focuses on GLES1 version of AndEngine, which is now outdated and it is hard to find support, since the GLES2 version of AndEngine has many changes over the old one. However, it is a nice book to teach you the basics, it is not rocket science to make the transition to GLES2 on your own.

Anyways, I have read the book and started to implement few simple scenes. The biggest problem in game development is that you cannot find graphical assets easily, which is the essential part. I know a little bit about designing, but it is not easy to design nice game assets. But Nail Akıncı, my team member has developed himself on modeling and animation and provided us nicely hand-made assets. (But if you Google enough, you can find very nice free 2D assets for your game). So, I have decided to re-implement our game for Android. It took almost a month, working on mostly nights during this hot summer. AndEngine simplifies many things that we had manually take care of on our PC version, so the Android version is much more faster and has small memory footprint.  It's source can be also viewed at [GitHub/izuna-android](http://github.com/mustafaakin/izuna-android). It is not perfect, for instance collisions are not pixel perfect, but overall it is a playable and enjoyable game with 5 levels and each level consisting 10 unique waves.

Here are some screenshots from the Izuna Drop:

[![s1](http://mustafaakin.files.wordpress.com/2013/08/s1.png?w=187)](http://mustafaakin.files.wordpress.com/2013/08/s1.png) [![s2](http://mustafaakin.files.wordpress.com/2013/08/s2.png?w=187)](http://mustafaakin.files.wordpress.com/2013/08/s2.png) [![s3](http://mustafaakin.files.wordpress.com/2013/08/s3.png?w=187)](http://mustafaakin.files.wordpress.com/2013/08/s3.png)



I would be glad if you could download it and test it:

[![get-it-on-google-play-store-logo](http://mustafaakin.files.wordpress.com/2013/08/get-it-on-google-play-store-logo.png?w=300)](https://play.google.com/store/apps/details?id=in.mustafaak.izuna)

Thank you!


