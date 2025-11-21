---
title: "Building PeerLink: A Journey into Distributed File Sharing"
date: 2025-01-24
draft: false
author: "Mohamed"
tags:
  - Distributed Systems
  - File Sharing
  - Docker
  - Redis
  - Node.js
image: /images/post.jpg
description: "A deep dive into building PeerLink, a distributed file sharing platform with cluster architecture, real-time transfers, and horizontal scaling"
toc: true
---

# Building PeerLink: A Journey into Distributed File Sharing

Hey everyone! I wanted to share what I've been working on lately - PeerLink, a distributed file sharing platform that I'm genuinely excited about. Let me walk you through what makes it special and some of the challenges I ran into along the way.

## The Problem I Was Trying to Solve

You know how most file sharing tools are either super simple peer-to-peer apps or massive cloud solutions? I wanted something in between - something that could scale horizontally, handle real-time transfers efficiently, and not die if one server goes down. Basically, I wanted a system that could grow with you.

## What Makes PeerLink Different

### The Cluster Architecture

Instead of having one big server doing everything, PeerLink runs as a cluster of nodes that work together. Think of it like a team where everyone has a specific role, but if someone's busy, others can step in.

The really cool part? These nodes can be on completely different machines, in different data centers, or just different Docker containers on your laptop. They discover each other automatically through Redis and elect a "master" node to coordinate things. If the master goes down, they just elect a new one. No human intervention needed.

### Real-Time Everything

File transfers happen in real-time using WebSockets (Socket.IO specifically). But here's where it gets interesting - if you're on Node A and your friend is on Node B, the system figures this out and routes the data chunks between nodes seamlessly. You never even know you're on different servers.

I implemented a smart routing system that tries local delivery first (fastest), then direct worker-to-worker communication through Redis, and finally falls back to the master node for coordination. This means most transfers are super fast because they skip unnecessary hops.

### Handling the Race Conditions

One of the trickiest bugs I dealt with was the "ghost socket" problem. When someone reconnects (maybe their WiFi hiccupped), they get a new socket ID, but Redis might still have the old one cached for a few milliseconds. Messages would get sent to a socket that doesn't exist anymore and just... vanish.

The fix? I added a fallback system that looks up clients by their ID if the socket isn't found, then checks the local cache for the current socket. It sounds simple, but it took forever to figure out why transfers were randomly failing!

### Going Native with C++

JavaScript is fast enough for most things, but for calculating checksums on huge files? Not quite. So I wrote a native C++ addon using N-API that uses SIMD instructions (fancy CPU parallelization) to compute checksums about 400% faster than the standard crypto libraries.

Is it overkill? Maybe. But when you're transferring gigabyte files and need to verify integrity, every millisecond counts.

## The Frontend Experience

I kept the UI deliberately simple - React with Tailwind CSS. You pick files, hit share, and get a link. That's it. No accounts, no signups, just instant peer-to-peer transfers.

But under the hood, the frontend is smarter than it looks. It automatically discovers available backend nodes, picks a random worker (not the master, to avoid overloading it), and handles reconnections gracefully. I even made it so when a peer disconnects, the page just refreshes instead of showing a confusing modal - users wanted it to feel like a clean slate.

### The Download Confirmation Bug

Here's a fun one - senders weren't seeing when receivers finished downloading. Turns out, the "upload-complete" event was firing after the "download-confirmed" event sometimes, overwriting the status back to "uploaded" instead of "completed."

The fix was simple once I understood it - just check if the status is already "completed" before updating it. Classic race condition that only showed up under specific network conditions.

## Dockerizing Everything

The whole stack runs in Docker with separate services for PostgreSQL, Redis, the backend, and frontend. This was crucial because I wanted to be able to scale horizontally - just run `docker-compose up --scale backend=5` and boom, you have 5 worker nodes.

The frontend automatically discovers them through Redis and load balances connections. It's pretty satisfying to watch in action.

## What I Learned

**Distributed systems are hard** - Way harder than I thought. Race conditions everywhere, network partitions, cache invalidation... it's all the classic problems but in real life.

**Redis is magic** - Seriously. Pub/Sub for inter-node messaging, caching for session data, and even leader election with TTL keys. It does everything.

**TypeScript saves lives** - Having strict types caught so many bugs before they made it to production. The initial setup time is worth it 10x over.

**Users want simplicity** - I built all this complex infrastructure, but users just want "click, share, done." And that's fine! The complexity should be invisible.

**Documentation matters** - I spent almost as much time on the README as the code. Making environment variables configurable, writing clear setup instructions, explaining the architecture - it all matters if you want people to actually use (or contribute to) your project.

## What's Next?

Right now, PeerLink is in a pretty good state. The core functionality works, it scales, and it's reasonably fast. But there's always more to do:

- Better error messages for users
- Metrics and monitoring dashboards
- Maybe WebRTC for direct peer connections (bypassing the server entirely for transfers)
- Mobile app? (That's a whole other adventure)

## Try It Out

If you're curious, the whole thing is open source on GitHub. You can run it locally with Docker or even deploy it to your own infrastructure. I tried to make the setup as painless as possible with detailed docs and sensible defaults.

The weirdest part about building this? I started just wanting a simple file sharing tool for myself, and it turned into this whole distributed systems project. But honestly, that's what makes it fun - solving problems you didn't even know you'd encounter.

Anyway, that's PeerLink. Built it, broke it, fixed it, broke it again, and now it actually works. If you end up trying it, let me know what you think!

**P.S.** - If you're wondering about the video in the README not showing up on GitHub, yeah, I went through that struggle too. Ended up converting it to a GIF. Sometimes the simplest solution is the right one.

