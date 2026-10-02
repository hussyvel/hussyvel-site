---
title: "Linux: Process, Socket, and Nginx"
date: 2026-10-01T09:20:00-03:00
draft: false
weight: 3
tags: ["linux", "nginx", "sockets", "processes", "series"]
categories: ["java"]
description: "How the Linux kernel turns a stream of bytes into something a server process can read, and why nginx sits in front of Java almost everywhere."
summary: "Before your Spring application sees anything, the kernel has already accepted a connection, handed it to a socket, and nginx has decided what to do with it."
ShowToc: true
---

Third post in the series — **[start from the hub]({{< ref "/java" >}})**. The [previous post]({{< ref "http-and-networking.md" >}}) ended with a TCP connection carrying an HTTP request arriving at a server. This post is about what happens the moment it lands on Linux, before a single line of Java code runs.

## A socket is a file

On Linux, almost everything is a file descriptor, and network connections are no exception. When a process wants to accept incoming connections, it:

1. Creates a **socket** (`socket()`) — gets a file descriptor back.
2. **Binds** it to an address and port (`bind()`) — e.g., `0.0.0.0:443`.
3. **Listens** on it (`listen()`) — tells the kernel to start queuing incoming connections.
4. **Accepts** connections (`accept()`) — each accepted connection is *another* file descriptor, distinct from the listening one.

This matters because it explains a very common confusion: the socket your server "listens" on and the socket it uses to talk to a specific client are not the same thing. One listening socket can spawn thousands of individual connection sockets, each a separate file descriptor the kernel tracks independently — read buffer, write buffer, TCP state, all of it.

## Why nginx sits in front of your Java app

In a typical SEMA-style deployment (and in most production Java setups), the process actually bound to port 443 is **not** the Java application — it's nginx. Your Spring Boot app, deployed to Tomcat, usually listens on something like `localhost:8080`, not reachable directly from the internet. Nginx sits in front and reverse-proxies to it:

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

This isn't bureaucracy for its own sake. Nginx is doing real work the JVM is comparatively bad at:

- **Terminating TLS** — the expensive handshake from the [previous post]({{< ref "http-and-networking.md" >}}) happens here, once, instead of inside the JVM.
- **Serving static files** directly, without waking up a Java thread for a `.css` file.
- **Buffering slow clients** — nginx can absorb a client trickling bytes in slowly, instead of tying up a Tomcat worker thread waiting on it.
- **Load balancing** across multiple backend instances, and acting as the single point for rate limiting, compression, and access logs.

From the JVM's perspective, every request it ever receives in this setup actually originated as a *new, separate connection from nginx*, not directly from the original client. That's why `X-Forwarded-For` and `X-Real-IP` headers exist — without them, your Java code would think every request came from `127.0.0.1`.

## Processes, not magic

Both nginx and the JVM running your Spring app are ordinary Linux processes — visible in `ps`, schedulable by the kernel, each with their own memory space, open file descriptor table, and set of threads. When nginx proxies a request, it's quite literally: read bytes from the file descriptor connected to the client, write (mostly) the same bytes to a different file descriptor connected to `127.0.0.1:8080`. The kernel's scheduler decides when each process's threads actually run on a CPU core — there's no special-casing for "the web server" versus any other process.

This is also why resource limits matter in ways that surprise people coming from higher-level languages: a Linux process has a limit on open file descriptors (`ulimit -n`), and every open connection — client-to-nginx, nginx-to-Tomcat, Tomcat-to-database — consumes one. Hit that limit under load, and you get `Too many open files` errors that have nothing to do with your application logic.

## Where the request goes next

The byte stream nginx proxied to `127.0.0.1:8080` is now sitting in a socket that belongs to the JVM process — specifically, to Tomcat's connector thread pool. What happens the instant Tomcat reads those bytes — how it turns them into a `HttpServletRequest`, hands them to a thread, and how Spring routes that thread's work to your controller — is where the JVM's own concurrency model takes over.

**Next: [Java and the JVM: Threads and Spring]({{< ref "jvm-threads-and-spring.md" >}})**
