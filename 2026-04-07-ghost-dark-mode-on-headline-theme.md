---
title: Ghost Dark Mode on Headline Theme
slug: ghost-dark-mode-on-headline-theme
date_published: 2026-04-07T07:01:14.000Z
date_updated: 2026-04-07T15:14:41.000Z
excerpt: Dark mode added to my Ghost blog on Headline theme, click 🌙 / ☀️ to switch themes for comfy reading!
---

Moved to a new room, and the lighting is darker than usual. Working on my laptop made me realize… my blog was too bright! I love dark mode on my laptop, so why not on my blog too?

Some sites support it automatically, but my personal blog didn’t. So I asked ChatGPT to help me add dark mode. Now, users can read articles comfortably in darker environments, and I can switch themes anytime!

> **Note:** Only affects code injection; safe, but backup Ghost files first at `/var/lib/ghost`.

### Try It Out!

Click the 🌙 button at the bottom-right corner to toggle dark/light mode. ✨

---

### Site Header (CSS Injection)

    <style>
    :root {
      --bg-color: #fefefe;
      --text-color: #222222;
      --heading-color: #333333;
      --code-bg: #f7f7f7;
      --code-text: #222222;
    }
    
    body.dark-mode {
      --bg-color: #1a1a1a;
      --text-color: #e5e5e5;
      --heading-color: #f5f5f5;
      --code-bg: #2a2a2a;
      --code-text: #e5e5e5;
    }
    
    body {
      background-color: var(--bg-color);
      color: var(--text-color);
      transition: background-color 0.3s ease, color 0.3s ease;
    }
    
    a {
      color: inherit;
      transition: color 0.3s ease;
    }
    
    body.dark-mode h1,
    body.dark-mode h2,
    body.dark-mode h3,
    body.dark-mode h4,
    body.dark-mode h5,
    body.dark-mode h6,
    body.dark-mode .gh-post-title {
      color: var(--heading-color) !important;
    }
    
    body.dark-mode .gh-author-name,
    body.dark-mode .gh-author-name-list {
      color: #b0b0b0 !important;
    }
    
    body.dark-mode pre,
    body.dark-mode code {
      background-color: var(--code-bg) !important;
      color: var(--code-text) !important;
    }
    
    body.dark-mode .language-python {
      color: var(--code-text) !important;
    }
    
    body.dark-mode .gh-head,
    body.dark-mode .gh-outer {
      background-color: var(--bg-color) !important;
      color: var(--text-color) !important;
    }
    
    body.dark-mode .gh-head *,
    body.dark-mode .gh-outer * {
      color: var(--text-color) !important;
    }
    
    body.dark-mode .gh-burger,
    body.dark-mode .gh-nav-toggle {
      background-color: transparent !important;
    }
    
    body.dark-mode .gh-burger {
      background-color: transparent !important;
    }
    
    body.dark-mode .gh-burger::before,
    body.dark-mode .gh-burger::after {
      background-color: var(--text-color) !important;
    }
    
    body.dark-mode .gh-head-actions .gh-search {
      color: var(--text-color) !important;
    }
    
    body.dark-mode .gh-head-actions .gh-search svg {
      stroke: var(--text-color) !important;
    }
      
    * {
      transition: background-color 0.3s ease, color 0.3s ease, border-color 0.3s ease;
    }
    
    #darkModeToggle {
      position: fixed;
      bottom: 24px;
      right: 24px;
      width: 60px;
      height: 60px;
      padding: 0;
      background: rgba(255,255,255,0.1);
      backdrop-filter: blur(14px);
      -webkit-backdrop-filter: blur(14px);
      border-radius: 16px;
      border: 1px solid rgba(255,255,255,0.2);
      display: flex;
      justify-content: center;
      align-items: center;
      box-shadow: 0 12px 30px rgba(0,0,0,0.25);
      color: #fff;
      font-size: 28px;
      cursor: pointer;
      z-index: 99999;
      overflow: visible;
      transition: all 0.25s ease;
    }
    
    #darkModeToggle:hover {
      transform: translateY(-3px) scale(1.05);
      box-shadow: 0 16px 35px rgba(0,0,0,0.35);
    }
    
    #darkModeToggle .ripple {
      position: absolute;
      border-radius: 50%;
      transform: scale(0);
      animation: ripple 0.6s ease-out;
      pointer-events: none;
    }
    
    @keyframes ripple {
      to {
        transform: scale(4);
        opacity: 0;
      }
    }
    </style>

---

### Site Footer (Button & Script Injection)

    <button id="darkModeToggle">🌙</button>
    
    <script>
      const toggleBtn = document.getElementById('darkModeToggle');
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)');
    
      function setTheme(mode) {
        if (mode === 'dark') {
          document.body.classList.add('dark-mode');
          toggleBtn.textContent = '☀️';
          localStorage.setItem('theme', 'dark');
        } else {
          document.body.classList.remove('dark-mode');
          toggleBtn.textContent = '🌙';
          localStorage.setItem('theme', 'light');
        }
      }
    
      const savedTheme = localStorage.getItem('theme');
      if (savedTheme) {
        setTheme(savedTheme);
      } else {
        setTheme(prefersDark.matches ? 'dark' : 'light');
      }
    
      toggleBtn.addEventListener('click', (e) => {
        const circle = document.createElement('span');
        circle.classList.add('ripple');
    
        const rect = toggleBtn.getBoundingClientRect();
        const size = Math.max(rect.width, rect.height);
        circle.style.width = circle.style.height = size + 'px';
        circle.style.left = (e.clientX - rect.left - size / 2) + 'px';
        circle.style.top = (e.clientY - rect.top - size / 2) + 'px';
    
        circle.style.backgroundColor = document.body.classList.contains('dark-mode')
          ? 'rgba(255,255,255,0.25)'
          : 'rgba(0,0,0,0.15)';
    
        toggleBtn.appendChild(circle);
        setTimeout(() => circle.remove(), 600);
    
        const isDark = document.body.classList.contains('dark-mode');
        setTheme(isDark ? 'light' : 'dark');
      });
    
      prefersDark.addEventListener('change', (e) => {
        if (!localStorage.getItem('theme')) {
          setTheme(e.matches ? 'dark' : 'light');
        }
      });
    </script>
    
    <script>
      window.addEventListener("load", function () {
        const nav = document.querySelector(".nav");
    
        setTimeout(() => {
          const dropdown = document.querySelector(".gh-dropdown");
          const more = document.querySelector(".nav-more-toggle");
    
          if (nav && dropdown) {
            dropdown.querySelectorAll("li").forEach(li => {
              nav.appendChild(li);
            });
            dropdown.remove();
          }
    
          if (more) more.remove();
        }, 100);
      });
    </script>

---

### Closing

That’s it! 🎉 Dark mode is now live on my blog. Whether you’re reading in a bright room or a dark one, you can switch themes anytime using the 🌙 / ☀️ button.
