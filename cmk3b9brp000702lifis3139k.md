---
title: "Fixing Frozen Timestamps in Docker & Spring: A Lesson in Bean Lifecycle"
seoTitle: "dockerfile"
seoDescription: "dockerfile"
datePublished: Wed Jan 07 2026 00:59:26 GMT+0000 (Coordinated Universal Time)
cuid: cmk3b9brp000702lifis3139k
slug: fixing-frozen-timestamps-in-docker-and-spring-a-lesson-in-bean-lifecycle
tags: docker, dockerfile

---

## The Issue

---

I discovered a bug where the dates in the user activity stream tab—which summarizes the last three days of activity—were stuck at the exact time the Docker container was started.

## Troubleshooting

---

### Attempt 1: Syncing Container Time with Local Server Time

Initially, I tried to synchronize the container's `localtime` with the host server's `localtime`. When the issue occurred, I ran the `date` command on the server, and it correctly displayed KST. However, the time inside the container was still being output in UTC. My Docker Compose configuration was set to mount `/etc/localtime:/etc/localtime`.

### Attempt 2: Syncing via Timezone File

After some research, I found feedback suggesting that syncing via `/etc/timezone` is often more reliable than using `/etc/localtime`. I applied this change immediately, but the container output remained in UTC.

### Attempt 3: Installing `tzdata` in the Dockerfile

To ensure the container could handle time dynamically, I modified the Dockerfile to install the `tzdata` package. I configured it to copy the `zoneinfo` of the desired timezone to the container's `/etc/localtime` and overwrite `/etc/timezone` with the specific TZ value.

Dockerfile

```dockerfile
ENV TZ=Asia/Seoul
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/$TZ /etc/localtime && \
    echo $TZ > /etc/timezone
```

After starting the container with this new image, the internal system time finally matched the server time.

## Re-evaluating the Root Cause

---

Despite fixing the system time, the API was still returning a fixed timestamp. I kept wondering, "How can `LocalDateTime`, which is supposed to fetch the system time, be a constant value?"

The fact that even the nanoseconds were identical was the smoking gun. It suddenly dawned on me: when defining a `DateUtil` class for global use, I had declared the date variables as `static final`.

Feeling a bit foolish, I removed the `final` keywords and tried registering the class as a Spring `@Component` (Bean). But of course, the value was still fixed—the instance variable held the value initialized when the singleton bean was first created. It was a lapse in judgment during debugging. After some self-reflection, I refactored the code to either use a standard class or initialize the time at the exact point of execution.

After running all test cases and deploying to the dev server, I confirmed the API works perfectly.

## Retrospective

---

I'm documenting this to remind myself to be more deliberate during development. Following the recent login issue, I really need to break the habit of reflexively registering everything as a Bean without considering its lifecycle.