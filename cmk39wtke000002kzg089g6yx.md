---
title: "생각하면서 개발해야 하는 이유"
seoTitle: "singleton"
seoDescription: "singleton"
datePublished: Wed Jan 07 2026 00:21:43 GMT+0000 (Coordinated Universal Time)
cuid: cmk39wtke000002kzg089g6yx
slug: 7iod6rcb7zwy66m07iscioqwnouwno2vtoyvvcdtlzjripqg7j207jyg

---

## \[Retrospective\] The Dangers of Shared State: Why Global Variables in Singletons Are a Nightmare

### Background

Hi, I’m a backend engineer currently working on building and maintaining LMS (Learning Management System) solutions for universities. Due to some "lucky" (?) timing with previous team members departing shortly after I joined, I found myself leading the current projects. This post is a self-reflection—and a bit of a self-reprimand—to ensure I never repeat the same mistakes again.

To give you some context, our team’s tech stack includes:

* **Language/Framework:** Java 17, Spring Boot 3.x
    
* **Persistence/Query:** JPA, QueryDSL, MySQL, PostgreSQL
    
* **Infrastructure/Middleware:** Redis, Docker, Nginx, Ubuntu 20.04/22.04
    

---

### The Incident: A Collision of State

The issue arose during the process of passing classroom information to **Canvas LMS** via **SAML authentication**. Specifically, the problem occurred when trying to include metadata about the specific classroom a user intended to access within the SAML payload.

**The Initial (Flawed) Approach:** My initial thought was: *"When a user requests the SAML login URL, let's capture the target classroom info, store it in an object, and then use that data to authenticate and redirect the user."*

However, I hit a roadblock. There was no straightforward way to retrieve that stored information when Canvas sent the callback request back to our server. Since Canvas didn't even provide identifying information about which user was making the request at that specific stage, I couldn't simply persist it in the database and query it later.

Under pressure, I made a decision I now deeply regret—a decision born from not yet having "felt" the dangers of stateful Singletons in my bones: **I decided to store these user-specific details in a global variable (static/class-level field) within a Singleton bean.**

### The Symptom: It Works on My Machine

Predictably, the logic worked perfectly during local development. Since I was the only one testing, there were no concurrent requests to expose the horror of global variables. This was also a failure of our QA process; we didn't adequately simulate high-concurrency scenarios.

The nightmare finally manifested during a **live client demonstration**. As multiple stakeholders logged in simultaneously, they began seeing other people's names and accessing classrooms belonging to different users. It was a catastrophic "identity swap" scenario that I’d rather forget.

---

### The Fix: From Dirty Patches to Proper State Management

**Attempt 1 (The Naive Fix):** In a panic, my first thought was to keep the global variable but immediately nullify/initialize it after the redirection. This didn't solve the problem; it only reduced the error frequency. If another request hit the server in the millisecond before the variable was cleared, the same collision occurred. Concurrency is not something you can "race" against with manual overrides.

**Attempt 2 (The Robust Solution):** The final and correct approach was to leverage **Client-side State**. I chose to store the encrypted classroom metadata in a **Cookie**. When the flow redirected to the point where Canvas needed the data, the server could retrieve it directly from the user's browser request. This effectively decoupled the state from the server's memory and tied it to the individual user's session. This completely resolved the identity-swapping issue.

---

### Lessons Learned

They say experience is a hard teacher because she gives the test first and the lesson afterward. While this was a "good" experience in the sense that I will now be obsessively cautious about state management and Singleton design, I realize that with just a bit more deep thinking, I could have avoided such a fundamental architectural flaw.

If you’ve experienced something similar, you have my deepest sympathies. If you haven't—let this be a warning: **Be extremely vigilant when handling state within Singleton objects.** In a multi-threaded environment, global variables are a ticking time bomb.