---
title: "Scaling Story: How We Solved a Critical Bottleneck in Our Rails Monolith"
date: 2025-08-25
draft: false
description: "A war story about evolving a monolithic architecture to an event-driven microservice to handle high-volume transaction processing."
tags: ["Architecture", "Ruby on Rails", "Microservices", "Kafka", "Scalability"]
---

Every successful SaaS platform eventually faces the same “good problem to have”: your application is so popular that its own growth becomes the biggest threat to its stability. This is the story of how our team at Jurnal (Mekari) tackled a major performance bottleneck in our core Ruby on Rails monolith, and how we evolved our architecture iteratively to support our rapidly growing user base.<!--more-->

### The Scene: A Successful Monolith

When I was at Jurnal, our core product was a feature-rich accounting platform built on a classic, robust stack: a Ruby on Rails monolith backed by a MySQL database on AWS RDS, with Redis and Sidekiq for caching and background job processing. All traffic came through an AWS Load Balancer and an API Gateway (Kong) to manage our growing third-party ecosystem.

This architecture had served us well, allowing us to build and ship features quickly. But one feature, in particular, was a victim of its own success.

> **IMAGE/ILLUSTRATION:**

### The Bottleneck: A Feature Too Popular for Its Own Good

Our data import tool was a key driver of new business. It allowed customers to easily migrate their entire accounting history from other systems. As our popularity grew, so did the size and frequency of these imports.

Soon, we started seeing a recurring, application-wide problem. Whenever a large import was in progress, the entire platform would become sluggish. Reports would time out, dashboards wouldn't load, and the user experience would degrade for everyone.

Using monitoring tools like Datadog, the team quickly diagnosed the issue. The import process was generating an enormous volume of write operations on our central `account_transactions` table. This single table was the core of the entire accounting system, and the intense database contention and locking during imports were starving all other application processes of resources.

### The First Fix: Background Jobs Aren't a Silver Bullet

Our first, logical step was to move the work off the main web request thread. We refactored the import logic into a **Sidekiq** background job. This provided an immediate user-facing improvement—the UI was no longer frozen during an import.

However, we soon realized we had only moved the bottleneck, not solved it. Our Sidekiq workers, running on the same shared infrastructure, were still creating massive spikes in CPU and database load. The intense write activity on the `account_transactions` table was now slowing down all other critical background jobs, like sending invoices or generating financial reports.

At the same time, the team implemented database-level optimizations. We added critical **indexing** to speed up queries and set up a **Master-Slave replication** model. This allowed us to direct all read-heavy traffic (like reports) to the slave databases, freeing up the master database to focus on writes. This gave us some breathing room, but the core write contention issue on the master database remained.

### The Real Solution: Decoupling with an Event-Driven Microservice

It was clear that true workload isolation was the only way forward. We decided to strategically decouple the entire transaction creation process—not just for imports, but for every transaction created via our web app, mobile app, or API.

We shifted to an asynchronous, event-driven architecture specifically only for `account_transactions` domain.

1.  **The Event:** Any time a transaction needed to be created, the Rails monolith would now simply validate the request and publish a `CreateTransaction` event to a **Kafka** topic. This was an incredibly fast and lightweight operation that didn't touch the main database.

2.  **The Microservice:** We built a new, dedicated **Java microservice** whose sole responsibility was to be the "owner" of the transaction ledger. It consumed events from the Kafka topic, performed the necessary calculations, and was the only service with permission to write to the `account_transactions` table.

3.  **The Scalable Database:** This new service was designed for the future. Its database was built with **sharding** in mind, partitioning the massive transaction table by customer ID. This distributed the write load across multiple database servers, finally solving the write bottleneck at its root.

> **IMAGE/ILLUSTRATION:**

### Key Takeaways

This project was a powerful lesson in iterative problem-solving and architectural evolution.

* **Background jobs are for deferring work, not for isolating it.** When a process is resource-intensive enough to impact shared resources, it needs true isolation.

* **Database optimization is crucial, but it has its limits.** Indexing and read replicas are powerful tools, but they can't solve a fundamental write bottleneck in a high-volume system.

* **Event-driven microservices are a powerful tool for scaling a monolith.** By strategically decoupling our most intensive workload, we not only solved the performance issue but also made the entire system more resilient and independently scalable.

Ultimately, this journey from a struggling monolith to a hybrid architecture was a testament to how a team can tackle scaling challenges head-on, delivering a more stable and performant platform for all users.