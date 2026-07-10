---
title: Traefik Proxy for Docker Containers with Cloudflare Full Strict TLS
slug: traefik-proxy-for-docker-containers-with-cloudflare-full-strict-tls
date_published: 2026-01-23T09:13:18.000Z
date_updated: 2026-01-23T09:54:35.000Z
excerpt: Exploring how to use Traefik for routing Docker containers while securely terminating traffic with Cloudflare Full Strict TLS. This PoC demonstrates setting up a clean internal Docker network, connecting multiple services like MinIO, and ensuring requests are routed correctly based on host rules.
---

My senior asked me to explore proxying Docker containers using Traefik. At that time, there was no existing setup yet, so everything had to start from zero. I also didn’t really ask too many questions at first. Turns out, a few simple apps were still running on Kubernetes, and that’s the main reason he wanted to migrate them to plain Docker instead.

He had already tried Nginx Proxy Manager before, so my task was to explore Traefik and Kong. Personally, for simple container setups, I usually just use Cloudflare Tunnel. It’s fast, less things to configure, and usually just works. If tunnel cannot work, then I fall back to plain Nginx.

The Nginx approach is simple and works fine for personal projects. Spin up, proxy, okay already. But production-grade? Not really lah. Below is an example of the kind of Nginx setup I often use when I just want something up quickly.

    version: "3.9"
    services:
      nginx:
        image: nginx:stable-alpine
        container_name: nginx-proxy
        network_mode: host
        volumes:
          - ./conf.d:/etc/nginx/conf.d:ro
        restart: always
    

    server {
        listen 80;
        server_name api.anantafatur.dev;
    
        location / {
            proxy_pass http://127.0.0.1:9001;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
    

---

## PoC Goal

This PoC demonstrates routing Docker containers using Traefik with Cloudflare Full Strict TLS. Traefik was chosen simply because that was the task assigned.

The goal here is not benchmarking or performance tuning. I’m not trying to squeeze every last bit of CPU. The goal is very straightforward: make sure routing works properly, TLS terminates correctly, and nothing weird happens along the way. Once this part is confirmed, only then it makes sense to go further.

---

## Traefik Setup

For this PoC, MinIO was used as the example application. Nothing special here, but realistic enough to test real-world routing and TLS termination.

Traefik runs inside Docker and handles Cloudflare Full Strict TLS using an origin certificate. A dedicated Docker network is used so Traefik and application containers can talk to each other internally. This way, not every container needs to expose ports directly to the host, which keeps things cleaner.

### Folder Structure

    traefik/
    ├── docker-compose.yml
    ├── traefik.yml
    ├── dynamic.yml
    ├── certs/
        ├── origin.crt
        └── origin.key
    

Folder Structure

Configuration is split into static (`traefik.yml`) and dynamic (`dynamic.yml`) files. Slightly more setup work at the start, but once done, it’s easier to manage when things grow bigger.

### Docker Network and Traefik Compose

    docker network create traefik-net
    

    version: "3.9"
    services:
      traefik:
        image: traefik:v3.6.7
        container_name: traefik
        restart: unless-stopped
        networks:
          - traefik-net
        ports:
          - "80:80"
          - "443:443"
          - "127.0.0.1:8080:8080"
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - ./certs:/certs:ro
          - ./traefik.yml:/traefik.yml:ro
          - ./dynamic.yml:/dynamic.yml:ro
        command:
          - "--configFile=/traefik.yml"
    
    networks:
      traefik-net:
        external: true
    

docker-compose.yaml

---

## Traefik Configuration

The `certs/origin.crt` and `certs/origin.key` files are Cloudflare Origin Certificates, which you can generate and download from [Cloudflare Origin CA](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/#deploy-an-origin-ca-certificate). Traefik’s configuration is split into **static** and **dynamic** files: the dynamic file points Traefik to these certificates and enforces a minimum TLS version, while the static file sets up entry points, the dashboard, and Docker/file providers so Traefik can watch containers and TLS settings.

    tls:
      certificates:
        - certFile: /certs/origin.crt
          keyFile: /certs/origin.key
      options:
        default:
          minVersion: VersionTLS12
    

dynamic.yaml

    log:
      level: INFO
    accessLog: {}
    api:
      dashboard:
        insecure: true
    
    entryPoints:
      web:
        address: ":80"
        http:
          redirections:
            entryPoint:
              to: websecure
              scheme: https
      websecure:
        address: ":443"
    
    providers:
      docker:
        exposedByDefault: false
        network: traefik-net
      file:
        filename: /dynamic.yml
        watch: true
    

traefik.yml

---

## MinIO Setup

For this PoC, MinIO was used to test Traefik routing. MinIO exposes two ports: `9001` for the console and `9000` for the S3 API.

Traefik routes traffic based on host rules. Console traffic goes to one hostname, API traffic goes to another. Very clear separation, easy to reason about when something cannot work.

There’s also a helper container that initialises buckets and access keys after a short delay. This is just to make sure MinIO is fully up before running any setup commands. Data persists using Docker volumes, and credentials are set using environment variables.

    version: "3.9"
    services:
      minio:
        image: quay.io/minio/minio:RELEASE.2025-09-07T16-13-09Z
        container_name: minio
        restart: always
        networks:
          - traefik-net
        environment:
          MINIO_ROOT_USER: minioadmin
          MINIO_ROOT_PASSWORD: supersecret
        command: ["server", "--console-address", ":9001", "/data"]
        labels:
          - "traefik.enable=true"
          # MinIO Console
          - "traefik.http.routers.minio-console.rule=Host(`s3.anantafatur.dev`)"
          - "traefik.http.routers.minio-console.entrypoints=websecure"
          - "traefik.http.routers.minio-console.tls=true"
          - "traefik.http.routers.minio-console.service=minio-console-svc"
          - "traefik.http.services.minio-console-svc.loadbalancer.server.port=9001"
          # MinIO API
          - "traefik.http.routers.minio-api.rule=Host(`s3-api.anantafatur.dev`)"
          - "traefik.http.routers.minio-api.entrypoints=websecure"
          - "traefik.http.routers.minio-api.tls=true"
          - "traefik.http.routers.minio-api.service=minio-api-svc"
          - "traefik.http.services.minio-api-svc.loadbalancer.server.port=9000"
    
      minio-buckets:
        image: quay.io/minio/mc:RELEASE.2025-08-13T08-35-41Z
        container_name: minio-buckets
        depends_on:
          - minio
        networks:
          - traefik-net
        entrypoint: >
          /bin/sh -c "
          sleep 5;
          mc alias set localminio http://minio:9000 minioadmin supersecret;
          mc mb localminio/mybucket || true;
          mc admin accesskey create localminio/ --access-key accesskey123 --secret-key secretkey1234567890 || true;
          "
          
    networks:
      traefik-net:
        external: true

docker-compose.yaml

---

## Architecture Diagram

              ┌─────────────────────┐
              │   Cloudflare CDN    │
              │  (Full Strict TLS) │
              └─────────┬──────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │       Traefik       │
              │  Docker Container   │
              │  EntryPoints: 80/443│
              └─────┬─────────┬─────┘
                    │         │
            ┌───────▼──────┐  │
            │ MinIO Console│  │
            │   port 9001  │  │
            └──────────────┘  │
                              │
                      ┌───────▼───────┐
                      │   MinIO API   │
                      │   port 9000   │
                      └───────────────┘
    

Traefik terminates Cloudflare TLS using the Cloudflare origin certificate. All traffic to MinIO is routed internally over the `traefik-net` Docker network. Simple flow, no magic.

---

## Resource Usage (Idle State)

The following Docker stats were captured while the environment was idle, with no active traffic routed through Traefik. This is included only to show baseline resource usage, not as a performance benchmark.

    NAME      CPU %     MEM USAGE / LIMIT     MEM %
    minio     0.02%     84.37MiB / 1.866GiB   4.42%
    traefik   0.00%     17.15MiB / 1.866GiB   0.90%
    

docker stats

At idle, Traefik uses very little CPU and memory. Nothing shocking here. Which is good. Means it’s not burning resources while just sitting there doing nothing.

---

## Conclusion

So far, this PoC works as expected. Routing is correct, TLS terminates cleanly with Cloudflare Full Strict mode, and everything stays neatly inside the Docker network.

No benchmarking or load testing done yet. That one can come later. For now, this setup is purely to confirm that Traefik can do its job without giving unnecessary headache. For that purpose, it’s good enough already.
