---
author: mustafa91
comments: true
date: 2012-04-27 23:31:51+00:00
layout: post
title: WSTP - File Transver Over Websockets
---

If you were following my blog, you may seem that I have once tried to make WS-FTP project with Java, however the web-sockets were on draft and it was changing quickly and they were not safe. Meantime, I have learned and mastered NodeJS and met Socket.io, and I decided to use them, since they were providing a better interface and it was much more easier to code a web server with NodeJS rather than Java.

The project I have made is very simple and not optimized. I quickly wanted to demonstrate the concept and will try to explain the workflow now:

1. NodeJS server is listening on port 3000. It both provides socket.io interface and a web-interface for administration (not implemented yet, but that's the idea)
2. Chrome Extension (Client) requests unlimited stroage and ability to connect any site, so connection can be established.
3. Client connects to the server. Asks for any arbitrary folder contents, with filenames and size.
4. Client wants to download a file asks for block 0 of file. And depending on size, it adds N requests to the work queue.
5. Server just opens the file, reads the requested block. In my app, block sizes are 8 KBs.
6. Whenever a block is gathered on client, it constructs a BLOB object and saves it in memory.
7. After consturction of BLOB, client pops 2 tasks from queue and wants them.
8. This processes continues until all the blocks are fetched. Client saves the data to Chrome's sandboxed filesystem, in which user can download the file to local filesystem.


As you  can see, it is very primitive, only 1 connection and 1 transfer at a time, and if you install it it will not have a very nice GUI, however, it will certainly be a starting point. Currently there are some issues:

* It's single threaded and upon recieving a packet from a socket, Chrome can hang depending on file size.
* Only 1 connection, 1 transfer at a time.	
* No nice GUI, but it will about to come!
* File saving, handling in memory is not efficient with this way, needs a better algorithm, some temproary disk access.

Additionally, as you can imagine, this can be asily turned into a P2P system. Bascially, we download blocks and there is no reason to not downlaod blocks from multiple clients. I have made this project available on GitHub, so anyone can easily access it and freely use it and maybe extend it!

[https://github.com/mustafaakin/wsftp](https://github.com/mustafaakin/wsftp)