---
title: "Hello, STUNned to meet you! (Deprecated)"
date: 2024-11-26
draft: false
author: "Mohamed"
tags:
  - Networking
  - Sockets
  - Peering
  - Deprecated
image: /images/post.jpg
description: "[DEPRECATED] This post describes an older version of PeerLink. See the latest blog post for the current distributed file sharing platform."
toc: 
---

# Hello, STUNned to meet you

> **⚠️ DEPRECATED:** This post describes an older version of PeerLink that is no longer in use. The current version uses a completely different distributed systems architecture. See the [latest blog post](/blogs/building-peerlink) for information about the current implementation.

So, here's the deal: file-sharing. It's everywhere. From memes to work docs, we share files like it's nobody's business. But when you need something secure, fast, and private? Well, that's a bit trickier. Today, I'm going to walk you through how I built **Peer Link**, a WebRTC-powered file transfer service that prioritizes privacy, speed, and reliability. Let's dive in, shall we?

---

## The Story

It all started when I needed to share a file but didn’t want services like WhatsApp or Messenger to be able to access it. So, I started brainstorming for a solution. The obvious options—self-hosting and encryption—seemed like a good fit, but I wanted something more efficient and seamless.

That’s when I had the idea for **Peer Link**.

---

## The Core: WebRTC and PeerJS

At the heart of **Peer Link** is WebRTC (Web Real-Time Communication). It’s like the secret sauce for peer-to-peer connections. But plain sauces are boring, so I decided to spice things up by wrapping WebRTC in **PeerJS**, making it elegant and much easier to use. This setup allows browsers to communicate directly with each other, without needing a server to store files. That’s right—no server-side storage required. It’s super efficient!

Here’s a brief example of how the setup works with the ICE servers:

```js
const peer = new Peer({
  config: {
    iceServers: [
      {
        urls: "stun:stun.example.com:3478",
        username: "user",
        credential: "pass"
      },
      {
        urls: "turn:turn.example.com:3478",
        username: "user",
        credential: "pass"
      }
    ]
  },
  secure: true
});
```

Every "peer" first connects to these ICE servers to establish a communication route.

If you’re unfamiliar with **STUN** (get it?) or **TURN** servers, don’t worry—I’ll explain them in a moment.

---

## Real-time Connection Monitoring

Now that we’ve got the core set up, what happens if someone disconnects during a transfer? This was one of the very first challenges I faced when designing the protocol. To solve it, I came up with a “heartbeat” mechanism that pings every 2 seconds. Based on this, we can update the connection status in real time.

```js
const startHeartbeat = (conn: DataConnection) => {
  const interval = setInterval(() => {
    if (conn.open) {
      conn.send({ type: 'heartbeat' });
    } else {
      setIsConnectionReady(false);
      clearInterval(interval);
    }
  }, 2000);
};

// Heartbeat acknowledgment
conn.on('data', (data) => {
  if (data.type === 'heartbeat') {
    conn.send({ type: 'heartbeatAck' });
  } else if (data.type === 'heartbeatAck') {
    lastHeartbeatRef.current = Date.now();
  }
});
```

Without this mechanism, your file transfers could just... drop dead mid-transfer, and you wouldn’t even know what happened.

---

## Dynamic Status Management

Next up—another problem I encountered: I like to know exactly what’s going on at all times. So, I built a dynamic status system that provides clear feedback on the connection and the transfer process.

```js
const getDisplayStatus = () => {
  if (isTransferring) {
    return 'Transferring';
  } else if (!isConnectionReady) {
    return 'Waiting for connection';
  }
  return 'Connected';
};

const getStatusColor = () => {
  if (isTransferring) {
    return 'bg-blue-500';
  } else if (status === 'Waiting for connection') {
    return 'bg-yellow-500';
  } else if (isConnectionReady) {
    return 'bg-green-500';
  }
  return 'bg-red-500';
};
```

This is a simplified version, but it ensures users always know exactly where things stand. Are we transferring? Waiting for a connection? Or are we good to go? It’s like a dashboard for your connection status—easy and intuitive.

---

## STUN/TURN Server Setup

Ah, the STUN and TURN servers. These are crucial for ensuring your peer-to-peer connection can happen, even if you’re behind a firewall or on different networks. Here’s how I set them up on an **Ubuntu 20.04** server using **coturn**.

