---
title: "Linux"
description: "Processes, sockets, file descriptors, and the kernel decisions happening under every server you deploy."
summary: "The layer under every other layer: processes, sockets, and the kernel quietly deciding what runs where."
ShowToc: false
---

Everything else in this site eventually runs on top of Linux — a process with its own memory space, a socket that's really just a file descriptor, a kernel scheduler deciding which thread gets the CPU next. This section covers that layer: process and file descriptor management, networking, nginx, and the day-to-day of running Java (and everything else) in production.
