---
title: "JavaScript in the Browser: Where the Request Begins"
date: 2026-10-01T09:00:00-03:00
draft: false
weight: 1
tags: ["javascript", "browser", "http", "series"]
categories: ["java"]
description: "Every backend request starts with a line of JavaScript in a browser. Here's what actually happens before a single byte hits the network."
summary: "Before HTTP, before Java, before SQL — there's a fetch() call sitting in an event loop. This is where our request's journey begins."
ShowToc: true
---

This is the first post in a series that follows a single request across five layers: **[the full chain lives here]({{< ref "/java" >}})**. We start where the request starts — not on a server, but in a browser tab, in JavaScript.

## The illusion of a blocking call

Code like this reads like a simple, synchronous call:

```javascript
const response = await fetch("/api/orders/42");
const order = await response.json();
renderOrder(order);
```

It isn't one. JavaScript in the browser runs on a **single thread**. If `fetch` actually blocked that thread while waiting for a server on another continent, the entire page — scrolling, clicking, animations — would freeze for however long the network takes. It doesn't, because `fetch` doesn't block anything.

## The event loop, briefly

The browser's JavaScript engine has one call stack and one event loop. When you call `fetch`, the engine doesn't do the networking itself inline — it hands the request off to browser APIs that live outside the JS engine (written in C++, running on their own threads), and immediately returns a `Promise`. Your function keeps running, the stack unwinds, and the browser's rendering and input handling continue uninterrupted.

When the network layer actually gets a response — bytes arriving from a socket the OS gave the browser process, which is exactly what the [next two posts]({{< ref "http-and-networking.md" >}}) in this series are about — it doesn't jump back into your code immediately either. It queues a **microtask** (for promises) that only runs once the call stack is empty. That's why `await` "pauses" your async function without ever blocking the thread: under the hood it's just a promise callback scheduled for later.

```
fetch() called
  → JS engine hands off to browser network stack, returns a Promise
  → call stack empties, browser stays responsive
  → ... time passes, TCP/TLS handshake, HTTP exchange happens off-thread ...
  → response arrives → microtask queued
  → event loop picks up the microtask → your `await` resumes
```

## What `fetch` actually triggers

Calling `fetch("/api/orders/42")` sets off a chain that this series follows layer by layer:

1. The browser needs an IP address for the host — a **DNS lookup**, possibly cached.
2. It opens a **TCP connection** to that IP on port 443 (or reuses one it already has, via HTTP keep-alive/connection pooling in the browser itself).
3. It performs a **TLS handshake** if the URL is `https://`.
4. It serializes your request into actual **HTTP** — method, headers, body — and writes it to the socket.

All of that is covered in the [next post on HTTP and networking]({{< ref "http-and-networking.md" >}}). From the browser's point of view, none of it is special-cased — it's just bytes going out on a socket the operating system manages for the browser process, same as any other network client.

## CORS: the browser's own gatekeeper

One thing that's unique to the browser layer — nothing downstream deals with this — is **CORS** (Cross-Origin Resource Sharing). If your JavaScript, served from `https://app.example.com`, calls `fetch("https://api.example.com/orders")`, the browser itself enforces that the API explicitly allows that origin via an `Access-Control-Allow-Origin` response header. This check happens entirely client-side; it exists to protect *users*, not servers — a malicious site can't read responses from your bank's API on your behalf just because your browser is authenticated there.

This matters for the rest of the chain because a Spring Boot application (which we'll get to) has to actively configure CORS rules, or browser-based clients will get requests silently blocked — even though the server responded with a perfectly valid 200.

## What's next

The browser has handed its request to the OS network stack. The next stop is the wire itself: DNS resolution, the TCP handshake, TLS, and what an HTTP request looks like as raw bytes.

**Next: [HTTP and Networking]({{< ref "http-and-networking.md" >}})**
