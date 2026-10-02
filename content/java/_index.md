---
title: "Java"
description: "Java as the hub of a request's journey: from a browser click to a database row and back."
summary: "A click in a browser becomes a row in a database and a response back on screen. This series follows that request through every layer, with Java holding the center."
ShowToc: false
---

A click happens in a browser. A few hundred milliseconds later, a page updates with data that, until that moment, only existed as a row in a database on another continent. In between, the request crosses five completely different worlds, each with its own rules, failure modes, and vocabulary.

This series follows that journey end to end:

```
JavaScript (browser) → HTTP/network → Linux (process, socket, nginx)
   → Java (JVM, threads, Spring) → SQL (connection, transaction, index) → and back
```

Java sits in the middle, which is exactly why it works as the hub for this series: to understand what a Spring controller is actually doing, you have to understand what arrived before it (a TCP connection accepted by the kernel, proxied by nginx) and what it hands off next (a connection pool, a transaction, an index lookup). Most "backend" explanations start and end at the application code. This one doesn't.

## The chain

1. **[JavaScript in the Browser]({{< ref "javascript-in-the-browser.md" >}})** — where the request is born: `fetch`, the event loop, and why the browser doesn't just block while waiting.
2. **[HTTP and Networking]({{< ref "http-and-networking.md" >}})** — DNS, TCP's handshake, TLS, and what an HTTP request actually looks like as bytes on the wire.
3. **[Linux: Process, Socket, and Nginx]({{< ref "linux-process-socket-nginx.md" >}})** — how the kernel turns a stream of bytes into something a server process can read, and why nginx sits in front of Java almost everywhere.
4. **[Java and the JVM: Threads and Spring]({{< ref "jvm-threads-and-spring.md" >}})** — the request lands in a thread pool, a `DispatcherServlet` routes it, and the JVM does a surprising amount of work you never asked for.
5. **[SQL: Connection, Transaction, and Index]({{< ref "sql-connection-transaction-index.md" >}})** — the request finally touches data: a pooled connection, a transaction boundary, and the index that decides if the query takes 2ms or 2s.

Read them in order if you want the full trip, or jump straight to the layer you're debugging today. Each post assumes you know roughly what the other layers do, but not the details — that's the point of connecting them.