First, install the necessary dependencies:

```bash
sudo apt-get update
sudo apt-get install coturn
sudo apt-get install ufw
```

Next, configure the firewall:

```bash
sudo ufw allow 3478/tcp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp
sudo ufw allow 5349/udp
sudo ufw allow 49152:65535/udp
```

And finally, here’s how to configure the **turnserver.conf**:

```bash
# Network settings
listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0
external-ip=SERVER_IP
min-port=49152
max-port=65535

# Authentication
user=username:password
realm=my.realm.org

# TLS settings
cert=/etc/letsencrypt/live/turn.domain.com/fullchain.pem
pkey=/etc/letsencrypt/live/turn.domain.com/privkey.pem

# Performance
total-quota=100
stale-nonce=600
```

This setup ensures that peers can always find a way to communicate, even if they’re behind complex network setups like NATs or firewalls.

---

## Chunked File Transfer with Progress

We don’t want to transfer huge files in a single go—that’s just asking for trouble. Instead, we break files into **chunks** and send them one by one. Plus, we track progress in real-time so users always know how much is left. Here’s how it works:

```js
const uploadFile = async (file: File, fileIndex: number) => {
  const chunkSize = 16 * 1024;
  const totalChunks = Math.ceil(file.size / chunkSize);
  let offset = 0;

  setFileUploads(prevUploads =>
    prevUploads.map((upload, index) =>
      index === fileIndex ? { ...upload, status: 'uploading' } : upload
    )
  );

  while (offset < file.size) {
    const chunk = file.slice(offset, offset + chunkSize);
    const buffer = await chunk.arrayBuffer();
    
    dataConnectionRef.current?.send({
      type: 'file',
      filename: file.name,
      fileData: buffer,
      chunkIndex: Math.floor(offset / chunkSize),
      totalChunks: totalChunks
    });

    const progress = Math.floor((offset / file.size) * 100);
    setFileUploads(prevUploads =>
      prevUploads.map((upload, index) =>
        index === fileIndex ? { ...upload, progress } : upload
      )
    );

    offset += chunkSize;
  }
};
```

This method ensures that file transfers are efficient, and users can track their progress along the way—because, let’s be honest, who doesn’t love a good progress bar?

---

## Connection Recovery

We’ve all been there—your connection drops, and you’re left wondering what happened. Well, **Peer Link** has a built-in connection recovery system. If something goes wrong, it’ll automatically reset the connection and give you a fresh start. Here’s how it works:

```js
const resetConnection = () => {
  if (peer) {
    peer.destroy();
  }
  setStatus('Resetting connection...');
  setPeer(null);
  setPeerId('');
  dataConnectionRef.current = null;
  fileChunksRef.current = {};
  setDownloadedFiles([]);
  setFileUploads([]);
  setDownloadProgress(0);
  setCurrentDownloadFileName('');
  setIsConnectionReady(false);

  // Reinitialize peer after cleanup
  initializePeer();
};
```

With this feature, you’re always ready to try again, even if something goes wrong.

---

## Transfer Status Interface

Finally, to keep everything organized, I’ve defined interfaces for the transfer status and file uploads in TypeScript. This helps ensure everything stays neat and type-safe.

```ts
interface TransferStatus {
  type: 'status';
  isTransferring: boolean;
}

interface FileUpload {
  file: File;
  progress: number;
  status: 'pending' | 'uploading' | 'completed' | 'error';
}
```

This way, I can track the status of every file upload and catch any issues before they become problems.

---

This concludes our deep dive into the making of **Peer Link**. By leveraging WebRTC, PeerJS, and a handful of clever optimizations, I created a file transfer service that is fast, secure, and easy to use. The focus on real-time connection monitoring, dynamic status management, and connection recovery ensures that file transfers always run smoothly. Hope this gives you some ideas for your next project!

--- 

---

## ⚠️ Deprecated Notice

**This blog post describes an older, deprecated version of PeerLink that is no longer in use.** The architecture and implementation described here have been completely replaced with a new distributed systems approach.

For information about the current version of PeerLink, please see the latest blog post: **[Building PeerLink: A Journey into Distributed File Sharing](/blogs/building-peerlink)**.

The old WebRTC-based implementation described in this post is no longer maintained or available.

---