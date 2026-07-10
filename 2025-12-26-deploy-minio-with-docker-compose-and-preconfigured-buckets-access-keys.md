---
title: Deploy MinIO with Docker Compose and Preconfigured Buckets & Access Keys
slug: deploy-minio-with-docker-compose-and-preconfigured-buckets-access-keys
date_published: 2025-12-25T22:42:34.000Z
date_updated: 2026-02-15T16:48:59.000Z
excerpt: Docker Compose makes running MinIO easy. Learn how to automatically create buckets and access keys.
---

> Prefer watching instead of reading?
> The video version is available [here](https://youtu.be/w_N8nMXFrrE) and you can check out the GitHub repo [here](https://github.com/anantafaturdev/yt-anantafatur/tree/main/minio).

I run Ghost using Docker volumes. At that point, backups are mandatory. Docker volumes are stateful and not portable, so losing the volume means losing the site.

After some digging, I ended up using Duplicati because it can back up Docker volumes and push them to S3-compatible storage. Instead of relying solely on managed S3, I wanted to apply the 3-2-1 backup principle properly. That required a self-hosted object storage as one of the backup targets.

This is where MinIO fits. Below is the Docker Compose setup I use to run MinIO and bootstrap it automatically. The goal is simple: start MinIO, create a bucket, and provision application credentials without touching the web UI.

The setup uses two containers. The first one runs the MinIO server and persists data using a Docker volume. The second container runs the MinIO client (`mc`) and acts as a bootstrapper. It waits until MinIO is healthy, then creates a bucket and generates an access key and secret key for application use. Everything is done programmatically, so the setup is reproducible.

MinIO exposes two endpoints. Port **9001** is used for the web console and administrative access. Port **9000** is the S3-compatible API endpoint and must be used by applications such as Duplicati.

    version: "3.9"
    
    services:
      minio:
        image: quay.io/minio/minio:RELEASE.2025-09-07T16-13-09Z
        container_name: minio
        restart: always
        ports:
          - "9000:9000"
          - "9001:9001"
        command: ["server", "--console-address", ":9001", "/data"]
        volumes:
          - data-minio:/data
        environment:
          MINIO_ROOT_USER: ${MINIO_ROOT_USER}
          MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
        healthcheck:
          test: ["CMD", "curl", "-f", "${MINIO_ENDPOINT}/minio/health/live"]
          interval: 5s
          retries: 5
          timeout: 2s
    
      minio-buckets:
        image: quay.io/minio/mc:RELEASE.2025-08-13T08-35-41Z
        container_name: minio-buckets
        depends_on:
          minio:
            condition: service_healthy
        restart: "on-failure"
        entrypoint: >
          /bin/sh -c "
          set -e;
    
          echo '>> Setting MinIO alias';
          mc alias set localminio ${MINIO_ENDPOINT} ${MINIO_ROOT_USER} ${MINIO_ROOT_PASSWORD};
    
          echo '>> Creating bucket';
          mc mb localminio/${MINIO_BUCKET_NAME} || true;
    
          echo '>> Creating app access key';
          mc admin accesskey create localminio/ --access-key ${MINIO_ACCESS_KEY} --secret-key ${MINIO_SECRET_KEY} || true;
    
          echo '>> Done';
          "
    
    volumes:
      data-minio:

The `.env` file supplies credentials and names. `MINIO_ENDPOINT` points to the Docker service name, not `localhost`. The generated access key and secret key are intended for applications like Duplicati, not for administrative use.

    MINIO_ROOT_USER=
    MINIO_ROOT_PASSWORD=
    
    MINIO_BUCKET_NAME=
    MINIO_ENDPOINT=http://minio:9000
    
    #openssl rand -hex 12 | cut -c1-20
    MINIO_ACCESS_KEY=
    #openssl rand -base64 48 | tr -d '\n' | cut -c1-40
    MINIO_SECRET_KEY=

---

After the stack is running, I checked the resource usage using `docker stats` to get a rough idea of its footprint.

    CONTAINER ID   NAME    CPU %   MEM USAGE / LIMIT   MEM %   NET I/O         BLOCK I/O    PIDS
    7ecc9b2d2d0c   minio   1.99%   85.2MiB / 2GiB      4.16%  38.9MB / 42MB    0B / 116MB   9
    

At idle, MinIO stays under 100 MB of memory with minimal CPU usage. This makes it reasonable to run on small VPS or homelab setups. Resource usage increases during backup or restore operations, but idle overhead remains low.

---

Recent MinIO versions no longer expose access key creation cleanly through the web UI. Creating application credentials via `mc admin accesskey create` keeps the setup fully automated. This also makes it easier to rebuild the stack or rotate credentials later without manual steps.

Once MinIO is running, Duplicati can be configured to use it as an S3-compatible backend by pointing to the API endpoint on port 9000 and supplying the generated bucket name, access key, and secret key.

If MinIO needs to be exposed remotely, it can be placed behind Cloudflared. On the free tier, Cloudflare Tunnel enforces a 100 MB upload size limit per request. If backup chunks exceed that limit, either reduce Duplicati’s chunk size or use a VPN-based approach such as WireGuard or Tailscale instead.

References:
[https://pliutau.com/local-minio-docker-compose-buckets/](https://pliutau.com/local-minio-docker-compose-buckets/)
[https://www.lukmanlab.com/membuat-accesskey-dan-secretkey-minio-terbaru/](https://www.lukmanlab.com/membuat-accesskey-dan-secretkey-minio-terbaru/)
