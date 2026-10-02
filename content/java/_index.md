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

Java sits in the middle, which is exactly why it works as the hub here: to understand what a Spring controller is actually doing, you have to understand what arrived before it (a TCP connection accepted by the kernel, proxied by nginx) and what it hands off next (a connection pool, a transaction, an index lookup). Most "backend" explanations start and end at the application code. This one won't.
