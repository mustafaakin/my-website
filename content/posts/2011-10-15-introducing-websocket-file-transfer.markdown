---
author: mustafa91
comments: true
date: 2011-10-15 23:10:07+00:00
layout: post
slug: introducing-websocket-file-transfer
title: Introducing WebSocket File Transfer
---

First of all, all my work below is in draft form, and not optimized, even a little bit. I have performance issues, but I just want to represent the capabilities of this new technology. I am neither IETF or any other organization that can define a protocol, but I hope this article would inspire some people.

## HTML5 Magic

As you might not know, the HTML 5 specification offers [File API](http://www.w3.org/TR/FileAPI/) for manipulating both binary and text files on client side. Your imagination is the only limitation for the capabilities of this technology. You can have a look at a detailed [tutorial](http://www.html5rocks.com/en/tutorials/file/filesystem/) at the HTML5Rocks website. Remember, the files are saved to a sandbox area, not in real file system.

HTML5 also offers a new interface called [WebSockets API](http://dev.w3.org/html5/websockets/), that allows you to open a socket to the webserver and remain opened until server or you close it and provides you a two-way communication. This saves a lot of time, when you have to save time. Because XHR requests just closes after the response is received, so a great time is saved when you do not have to re-open a TCP connection. WebSockets offer both binary and text transfer. However, only Google Chrome 15 Beta version implements binary conversion. One ugly fact, a single message cannot exceed 32KB, or I cannot, Chrome throws an error that frame size is too big.

## Platforms
 
Anyways, with these two technologies, I had a great idea, I could implement a simple file server. The server is implemented with Java, but actually what it does is listening to a user defined port and respond on each request, defined by specification. I implemented the server using [Webbit server](https://github.com/joewalnes/webbit).
 

Of course, writing data to client, opening sockets to each computer in the world is not allowed by any browser. However, Google Chrome has an [application model](http://code.google.com/chrome/apps/) where you can request user permissions to do more. So I could open sockets to each server on the world and use unlimited storage. Implementation was JavaScript on the client side, which had a little performance issues, but a great scripting language.

## Server / Client Interaction

  1. Client opens a web socket to the server. 
  2. Server accepts it.   
  3. Client sends authentication information.
  4. If user is accepted file list is sent to user as a string message.
  5. User wants information about file X.
  6. Server sends how many pieces does file have.
  7. Client requests File X, piece 0.
  8. Server sends first, 32 KB of data as binary. Until 4 MB is reached, server keeps sending 32 KB chunks of data.   
  9. Server sends OK after first message with piece number.
  10. Client saves file as 4 MB to disk, free the BLOB from memory. 
  11. Go to step 7, with piece + 1 and repeat until the last piece is achieved.   
  12. After all pieces received, read all pieces 1 by 1 and append them to the final file.   
  13. Close the connection.

## Results

This works! Really! But not with very large files. Because, some of you may imagine, there are concurrency problems. Also, the binary data received from server is not marked with piece and chunk number. I thought, I was using local, the packets should come in the order I send them, but this is not the case. If I try to fetch a ~250MB file, It fails. But I could successfully transfer a 30 MB file, inside my computer. However, it is not suitable for real-life client-server model. It could even fail in a LAN. 


My point was to prove that HTML 5 can be used to create a FTP-like client. File API and WebSockets API provides a lot of opportunities. What I have done is not a distributable project, yet. But I have [a page on Google Project](http://code.google.com/p/wsftp/). I am just learning SVN, so please forgive me for any inconvenience ![Smile](http://mustafaakin.files.wordpress.com/2011/10/wlemoticon-smile.png)

## Screenshot
 
[![image](http://mustafaakin.files.wordpress.com/2011/10/image_thumb.png)](http://mustafaakin.files.wordpress.com/2011/10/image.png)

[![image](http://mustafaakin.files.wordpress.com/2011/10/image_thumb1.png)](http://mustafaakin.files.wordpress.com/2011/10/image1.png)