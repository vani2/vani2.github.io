---
categories: network ios
date: "2018-01-04T14:04:00Z"
title: Resumable file upload

---

![Header image](/assets/2018-01-04/header.jpg)

In one of our mobile apps client wants to upload a big size file to a server. The main requirement was to resume upload after connection lost. Also if a connection is poor, it can be interrupted by timeout.

<!--more-->


Let’s begin with Apple docs for [NSURLSession](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/URLLoadingSystem/Articles/UsingNSURLSession.html). It says “Upload tasks send data in the form of a file, and support background uploads while the app is not running“.

That’s it we are looking for! But, if you dive in, you get that in background mode only download task resume is guaranteed. So, after connection lost our upload task start the upload process from the beginning. That’s not what we are really wanted.

After some discussions with a backend team, we consider using [this protocol](https://github.com/pgaertig/nginx-big-upload).

Shortly about how it works:

1. Take a file chunk equal to chunk size.
2. In request, a header sends from to end byte of current chunk and full size of the file.
3. Also, send SHA-1 of data in a request header.
4. Receive session token and CRC32 of current data of all chunks, that server has already.
5. After successful upload check control sums.

We also cache the progress of upload for every file and its additional info: session token, file URL, etc.

The disadvantage is you should implement this on your own. The algorithm is straightforward — recursive call to take the next chunk and checking control sum.
