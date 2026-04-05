# 🚀 Java Developer Roadmap: Core → Senior (Banking-Scale Systems)

> Tự động hóa việc học Java từ cơ bản đến nâng cao, hướng tới xây dựng hệ thống lớn như ngân hàng.
> Mỗi task được nghiên cứu tự động qua OpenClaw cron job.
> **Toàn bộ nội dung nghiên cứu bằng tiếng Việt.**

## 📝 Yêu cầu nội dung nghiên cứu

Mỗi file research PHẢI viết bằng **tiếng Việt** và tập trung vào:

1. **Lý thuyết cốt lõi** — Bản chất của vấn đề là gì, tại sao tồn tại
2. **Cơ chế bên trong** — Hoạt động như thế nào ở tầng thấp, chi tiết kỹ thuật
3. **Mục tiêu thiết kế** — Người tạo ra nó muốn giải quyết vấn đề gì
4. **So sánh giải pháp** — Có những cách tiếp cận nào, cái nào hơn kém ra sao
5. **Trade-off** — Đánh đổi cái gì khi chọn giải pháp này (bảo mật vs hiệu năng, đơn giản vs linh hoạt,...)
6. **Failure modes** — Khi nào nó fail, fail như thế nào, cách phát hiện và xử lý
7. **Ứng dụng trong hệ thống ngân hàng** — Liên hệ trực tiếp với bài toán thực tế
8. **Ví dụ code Java** — Code mẫu có comment giải thích chi tiết

---

## 📋 How It Works

- Mỗi task có checkbox `[ ]` = chưa làm, `[x]` =已完成
- Cron job sẽ đọc file này, tìm task `[ ]` đầu tiên
- Sau khi hoàn thành: đánh `[x]`, thêm link file research trong `/research/`
- Mỗi lần cron chạy = **1 task duy nhất**

---

## Phase 1: Java Core Mastery (Tuần 1–4)

- [ ] **Task 01** — JVM Architecture: Class Loader, Runtime Data Areas, Execution Engine
- [ ] **Task 02** — Memory Model: Heap, Stack, Metaspace, GC Algorithms (G1, ZGC, Shenandoah)
- [ ] **Task 03** — Concurrency Foundations: Threads, ExecutorService, CompletableFuture, Virtual Threads (Java 21)
- [ ] **Task 04** — Java Memory Model (JMM): happens-before, volatile, atomic classes, locks
- [ ] **Task 05** — Collections Deep Dive: HashMap internals, ConcurrentHashMap, performance trade-offs
- [ ] **Task 06** — Functional Programming: Streams API, Optional, functional interfaces, lazy evaluation
- [ ] **Task 07** — Design Patterns in Java: Singleton, Factory, Builder, Strategy, Observer — with real examples
- [ ] **Task 08** — SOLID Principles with Java: practical code examples, refactoring before/after

## Phase 2: Spring Ecosystem (Tuần 5–8)

- [ ] **Task 09** — Spring Core: IoC Container, Bean Lifecycle, Dependency Injection (constructor vs setter)
- [ ] **Task 10** — Spring Boot Auto-configuration: how it works internally, custom starters
- [ ] **Task 11** — Spring MVC vs WebFlux: blocking vs non-blocking, when to use which
- [ ] **Task 12** — Spring Security: Authentication vs Authorization, JWT, OAuth2 flow
- [ ] **Task 13** — Spring Data JPA: entity lifecycle, caching (L1/L2), N+1 problem, batch operations
- [ ] **Task 14** — Spring Transaction Management: propagation, isolation levels, distributed transactions (XA)
- [ ] **Task 15** — Spring AOP: proxy-based AOP, @Aspect, pointcut expressions, real-world use cases
- [ ] **Task 16** — Spring Event & Messaging: ApplicationEvent, @EventListener, async events

## Phase 3: Database & Persistence (Tuần 9–11)

- [ ] **Task 17** — SQL Mastery for Java Devs: indexing strategies, query optimization, EXPLAIN plans
- [ ] **Task 18** — Connection Pooling: HikariCP internals, tuning parameters, monitoring
- [ ] **Task 19** — Database Migration: Flyway vs Liquibase, versioned migrations, rollback strategies
- [ ] **Task 20** — NoSQL for Java: MongoDB with Spring Data, Redis with Lettuce/Jedis — when and why
- [ ] **Task 21** — CQRS Pattern: Command Query Responsibility Segregation, Event Sourcing basics

## Phase 4: System Design & Architecture (Tuần 12–15)

- [ ] **Task 22** — Microservices Architecture: service decomposition, API Gateway, Service Discovery
- [ ] **Task 23** — Distributed Systems Fundamentals: CAP theorem, eventual consistency, consensus (Raft)
- [ ] **Task 24** — Message Queues: Apache Kafka deep dive — partitions, consumer groups, exactly-once semantics
- [ ] **Task 25** — Caching Strategies: cache-aside, write-through, write-behind, cache invalidation patterns
- [ ] **Task 26** — Circuit Breaker & Resilience: Resilience4j, retry, bulkhead, rate limiting
- [ ] **Task 27** — API Design: REST best practices, versioning, pagination, HATEOAS, gRPC introduction
- [ ] **Task 28** — Event-Driven Architecture: saga pattern (choreography vs orchestration), eventual consistency

## Phase 5: Banking-Grade Concerns (Tuần 16–19)

- [ ] **Task 29** — Idempotency: idempotency keys, deduplication strategies for payment systems
- [ ] **Task 30** — Double-Entry Accounting System: ledger design, transaction balancing, immutable records
- [ ] **Task 31** — Security Hardening: OWASP Top 10 for Java, input validation, SQL injection, XSS prevention
- [ ] **Task 32** — Audit Logging: immutable audit trails, compliance requirements, structured logging
- [ ] **Task 33** — High Availability: active-active, active-passive, failover strategies, health checks
- [ ] **Task 34** — Performance Engineering: JVM tuning, profiling (JFR, async-profiler), load testing (JMeter/Gatling)
- [ ] **Task 35** — Observability: distributed tracing (OpenTelemetry), metrics (Micrometer/Prometheus), structured logging

## Phase 6: DevOps & Production (Tuần 20–22)

- [ ] **Task 36** — Containerization: Docker multi-stage builds, JVM in containers, resource limits
- [ ] **Task 37** — Kubernetes for Java Apps: deployments, services, configmaps, liveness/readiness probes
- [ ] **Task 38** — CI/CD Pipeline: GitHub Actions for Java, automated testing, deployment strategies (blue-green, canary)
- [ ] **Task 39** — Infrastructure as Code: Terraform basics, environment management, secrets handling
- [ ] **Task 40** — Production Readiness Checklist: the complete guide to shipping Java apps to production

---

## 📁 Research Files

Completed research files are stored in `/research/` directory:

| # | Topic | File | Status |
|---|-------|------|--------|
| 01 | JVM Architecture: Class Loader, Runtime Data Areas, Execution Engine | research/01-jvm-architecture.md | ✅ |

---

## 📊 Progress

- **Total Tasks:** 40
- **Completed:** 1
- **Remaining:** 39
- **Current Phase:** Phase 1 — Java Core Mastery

---

> _Last updated: 2026-04-05_
