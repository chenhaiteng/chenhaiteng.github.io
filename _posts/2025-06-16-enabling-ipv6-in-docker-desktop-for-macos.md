---
layout: post
title:  "Enabling IPv6 in Docker Desktop for macOS"
date:   2025-06-16 14:00:00 +0800
categories: docker
tags: macos IPv6 docker
---
 
This guide walks through enabling IPv6 networking for containers in Docker Desktop for macOS.

A few days ago, I started using Docker to build a RESTful API service that retrieves data from a third-party cloud provider and stores it in a separate database for future use.

Initially, the database was hosted in a local Docker container, and everything worked fine. However, when I moved the database to Supabase and tried connecting to it from my Dockerized application, the connection failed with the following error:
```
  Network is unreachable
      Is the server running on that host and accepting TCP/IP connections?
```
After digging around and consulting ChatGPT, I discovered two key issues:

1. Docker Desktop doesn’t enable IPv6 or dual-stack mode by default.

2. On Supabase, direct connections to PostgreSQL over IPv4 are only supported on Pro plans+.

These two factors together caused the “network is unreachable” error when my container tried to connect.

Although Supabase offers other IPv4-compatible alternatives (such as using the transaction pooler), I chose to enable IPv6 on the Docker side instead — considering both long-term trends and the cost of upgrading to a Pro plan.

That led me to explore Docker's IPv6 configuration — and here’s a step-by-step guide based on what worked for me.

---
## How to setup IPv6 for Dockerized services

**Issue:** 
> **Dockerized services failed to connect to Supabase due to lack of outbound IPv6 support**

**Goal:**  
> **Enable outbound IPv6 networking for containers to connect to services like Supabase**

---

### **Environment:**
> - **macOS 15.5**  
> - **Docker Desktop 4.42.0**  
> - **Docker Engine v28.2.2**

---

### Steps:

#### 1. Enable Dual Stack IPv4/IPv6 in Docker Settings<a name="step-1"/>

- Open **Docker Desktop**
- Go to `Settings → Resources → Network`
- Find the section **Default networking mode**, and select `Dual IPv4/IPv6`

---

#### 2. Enable IPv6 in Docker Engine Configuration<a name="step-2"/>

- Go to `Settings → Docker Engine`
- Append the following config:

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "{Unique local address}"
}
```
The [Unique local address (ULA)](https://en.wikipedia.org/wiki/Unique_local_address) has the format: **fdxx:xxxx:xxxx::/64**

- Click **Apply & Restart**

---

#### 3. Create a Custom IPv6 Network<a name="step-3"/>

```bash
docker network create \
  --driver=bridge \
  --ipv6 \
  --subnet="{Unique local address}" \
  ipv6net
```

Note: the subnet should not overlap with "fixed-cidr-v6".

---

#### 4. Configure `docker-compose.yml` to Use the IPv6 Network<a name="step-4"/>

```yaml
services:
  your_service:
    build: .
    networks:
      - ipv6net

networks:
  ipv6net:
    external: true
```

---

#### 5. Inside the Container: Test Connectivity<a name="step-5"/>

In root of the docker project, use following command to enter the bash in container:
```docker compose exec your_service bash```

Then, execute following command to install netcat:
```bash
apt update && apt install -y netcat-openbsd
```

And test connectivity with following command:
```bash
nc -vz {the.domain.to.test} {test_port}
```

Expected output:
```
Connection to {the.domain.to.test} port {test_port} [tcp/postgresql] succeeded!
```

---
## Summary

Key requirements to enable outbound IPv6 networking from Docker containers on macOS:  
- Docker Desktop IPv6 support must be turned on in both settings and engine config (See: [Step 1](#step-1) and [Step 2](#step-2))  
- A custom Docker network with IPv6 enabled must be created (See: [Step 3](#step-3))  
- Containers must be attached to that network(See: [Step 4](#step-4))

With this setup, services running in containers can connect to external IPv6-only hosts — such as Supabase on the free plan.

---

## Appendix

### Network Tools

- **Docker built-in network command**  
  - List networks: `docker network ls`

- **Netcat (for connectivity testing)**  
  - Install in container:  
    `apt update && apt install -y netcat-openbsd`
  - Usage (inside container):  
    `nc -vz {the.domain.to.test} {test_port}`
  - Usage (from host to container):  
    `docker exec <container_name> nc -vz {the.domain.to.test} {test_port}`

---

## References

- [Dedicated IPv4 Address for Ingress (Supabase Docs)](https://supabase.com/docs/guides/platform/ipv4-address)
- [Manage IPv4 Usage (Supabase Docs)](https://supabase.com/docs/guides/platform/manage-your-usage/ipv4)
- [Direct Connection (Supabase Docs)](https://supabase.com/docs/guides/database/connecting-to-postgres#direct-connection)
- [Docker for Mac Doesn’t Work on IPv6 Network (GitHub Issue)](https://github.com/docker/for-mac/issues/1432)

