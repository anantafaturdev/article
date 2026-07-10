---
title: How to Get a Free SMTP for Your Personal Homelab Using SMTP2GO
slug: how-to-get-a-free-smtp-for-your-personal-homelab-using-smtp2go
date_published: 2025-12-24T10:50:21.000Z
date_updated: 2025-12-25T22:44:24.000Z
excerpt: Send emails from your homelab apps for free using SMTP2GO. No credit card needed, easy domain verification, and perfect for low-volume email notifications.
---

If you’ve been running your own homelab, you know that many self-hosted apps need SMTP for sending email notifications, registration confirmations, or password resets. Paying for third-party services is always an option, but if your usage is small, why not try a free tier? In this article, we’ll show you how to set up **SMTP for free using SMTP2GO**—no credit card required!
![](__GHOST_URL__/content/images/2025/12/image-2.png)
---

## Why SMTP2GO?

SMTP2GO is a simple, beginner-friendly solution for sending emails from your own apps. Here’s why it’s perfect for personal homelab usage:

- **Free plan available** (0 cost, no credit card)
- **200 emails/day** (1,000 emails/month)
- **5 days of email reporting**
- **Ticket support**

Perfect for testing or low-volume email sending.

> Note: You need a domain that’s at least ~1 month old. This delay helps prevent spam registrations.

---

## Step 1: Register Your Free Account

1. Go to [SMTP2GO signup page](https://www.smtp2go.com/pricing/)
2. Input your email with your own domain (you can set up custom domain emails with services like Cloudflare or ImprovMX—more on that in a future article).
3. Confirm your email. Some accounts may require phone verification.

---

## Step 2: Verify Your Domain

Once logged in:

1. Go to [**Verified Senders**](https://app-us.smtp2go.com/sending/verified_senders)
2. Add your sender domain.
3. Add the **CNAME record** to your domain provider service.
4. Wait ~15 minutes for DNS propagation.
5. Once verified, your domain status will show as “verified.”

> I know screenshots make things easier… but I’m too lazy to include them here 😅. For a visual step-by-step, kindly check YouTube :D, or I might create a video on my own channel in the future.

---

## Step 3: Create SMTP Users

Now that your domain is verified:

1. Go to [**SMTP Users**](https://app-us.smtp2go.com/sending/smtp_users/)
2. Click **Add SMTP User**, set a username and password.
- Tip: Split users per app to track usage easily.

And… voila! Your SMTP is ready to use **completely free**. 🎉

---

## Step 4: Configure Your Apps

Use the credentials you just created to configure your apps. You can test the SMTP with [SMTPer.net](https://www.smtper.net/) to ensure it works.

You’ll find all necessary details (SMTP server, ports, etc.) in the **SMTP User menu** on SMTP2GO.

---

## Summary

Using SMTP2GO, you can:

- Send emails from your homelab apps without spending a dime
- Easily manage multiple apps with separate SMTP users
- Verify your domain and ensure your emails are delivered properly

For personal or testing purposes, this setup is more than enough. No credit card, no hassle, just simple, free SMTP for your homelab.
