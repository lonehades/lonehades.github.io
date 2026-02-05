---
layout: post
title: "AI vs Korail: The Siege of the Iron Fortress"
subtitle: "When the front door is locked, dig a tunnel"
date: 2026-02-05 22:45:00 +0900
categories: [AI Agent, OpenClaw]
tags: [OpenClaw, Korail, Python, Automation]
author: Benjamin Ryu
---

Hello! Today, I want to share the chaotic yet triumphant journey of building a **"KTX Auto-Reservation Bot"** with my AI assistant (OpenClaw/Zhuge Liang).

### 1. The Prologue: The Iron Fortress
It started simply. "I need a ticket to Seoul tomorrow, but it's sold out. Let's make a bot."
I commanded my AI agent:
*"Hey, open Chrome, go to Korail, and book a ticket."*

But Korail was not to be trifled with.
As soon as the AI launched the browser, a warning popped up:
> **⛔ "Security Program Detected: Browser Automation Blocked"**

My AI tactician said: *"Sire, the walls are too high. We cannot enter through the main gate."*

### 2. The Pivot: Digging a Tunnel (Python)
Since the direct attack failed, we sought a detour.
Instead of launching a browser, we decided to use **Python code** to talk directly to the server. Thankfully, some pioneers had already built a library called `korail2`.

*"Good, use this to login and search!"*
Login success! Search success! Reservation success!
...or so we thought.

### 3. The Crisis: The Bot's Rebellion (Cannot Cancel!)
I told the bot to cancel the test reservation, but it froze.
> **❌ Error: Extra data: line 1 column 5...**

The Korail server sent a response the library couldn't understand, and it crashed. The ticket was booked, but we couldn't cancel it. Money was at risk!

### 4. The Surgery: Patching the Heart
My AI said: *"Sire, we must perform surgery on the tool's heart (source code)."*

We opened up the library guts (`korail2.py`), found the code that was choking on the new server response format, and patched it to forcefully ignore the error and proceed.

> *"Surgery complete. Retrying cancellation... Success! The ticket is gone!"*

### 5. The Climax: The 5-Minute Watchman
Obstacles cleared. Now for the **Infinite Loop**.
*   **Target:** Tomorrow 14:00~16:00 Daejeon -> Seoul
*   **Interval:** Check every 5 minutes
*   **Alert:** Telegram me immediately!

While I was distracted, *Ding!* 🔔
> **🎉 [Reservation Success!] Daejeon~Seoul KTX 14:35**
> **🚨 Pay quickly via KorailTalk!**

And thus, we secured a sold-out ticket. (Although the bot once booked a ticket from 'Seodaejeon' by mistake and got scolded, but we fixed that logic too. 😂)

### Conclusion
If it doesn't work, make it work. If the web is blocked, use code.
With AI, even Korail holiday booking isn't scary! (Maybe?)
