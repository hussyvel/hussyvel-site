---
title: "Java and the JVM: Threads and Spring"
date: 2026-10-01T09:30:00-03:00
draft: false
weight: 4
tags: ["java", "jvm", "threads", "spring", "series"]
categories: ["java"]
description: "The request lands in a thread pool, a DispatcherServlet routes it, and the JVM does a surprising amount of work you never asked for."
summary: "This is the hub of the series. The request is finally in your code — but it took a thread pool, a dispatcher, and a garbage collector you never call directly to get here."
ShowToc: true
---

Fourth post, and the center of this series — **[see the full chain]({{< ref "/java" >}})**. The [previous post]({{< ref "linux-process-socket-nginx.md" >}}) left off with bytes sitting in a socket owned by the JVM process. This is where they finally become something resembling "your code."

## Tomcat's thread pool: the first JVM-level decision

Spring Boot's embedded Tomcat doesn't spawn a new OS thread per request indefinitely — it maintains a bounded **thread pool** (`server.tomcat.threads.max`, default 200). A connector thread accepts the connection, parses the raw bytes into an `HttpServletRequest`, and hands execution to a worker thread from that pool.

This bound is not a performance tweak you can ignore — it's a hard ceiling on concurrency. If all 200 threads are busy (say, each blocked waiting on a slow database query), request #201 queues. This is precisely why a slow SQL query — the kind the [last post in this series]({{< ref "sql-connection-transaction-index.md" >}}) deals with directly — doesn't just make *that* request slow. It can starve the thread pool and make the entire application unresponsive, even for requests that don't touch the database at all.

![Comic: a developer panics that everything is slow, even the health check, and it turns out one slow query is blocking all 200 threads in the pool](/img/comics/thread-pool-exhausted.svg)

## The JVM thread, not the OS thread

It's worth being precise here: a Java `Thread` is, in the standard HotSpot JVM (pre–Project Loom/virtual threads), a thin wrapper around a native OS thread. When Tomcat's worker thread blocks on a JDBC call waiting for the database, it isn't doing clever cooperative scheduling — it's genuinely parking an OS thread, which the Linux kernel's scheduler then simply doesn't run until the I/O completes. This is the direct, unglamorous link back to the [process and scheduling model]({{< ref "linux-process-socket-nginx.md" >}}) from the previous post: Java's concurrency model, for traditional (non-reactive) Spring MVC, rides entirely on OS-level threads and blocking I/O.

```
Tomcat connector thread
  → parses HTTP request
  → borrows a worker thread from the pool
     → worker thread blocks on JDBC call (real OS-level block)
     → ... database does its work ...
     → worker thread resumes, builds response
  → worker thread returned to pool
```

This is also exactly why **virtual threads** (Project Loom, standard since Java 21) matter: they let you write the same blocking-looking code, but the JVM — not the OS — parks the lightweight virtual thread and frees the underlying OS "carrier" thread to do other work while waiting. Spring Boot 3.2+ can run on a virtual-thread executor with essentially no code changes, which turns that hard 200-thread ceiling into something far more elastic.

## Spring's DispatcherServlet: one front door

Once a worker thread has the request, Spring MVC's `DispatcherServlet` is the single entry point that decides where it goes:

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }
}
```

`DispatcherServlet` matches the incoming `GET /orders/42` against registered handler mappings, resolves method arguments (`@PathVariable Long id` gets parsed straight out of the URL), runs any configured interceptors or filters (this is often where authentication, logging, and — tying back to the [JavaScript post]({{< ref "javascript-in-the-browser.md" >}}) — CORS headers get applied), and finally invokes your controller method on the current thread. There's no additional threading introduced here by default: the same worker thread that Tomcat handed the request to is the one running your business logic.

## What the JVM is doing that you didn't ask for

While your controller and service code run, the JVM itself is doing work you never explicitly invoke:

- **Garbage collection** — every `OrderDto`, every intermediate `String`, every `Optional` you allocate becomes garbage the moment it's no longer referenced, and a GC thread (running concurrently, on modern collectors like G1 or ZGC) will eventually reclaim that memory. A GC pause under load is a very real, very measurable source of request latency spikes.
- **JIT compilation** — your bytecode starts out interpreted, but the JVM profiles hot methods and compiles them to native machine code at runtime (the C1/C2 compilers), which is part of why Java services are often slower on their very first requests ("warm-up") than after sustained load.
- **Class loading** — Spring's component scanning, proxy generation for `@Transactional` methods, and AOP-based features all happen through dynamic class loading and bytecode generation at startup, which is a large part of why Spring Boot apps have a noticeable startup time compared to, say, a Go binary.

## Handing off to the database

When `orderService.findById(id)` needs data, it doesn't open a fresh TCP connection to Postgres or MariaDB for every call — it borrows one from a connection pool (HikariCP, Spring Boot's default), issues SQL over it, and returns it when done. That handoff — a pooled connection, a transaction boundary, and whatever index the database uses (or fails to use) to satisfy the query — is the last leg of the outbound journey.

**Next: [SQL: Connection, Transaction, and Index]({{< ref "sql-connection-transaction-index.md" >}})**
