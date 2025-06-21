---
layout: post
title:  "High Availability Demo 🚀 Pacemaker + Corosync + Docker"
date:   2025-06-20 10:00:00 +0530
tags:   [ha, pacemaker, corosync, docker, clustering]
---

Setting up high availability (HA) can feel overwhelming, but tools like **Pacemaker**, **Corosync**, and **Docker** make it surprisingly approachable. In this post, I’ll walk through how I built a multi-node HA cluster that runs a Flask API across three Docker containers using Pacemaker as the cluster manager and Corosync as the messaging layer.

GitHub Repo: [Sayantankhan/HA-Setup](https://github.com/Sayantankhan/HA-Setup)

---

## 🛠️ Project Overview

The `HA-Setup` repo uses:

- **docker-compose**: to launch a cluster of three nodes + one standby
- **Corosync**: provides cluster messaging
- **Pacemaker**: manages resources and failover logic
- **Anything OCF agent**: wraps the Flask API script
- **DRBD (optional)**: for stateful block replication

---

## ⚙️ Architecture

We use Docker to simulate four independent nodes that communicate over a custom bridge network. Each node runs Ubuntu and is equipped with Corosync and Pacemaker. A sample `flask-api` application is used as the cluster resource, which Pacemaker will manage and failover based on health checks.

---

## 🐳 Dockerfile

To build our cluster nodes, we start with a Dockerfile that installs all required packages and sets up the application environment.

```dockerfile
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
  pacemaker corosync crmsh resource-agents iputils-ping \
  python3 python3-pip systemd systemd-sysv vim net-tools \
  && apt-get clean

RUN pip3 install flask

COPY app.py /usr/local/bin/app.py
COPY config/init-pacemaker*.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/init-pacemaker*.sh
RUN echo -e '#!/bin/bash\npython3 /usr/local/bin/app.py' > /usr/local/bin/start-api \
    && chmod +x /usr/local/bin/start-api

CMD ["/sbin/init"]
```

---

## 🧱 Docker Compose

To bring up the multi-node cluster, we define a `docker-compose.yml` file with four services, each representing a node.

```yaml
version: "3"

services:
  node1:
    build: .
    container_name: node1
    hostname: node1
    privileged: true
    environment:
      - APP_ID=node1
    networks:
      ha_net:
        ipv4_address: 10.0.0.11
    volumes:
      - ./config/corosync-multi-node.conf:/etc/corosync/corosync.conf
    ports:
      - "5001:5000"

  node2:
    build: .
    container_name: node2
    hostname: node2
    privileged: true
    environment:
      - APP_ID=node2
    networks:
      ha_net:
        ipv4_address: 10.0.0.12
    volumes:
      - ./config/corosync-multi-node.conf:/etc/corosync/corosync.conf
    ports:
      - "5002:5000"
  
  node3:
    build: .
    container_name: node3
    hostname: node3
    privileged: true
    environment:
      - APP_ID=node3
    networks:
      ha_net:
        ipv4_address: 10.0.0.13
    volumes:
      - ./config/corosync-multi-node.conf:/etc/corosync/corosync.conf
    ports:
      - "5003:5000"

  node4:
    build: .
    container_name: node4
    hostname: node4
    privileged: true
    environment:
      - APP_ID=node4
    networks:
      ha_net:
        ipv4_address: 10.0.0.14
    volumes:
      - ./config/corosync-multi-node.conf:/etc/corosync/corosync.conf
    ports:
      - "5004:5000"

networks:
  ha_net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
```

---

## 📡 Corosync Configuration

The `corosync.conf` sets up communication between the cluster nodes using UDPU (UDP unicast). Here is a sample configuration:

```bash
# Filename: corosync.conf
totem {
    version: 2
    secauth: off
    cluster_name: docker-cluster
    transport: udpu
}

nodelist {
    node {
        ring0_addr: 10.0.0.11
        nodeid: 1
        name: node1
    }
    node {
        ring0_addr: 10.0.0.12
        nodeid: 2
        name: node2
    }
    node {
        ring0_addr: 10.0.0.13
        nodeid: 3
        name: node3
    }
    node {
        ring0_addr: 10.0.0.14
        nodeid: 4
        name: node4
    }
}

quorum {
    provider: corosync_votequorum
    expected_votes: 3
    two_node: 0
    wait_for_all: 0
}

logging {
    to_syslog: yes
}
```

This ensures each node knows about the others and can form a quorum.

---

## 🔁 Cluster Bootstrap Script

`init-pacemaker-multi-member.sh` waits for all nodes to join and then sets cluster parameters, disables stonith and quorum checks, and registers the `flask-api` resource.

```bash
#!/bin/bash

echo "[INFO] Waiting for Corosync membership..."
expected_peers=3  # Total nodes - 1
for i in {1..30}; do
    connected=$(corosync-cfgtool -s | grep -c "connected")
    if [ "$connected" -ge "$expected_peers" ]; then
        echo "[INFO] Corosync is up. ($connected/$expected_peers peers connected)"
        break
    fi
    echo "[INFO] Waiting for Corosync... ($i) ($connected/$expected_peers connected)"
    sleep 1
done

echo "[INFO] Waiting for Pacemaker to be ready..."
for i in {1..30}; do
    crm_mon -1 > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo "[INFO] Pacemaker is ready!"
        break
    else
        echo "[INFO] Waiting for Pacemaker... ($i)"
        sleep 2
    fi
done

if ! crm_mon -1 > /dev/null 2>&1; then
    echo "[ERROR] Pacemaker not ready after waiting."
    exit 1
fi

echo "[INFO] Setting Pacemaker cluster properties..."
crm configure property stonith-enabled=false
crm configure property no-quorum-policy=ignore

# Create the primitive if not exists
if ! crm_resource --resource flask-api --query >/dev/null 2>&1; then
    echo "[INFO] Creating flask-api primitive..."
    crm configure primitive flask-api ocf:heartbeat:anything \
      params binfile="/usr/local/bin/start-api" \
             pidfile="/var/run/flask-api.pid" \
             logfile="/var/log/flask-api.log" \
      op monitor interval=10s
else
    echo "[INFO] Primitive 'flask-api' already exists. Skipping."
fi

# # Create the clone wrapper if not exists
# if ! crm configure show flask-api-clone >/dev/null 2>&1; then
#     echo "[INFO] Creating clone of flask-api..."
#     crm configure clone flask-api-clone flask-api
# else
#     echo "[INFO] Clone 'flask-api-clone' already exists. Skipping."
# fi

echo "[INFO] Pacemaker configuration complete."
exec "$@"
```

---

## 🚀 Quick Start

```bash
docker compose build
docker compose up -d
docker exec -it node1 /usr/local/bin/init-pacemaker-multi-member.sh
```

To test quorum:
```bash
docker stop node2
docker exec -it node1 crm status
```

---

## 🔎 Monitor Cluster

```bash
crm status
crm_resource --resource flask-api --locate
watch crm status
```

---

## 🧠 Pacemaker Tips

- `primitive`: defines a single instance of a resource
- `clone`: allows running on multiple nodes
- You can use location rules to control preferred nodes

```bash
crm configure clone flask-api-clone flask-api meta clone-max=3
```

### Set Node Preferences

- **Prefer `node1` (Active)**
```bash
crm configure location prefer-node1 flask-api 100: node1
```

- **Set Lower Priority for Standby Nodes**
```bash
crm configure location prefer-node2 flask-api 50: node2
crm configure location prefer-node3 flask-api 10: node3
```

- **Cold Standby (Avoid a Node Completely)**
```bash
crm configure location avoid-node3 flask-api -INFINITY: node3
```

- **Remove a Rule**
```bash
crm configure delete avoid-node3
```

- ### Put a Node in Standby
```bash
crm node standby node4
```

---

### Manual Overrides

- **Force Resource to Run on node2**
```bash
crm configure location force-api-on-node2 flask-api inf: node2
```

- **Manually Stop the Resource**
```bash
crm resource stop flask-api
```
---

## ⚖️ Multi-Node Quorum & Resource Types

### Primitive vs Clone Behavior

- A `primitive` in Pacemaker is meant to run on **exactly one node at a time**.
- Pacemaker assumes running it on multiple nodes may cause conflicts (ports, data integrity, etc.).
- If you want to run the same service **on multiple nodes in parallel**, use a **`clone`**.

### Best Practice

```bash
sudo docker compose -f docker-compose-multinode.yml up -d
sudo docker exec -it node1 /usr/local/bin/init-pacemaker-multi-member.sh
```
✅ Do this:
- Define a `primitive` → defines what to run
- Wrap it in a `clone` → tells Pacemaker to run it on multiple nodes
```bash
sudo docker exec -it node2 bash
crm_resource --resource flask-api --locate
crm configure clone flask-api-clone flask-api meta clone-max=3
```

## Updating clone numbers:
```crm
clone flask-api-clone flask-api \
    meta clone-max="3"
```

### Disabling Quorum Check
```bash
crm configure property no-quorum-policy=ignore
```
### Cleaning a Failed
```bash
crm resource cleanup flask-api
```

`clone-max` defines how many **copies (instances)** of the service are allowed to run cluster-wide. The purpose of clone-max in:  how many copies (instances) of the clone Pacemaker is allowed to run at the same time across the cluster.

