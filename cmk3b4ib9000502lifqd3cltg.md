---
title: "From Monolithic to Strategic Decoupling: Ensuring 24/7 Availability in University LMS"
seoTitle: "architecture"
seoDescription: "architecture"
datePublished: Wed Jan 07 2026 00:55:41 GMT+0000 (Coordinated Universal Time)
cuid: cmk3b4ib9000502lifqd3cltg
slug: from-monolithic-to-strategic-decoupling-ensuring-247-availability-in-university-lms
tags: refactoring, architecture

---

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767747218405/369b8974-4a63-4d86-9e20-58fc885d41ae.png align="center")

This is the architecture of the solution we previously deployed to our clients. Most of our clients were universities (occasionally graduate schools, but the requirements were largely similar). These clients generally fell into three categories: those who outsourced server management to a third party, those with an in-house IT team managing the servers, and those with no dedicated IT team or physical server infrastructure at all.

Despite these three categories, they all shared a common constraint: an environment where **scaling out servers at will was impossible**, primarily due to budget limitations. Given these constraints, our initial delivery strategy was to request the highest-specification server the university could provide and deploy all services onto that single instance. Consequently, all business logic was concentrated within a single monolithic backend server.

Initially, this didn't pose significant issues. Since the solution was typically managed by the university's IT team or relevant departments after delivery, we didn't have much exposure to the operational challenges. However, the friction in this architecture began to surface when a client signed a larger contract that included a full year of operational support for their freshman class.

### Requirements

---

As the project included a commitment to develop additional requirements, the university requested features like QR check-ins for mid-semester events (festivals, faculty consultations, etc.). The challenge arose from the nature of an LMS (Learning Management System) within a university setting: students can engage in coursework at any time.

Therefore, the server had to be constantly available to handle attendance data. We realized that shutting down the server to deploy new features could potentially compromise the **data integrity and reliability** of attendance records.

After some deliberation, we opted for a manual **Blue/Green deployment** strategy. We would update half of the running containers first, followed by the remaining half. While this method didn't cause any major incidents—aside from being incredibly tedious and nerve-wracking every time—it allowed us to navigate the semester successfully.

Once the semester concluded and the final grades were delivered, I began to reconsider the architecture for future deployments.

### Decoupling Mission-Critical Functions

---

Based on the insights gained during operations, I decided to isolate the functions that required constant uptime. These primarily included the attendance module and the logic for aggregating assignment and quiz scores. However, since the assignment and quiz logic was subject to frequent changes, I decided not to group it with the attendance module.

Attendance data required **zero-tolerance for error**. Even if a discrepancy occurred, we needed a system that allowed for rapid feedback and correction via the database where all records were stored.

I further decoupled the servers related to attendance. Since the attendance server needed to be constantly available when requesting data, I designed separate servers to collect video viewing data and Zoom session information.

### Lessons from Decoupling

---

To be clear, decoupling isn't purely beneficial. It adds complexity to areas that were previously easier to develop within a single server. Our decision wasn't driven by a blind desire to follow the MSA (Microservices Architecture) hype; it was a pragmatic choice to reduce the operational burden for future projects. Since we rarely use more than two servers, managing an increasing number of containers could have led to overhead, especially since Kubernetes was overkill for our scale.

However, the conclusion so far is one of **great satisfaction**. While there are downsides—such as having to fetch data via external API communication—the overall service feels significantly more robust and stable. In the next post, I will review the performance and speed aspects of this change.