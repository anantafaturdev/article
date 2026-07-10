---
title: How to Receive Emails from Your Own Domain (for Free)
slug: how-to-receive-emails-from-your-own-domain-for-free
date_published: 2025-12-24T11:50:30.000Z
date_updated: 2025-12-25T22:43:24.000Z
excerpt: Learn how to receive emails on your own domain for free using Cloudflare Email Routing or ImprovMX.
---

After buying a domain, it’s always exciting to think about having your own email like `hi@anantafatur.dev`. It just feels… professional, right? 😎 But the question is: how do you actually receive emails on your own domain without spending a lot?

Good news: there are quick and free solutions that work really well. I’ll share the two easiest ways I know.

---

## 1. Cloudflare Email Routing (Free & Easy)

If you’re already using Cloudflare for your domain, this is probably the simplest option:

- First, **add your domain to Cloudflare** by changing the name servers at your domain registrar.
- Then, go to **Email Routing** in Cloudflare.
- Set it up so that all emails like `*@yourdomain.com` get forwarded to your personal email.

Basically, any email sent to your domain will end up in your normal inbox automatically.

The best part? It’s **totally free**, no limits for regular personal use.

> ⚠️ Note: This method **only allows you to receive emails**, not send them from your custom domain.

---

## 2. ImprovMX (No Need to Move Domain)

Not a fan of moving your domain to Cloudflare? No problem. There’s [ImprovMX](https://improvmx.com/) — a service that lets you forward emails from your domain to your personal inbox.

Here’s how it works:

1. Register with your domain + your personal email.
2. Follow the setup instructions on their dashboard. They will give you **DNS records** that you need to add at your domain provider.
3. Once DNS propagates, emails to your domain start arriving in your personal inbox automatically.

ImprovMX is also free for basic usage, but they have limits if you need advanced features. Check their pricing if you want to scale.

> ⚠️ Note: Just like Cloudflare Email Routing, this method **only receives emails**. You won’t be able to send emails from `hi@yourdomain.com` with this setup.

---

## Why This Works

With both methods, you don’t need to manage your own mail server. No Postfix, no Dovecot, no headaches with spam filters. You just get the professional-looking email you wanted, and it lands in your inbox reliably.
![](__GHOST_URL__/content/images/2025/12/image-1.png)
---

💡 **Pro tip:**
If you’re lazy like me, I didn’t include screenshots here. For visual step-by-step guides, you can check YouTube, or maybe I’ll make a video tutorial on my channel later 😎.

---

By following either of these methods, you can finally have emails from your own domain, looking professional and without paying a dime. Just keep in mind — **this is for receiving emails only, not sending**.
