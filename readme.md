  # Lead .NET Software Engineer — Interview Prep

Complete study guide covering all rounds of a Lead .NET Software Engineer interview.
Open `index.html` in your browser for the interactive version with navigation.

---

## Study Modules

| # | Topic | Files | Key Areas |
|---|-------|-------|-----------|
| 01 | [C# Advanced](./01-csharp-advanced/csharp-advanced.md) | [md](./01-csharp-advanced/csharp-advanced.md) · [html](./01-csharp-advanced/csharp-advanced.html) | Delegates, LINQ, Generics, Reflection, Expression Trees |
| 02 | [async/await & Threading](./02-async-threading/async-threading.md) | [md](./02-async-threading/async-threading.md) · [html](./02-async-threading/async-threading.html) | Task vs Thread, ConfigureAwait, Deadlocks, Span\<T\>, CancellationToken |
| 03 | [ASP.NET Core](./03-aspnet-core/aspnet-core.md) | [md](./03-aspnet-core/aspnet-core.md) · [html](./03-aspnet-core/aspnet-core.html) | Middleware, DI Lifetimes, JWT, Filters, SignalR, Rate Limiting |
| 04 | [Entity Framework Core](./04-ef-core/ef-core.md) | [md](./04-ef-core/ef-core.md) · [html](./04-ef-core/ef-core.html) | Change Tracker, N+1, AsNoTracking, Bulk Ops, Concurrency |
| 05 | [SQL](./05-sql/sql.md) | [md](./05-sql/sql.md) · [html](./05-sql/sql.html) | Indexes, Window Functions, Isolation Levels, 6 Coding Problems |
| 06 | [System Design](./06-system-design/system-design.md) | [md](./06-system-design/system-design.md) · [html](./06-system-design/system-design.html) | SOLID, CQRS, Microservices, Order/Chat/URL Shortener Design |
| 07 | [Cloud — AWS](./07-cloud-aws/cloud-aws.md) | [md](./07-cloud-aws/cloud-aws.md) · [html](./07-cloud-aws/cloud-aws.html) | ECS/EKS, SQS/SNS, IAM, VPC, Zero-Downtime Deploy |
| 08 | [Docker & Kubernetes](./08-docker-kubernetes/docker-kubernetes.md) | [md](./08-docker-kubernetes/docker-kubernetes.md) · [html](./08-docker-kubernetes/docker-kubernetes.html) | Multi-stage Builds, HPA, Probes, Rolling Updates |
| 09 | [DevOps](./09-devops/devops.md) | [md](./09-devops/devops.md) · [html](./09-devops/devops.html) | GitHub Actions CI/CD, Blue/Green, Canary, Feature Flags |
| 10 | [Leadership & Behavioral](./10-leadership/leadership.md) | [md](./10-leadership/leadership.md) · [html](./10-leadership/leadership.html) | 12 STAR Answers, Conflict, Outage, Code Review, Estimation |
| 11 | [Coding Algorithms](./11-coding-algorithms/coding-algorithms.md) | [md](./11-coding-algorithms/coding-algorithms.md) · [html](./11-coding-algorithms/coding-algorithms.html) | 15 LeetCode Problems, Full C# Solutions, Pattern Guide |
| 12 | [Performance & Optimization](./12-performance/performance.md) | [md](./12-performance/performance.md) · [html](./12-performance/performance.html) | GC, Memory Leaks, Redis, ObjectPool, BenchmarkDotNet |
| 13 | [Security](./13-security/security.md) | [md](./13-security/security.md) · [html](./13-security/security.html) | JWT, OAuth, OWASP Top 10, XSS, CSRF, Secrets Management |

---

## Recommended Study Order

### Week 1 — Core .NET
1. **01** C# Advanced — delegates, LINQ, generics
2. **02** async/await & Threading — most common interview topic
3. **03** ASP.NET Core — middleware, DI, JWT
4. **04** Entity Framework Core — N+1, change tracker

### Week 2 — Design & Data
5. **06** System Design — SOLID, CQRS, architecture (most important round)
6. **05** SQL — indexes, window functions, coding problems
7. **11** Coding Algorithms — practice 2-3 problems daily

### Week 3 — Cloud & DevOps
8. **07** Cloud AWS — ECS, SQS/SNS, VPC
9. **08** Docker & Kubernetes — Dockerfile, probes, HPA
10. **09** DevOps — CI/CD pipelines, deployment strategies

### Week 4 — Polish
11. **12** Performance & Optimization
12. **13** Security — OWASP, JWT internals
13. **10** Leadership & Behavioral — rehearse STAR answers out loud

---

## Quick Interview Checklist

### Round 1 — C# & .NET
- [ ] Explain `async`/`await` state machine
- [ ] Difference `Task` vs `Thread`
- [ ] What is `ConfigureAwait(false)` and when to use it
- [ ] Deferred vs immediate LINQ execution
- [ ] `IDisposable` pattern + `using`

### Round 2 — ASP.NET Core
- [ ] Middleware vs Filter differences
- [ ] DI service lifetimes (Singleton/Scoped/Transient)
- [ ] JWT validation steps
- [ ] Captive dependency problem

### Round 3 — Entity Framework & SQL
- [ ] N+1 problem and fix
- [ ] `AsNoTracking()` use case
- [ ] `First()` vs `Single()` vs `Find()`
- [ ] Clustered vs non-clustered index
- [ ] Delete duplicate rows (SQL)

### Round 4 — System Design
- [ ] SOLID with real violations
- [ ] CQRS pattern
- [ ] Design a notification system
- [ ] Design an order management system

### Round 5 — Leadership
- [ ] Production outage story (STAR)
- [ ] Biggest technical decision
- [ ] How you mentor junior devs
- [ ] How you handle missed deadlines