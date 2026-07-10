---
title: Bitwarden vs Vaultwarden Resource Comparison
slug: bitwarden-vs-vaultwarden-resource-comparison
date_published: 2026-04-09T05:33:32.000Z
date_updated: 2026-04-09T05:33:32.000Z
excerpt: Vaultwarden is a much lighter alternative to Bitwarden for low-spec setups. In my case, Bitwarden used around 850 MB RAM at idle, while Vaultwarden only needed about 42 MB, with a much smaller image size.
---

Last year, I implemented Vaultwarden for my previous office because it’s much lighter than Bitwarden and already covered our needs, like storing credentials and API keys. For personal use, I’ve been running Bitwarden on my homelab, so this comparison comes from actual usage, not just reading docs.

Recently, I moved my password manager from my homelab to a VPS for better uptime. My homelab is not running 24/7, and for something as critical as a password manager, that’s not ideal. The problem is the VPS itself. It has limited CPU and RAM, so I needed to optimize resource usage. That’s where I decided to migrate from Bitwarden to Vaultwarden.

This post focuses only on resource usage. No feature comparison, no deep security discussion. Just what I observed from running both.

At the time of testing, I used:
Bitwarden (lite) version 2026.3.2
Vaultwarden version 1.35.4-alpine

---

### Bitwarden Setup

Here is the Docker Compose I used for Bitwarden:

    services:
      bitwarden-app:
        depends_on:
          - bitwarden-mariadb
        env_file:
          - .env
        image: ghcr.io/bitwarden/lite:2026.3.2
        restart: always
        ports:
          - "80:8080"
        volumes:
          - bitwarden-app:/etc/bitwarden
    
      bitwarden-mariadb:
        image: mariadb:10
        restart: always
        env_file:
          - .env
        environment:
          MARIADB_USER: ${MARIADB_USER}
          MARIADB_PASSWORD: ${MARIADB_PASSWORD}
          MARIADB_DATABASE: ${MARIADB_DATABASE}
          MARIADB_RANDOM_ROOT_PASSWORD: "true"
        volumes:
          - bitwarden-mariadb:/var/lib/mysql
    
    volumes:
      bitwarden-app:
      bitwarden-mariadb:
    

Bitwarden runs with two containers. One for the application and one for MariaDB. Even using the lite image, the total size is still quite large. The Bitwarden image is around 1.75 GB and MariaDB adds another 331 MB, so in total it’s about 2.08 GB.

When I checked `docker stats` in idle condition, the Bitwarden app container was using around 771 MB of RAM with a small but steady CPU usage. The MariaDB container added about 95 MB. So overall, it’s close to 850 MB RAM just sitting idle.

### Vaultwarden Setup

Now let’s compare that with Vaultwarden.

    services:
      vaultwarden-app:
        image: vaultwarden/server:1.35.4-alpine
        container_name: vault-app
        restart: always
        environment:
          DOMAIN: ${DOMAIN}
          SIGNUPS_ALLOWED: ${SIGNUPS_ALLOWED}
          INVITATIONS_ALLOWED: ${INVITATIONS_ALLOWED}
          ADMIN_TOKEN: ${ADMIN_TOKEN}
        volumes:
          - /root/production/vault/vault-data/:/data/
        ports:
          - 127.0.0.1:8003:80
    

Vaultwarden only uses a single container. It relies on SQLite, so there’s no separate database service. I mounted the data directory to the host to make backups easier, but that doesn’t affect runtime usage.

The image size is only around 168 MB, which is drastically smaller than Bitwarden. Looking at `docker stats`, the container was using around 42 MB of RAM and almost zero CPU while idle. That’s the total usage, since there’s no extra container behind it.

---

### Final Thoughts

After migrating, the difference was obvious. Bitwarden used around 850 MB RAM in idle state, while Vaultwarden dropped to about 42 MB, more than 90 percent reduction. Disk usage also went down from over 2 GB to under 200 MB, which makes a big difference on a small VPS.

For my setup, Vaultwarden is the more practical choice. It’s simpler, much lighter, and easier to manage since there’s no separate database. It still covers everything I need, so there’s no real downside for my use case.

Both setups were tested with the same data and similar uptime, all measured in idle conditions. This doesn’t mean Vaultwarden is always better, but if your priority is low resource usage, it’s clearly the better fit here.
