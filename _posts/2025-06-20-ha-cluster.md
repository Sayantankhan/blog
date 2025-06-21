---
layout: post
title:  "High Availability Demo 🚀 Pacemaker + Corosync + Docker"
date:   2025-06-20 10:00:00 +0530
tags:   [ha, pacemaker, corosync, docker, clustering]
---

Setting up high availability (HA) can feel overwhelming, but tools like **Pacemaker**, **Corosync**, and **Docker** make it surprisingly approachable. In this post, I’ll walk through how I built a multi-node HA cluster that runs a Flask API across three Docker containers using Pacemaker as the cluster manager and Corosync as the messaging layer.

---

## 🛠️ Project Overview

The `HA-Setup` repo uses:

- **docker-compose**: to launch a cluster of three nodes + one standby
- **Corosync**: provides cluster messaging
- **Pacemaker**: manages resources and failover logic
- **Anything OCF agent**: wraps the Flask API script
- **DRBD (optional)**: for stateful block replication

---
