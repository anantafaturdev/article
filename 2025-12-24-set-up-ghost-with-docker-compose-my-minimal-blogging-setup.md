---
title: Set Up Ghost with Docker Compose – My Minimal Blogging Setup
slug: set-up-ghost-with-docker-compose-my-minimal-blogging-setup
date_published: 2025-12-23T21:29:38.000Z
date_updated: 2026-02-22T16:35:40.000Z
excerpt: Learn how to run Ghost on a cheap VPS using Docker Compose. Persist content with volumes, configure SMTP for notifications, and deploy quickly with minimal setup. Perfect for personal tech blogs or homelabs.
---

> Prefer watching instead of reading?
> The video version is available [here](https://youtu.be/qJV4spRXBPg) and you can check out the GitHub repo [here](https://github.com/anantafaturdev/yt-anantafatur/tree/main/ghost).

So, I want to break into DevOps. Sounds cool, right? But here’s the catch: it’s **not entry-level**. Most people start from dev or ops roles, gain experience, then move into DevOps. Me? I wanted to jump straight in. 😅

To prove I can do it (with a senior supervisor watching over me, of course), I decided to start writing **technical docs on my own website**.

Years ago, I tried Ghost for blogging. And, well… perfectionism struck hard. Every draft felt “not perfect yet,” and I’d leave it there forever. Drafts piled up like unfinished code. This time, I said, **screw it**—messy writing is better than no writing. Nobody reads my blog anyway… yet.

---

## Finding a Cheap VPS (because my homelab is noisy af)

My homelab lives in my bedroom. Fun fact: it **shuts down every night** because it’s louder than a rocket launch 😅. So, I needed a cheap VPS to run Ghost 24/7.

Enter [hostdata.id NAT VPS](https://hostdata.id/nat-vps-indonesia/) — **30K/month**, 2GB RAM, 1 vCPU, 25GB SSD. Small but mighty enough for my Ghost instance.

Bench.sh says:

     CPU Model          : Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz
     CPU Cores          : 1 @ 1572.582 MHz
     CPU Cache          : 35840 KB
     AES-NI             : ✓ Enabled
     VM-x/AMD-V         : ✓ Enabled
     Total Disk         : 24.5 GB (3.5 GB Used)
     Total Mem          : 2.0 GB (747.4 MB Used)
     System uptime      : 1 days, 20 hour 24 min
     Load average       : 0.14, 0.09, 0.07
     OS                 : Debian GNU/Linux 11
     Arch               : x86_64 (64 Bit)
     Kernel             : 4.19.0
     TCP CC             : 
     Virtualization     : OpenVZ
     IPv4/IPv6          : ✓ Online / ✓ Online
     Organization       : AS141968 PT Industri Kreatif Digital
     Location           : Jakarta / ID
     Region             : Jakarta
    ----------------------------------------------------------------------
     I/O Speed(1st run) : 50.5 MB/s
     I/O Speed(2nd run) : 65.8 MB/s
     I/O Speed(3rd run) : 74.2 MB/s
     I/O Speed(average) : 63.5 MB/s
    ----------------------------------------------------------------------
     Node Name        Upload Speed      Download Speed      Latency     
     Speedtest.net    4924.37 Mbps      412.77 Mbps         1.19 ms     
     Paris, FR        505.69 Mbps       443.70 Mbps         157.49 ms   
     Shanghai, CN     181.38 Mbps       342.45 Mbps         338.07 ms   
     Hong Kong, CN    870.14 Mbps       394.31 Mbps         43.04 ms    
     Singapore, SG    864.16 Mbps       413.19 Mbps         13.36 ms    
     Tokyo, JP        406.46 Mbps       425.12 Mbps         124.63 ms   

Honestly, I don’t care about SLA, speed, or fancy specs. This is **for starting**, not for winning any awards.

---

## Why Docker Compose? Because I’m lazy (and smart 😎)

I might migrate Ghost to a bigger VPS later. Docker Compose is perfect: move the containers + volumes + configs anywhere with minimal headache. No “install everything from scratch” nightmares.

---

## Minimal Docker Compose Setup

Here’s what I actually use:

    services:
    
      ghost:
        image: ghost:6.10.3-alpine3.23
        restart: always
        ports:
          - 8087:2368
        environment:
          database__client: mysql
          database__connection__host: db
          database__connection__user: root
          database__connection__password: 96WlUp5b/p2VbBx9
          database__connection__database: ghost
          url: https://anantafatur.dev
          mail__transport: SMTP
          mail__options__host: mail.smtp2go.com
          mail__options__port: 587
          mail__options__auth__user: XXXXXXXXXXXX
          mail__options__auth__pass:XXXXXXXXXXXX
          mail__from: XXXXXXXXXXXX <XXXXXX@XXXXXXXXXXXX.XXX>
        volumes:
          - ghost:/var/lib/ghost/content
    
      db:
        image: mysql:8.0-bookworm
        restart: always
        environment:
          MYSQL_ROOT_PASSWORD: 96WlUp5b/p2VbBx9
        volumes:
          - db:/var/lib/mysql
    
    volumes:
      ghost:
      db:

Why this works for me:

- Minimalist: no Caddy/Nginx, no analytics, no nonsense. Just **writing first**.
- Using volumes to persist Ghost content—your posts won’t vanish if the container restarts. I’ve got plans to cover backups and S3 in another article.
- Ghost is live at `http://<VPS_IP>:8087`. Easy.
- I expose it via **Cloudflared**, so I can access it with my domain without touching reverse proxy configs.

---

## Lessons Learned

1. **Start now, imperfectly.** Drafts piling up are just “art in progress.”
2. **Cheap VPS + Docker Compose** = fast way to get your tech blog online.
3. **Perfectionism is overrated.** Seriously, just write.

---

Honestly, this blog isn’t for views or awards. It’s my **proof to myself** that I can start, finish, and publish technical work. Plus, it’s a safe place to experiment, mess up, and maybe laugh at my old drafts later.
