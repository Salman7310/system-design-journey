# System Design & Scalability: Vertical Scaling, Benchmarks and Trade-offs

> **Lecture:** Episode 2, *Vertical Scaling Explained: NodeJS vs Go vs Fastify vs NestJS, Nginx | More CPU = More Power?*  
> **Instructor:** Ankit Jain  
> **Video:** https://www.youtube.com/watch?v=PMD1fat6QdY&list=PLSvDIozU0pRk&index=4  
> **Scope:** Vertical scaling, Node.js concurrency, framework benchmarks, PM2, Nginx, caching and compression.
>
> **Evidence note:** Benchmark figures below are transcribed from the Gemini Notebook notes supplied for review, **not independently verified against the full video transcript**. Treat them as lecture-reported examples rather than independently verified measurements.

## TL;DR: Five ideas to remember

1. **More cores ≠ proportionally more requests.** Throughput depends on workload, architecture and contention.
2. **One Node.js process has one main JavaScript event-loop thread**, but asynchronous I/O and native worker pools can still use other threads; multiple processes or worker threads enable parallel JavaScript execution.
3. **PM2 clustering helps exploit multicore hardware**, but the best process count must be measured.
4. **Nginx caching can remove backend work altogether**, often producing larger gains than tuning framework overhead.
5. **Benchmark throughput, tail latency, errors and resource use together.** A high RPS figure alone is not proof of a healthy system.

---

## 1. Scaling fundamentals [00:00–06:00]

**Scalability** is a system's ability to handle increased workload while maintaining acceptable performance and reliability.

| Dimension | Vertical scaling (scale up/down) | Horizontal scaling (scale out/in) |
|---|---|---|
| Change | More/fewer resources on one machine | More/fewer machines or instances |
| Example | 2 vCPU → 8 vCPU | 1 app server → 4 app servers |
| Main benefit | Operational simplicity | Capacity and fault-tolerance options |
| Main constraint | Hardware ceiling and machine-level failure risk | Distributed state, networking and coordination |

**Analogy:** A faster vehicle helps one courier, but a fleet can perform deliveries in parallel. Neither solves every bottleneck: traffic jams and a slow warehouse can constrain both.

**Key nuance:** A single large server is a potential **single point of failure (SPOF)** unless the deployment includes redundancy or failover.

## 2. Node.js and multicore execution [06:00–08:28; 27:00–32:20]

```text
One Node.js process
  ├─ Main JS thread + event loop
  ├─ Asynchronous OS/network I/O
  └─ Native/libuv worker pool for certain operations

PM2 cluster on one host
  ├─ Node worker process 1
  ├─ Node worker process 2
  ├─ Node worker process 3
  └─ Node worker process 4
       (workers can share an application listening port)
```

- **I/O-bound work:** Node.js can serve many concurrent requests using nonblocking I/O, even with one main JS thread.
- **CPU-bound JS:** Long-running synchronous computation blocks that process's event loop.
- **Parallelism:** Use `worker_threads` for CPU-heavy JS tasks or multiple Node.js processes for request-serving capacity.
- **PM2:** `pm2 start app.js -i 4` launches four clustered instances; `-i max` selects a count based on available CPUs, which is **not necessarily optimal**.
- **Operational caveat:** Each process has its own memory. In-memory sessions, rate limits and caches need shared-state strategies if requests can land on different workers.

**Do not say:** “A Node.js process can use only one CPU core.” More precisely, its *main JS execution thread* runs on one core at a time; the process may also use other native threads.

## 3. Benchmark setup and measurement [08:28–13:38]

**Reported workload:** A paginated product API backed by MongoDB and Redis, tested with approximately **3,000 concurrent virtual users** using k6. The provided notes mention Prometheus/Grafana for monitoring.

**Metrics to record:**

| Metric | What it answers | Pitfall |
|---|---|---|
| Throughput (RPS) | How many requests finish per second? | Can hide failures or slow responses |
| P50 / P95 / P99 latency | How long do typical and tail requests take? | Average latency hides spikes |
| Error rate | What proportion failed? | High throughput of errors is not success |
| CPU utilization | How busy is the host/process? | High CPU alone does not identify the bottleneck |
| Load average / runnable queue | Is work competing for execution? | **Load average is not a percentage**; interpret relative to CPU count and workload |
| Memory, GC, I/O | Is another resource limiting throughput? | CPU may be idle while DB/network is saturated |

**Fair-test checklist:** Hold hardware, endpoint behavior, dataset, response size, concurrency/ramp-up, test duration and warm-up constant. Separate load generator from server where possible. Repeat runs and compare equivalent conditions.

## 4. Lecture-reported benchmark snapshot [13:38–27:00; 28:18–45:48]

*Approximate figures below come from the user-provided Gemini notes. Some columns mix requests completed over a test window with RPS; compare only runs with equivalent duration and conditions.*

| Configuration | Total requests reported | Approx. RPS reported | Interpretation |
|---|---:|---:|---|
| Bare Node.js HTTP, single process | 574,000 | 5,225 | Minimal framework overhead |
| Express, single process | 561,480 | 5,000 | Convenient framework, some overhead |
| Fastify, single process | 719,992 | 6,000+ | Optimized routing/serialization can help |
| Go server | 1,947,578 | 17,700 | Go runtime can exploit multiple cores; not an apples-to-apples single-core comparison |
| Fastify + PM2, 4 workers | 1,651,200 | 15,000+ | Roughly 2.3× single-process throughput in these reported runs |
| Fastify + PM2, 10 workers | 1,071,639 | Not specified | About **35% fewer completed requests** than 4 workers, assuming comparable test duration |
| Express + PM2, 4 workers | 1,584,215 | 14,000+ | Large gain over single-process Express |
| Fastify behind Nginx micro-cache | Not consistently specified | 56,631–62,963 | Cache-hit traffic avoids much of the app/DB work; **not directly comparable** to uncached API processing |

### Reading the results correctly

- **Fastify vs Express:** Fastify leads in the reported test, but real applications may be dominated by database, network or business-logic costs.
- **Go vs Node:** Go's reported lead includes a different runtime/concurrency model; do not conclude that the language alone caused the entire difference.
- **4 vs 10 PM2 workers:** The reported regression suggests oversubscription, resource contention or another bottleneck. It does **not** prove that four workers are always best.
- **Cached vs uncached:** A cached response is fundamentally less work than a fresh backend response. Benchmark them as different service modes.

**Data-quality flags:** The original notes label “System Load past 600%,” but conventional Linux load average is a number, not a percent. The Nginx row also mixes RPS and total requests. Verify units, test duration and cache conditions before drawing conclusions.

## 5. Request path and infrastructure [32:20–38:05]

```text
               Client / k6
                    |
                    v
              Nginx reverse proxy
           /          |           \
      cache hit   cache miss    TLS / compression
         |            |
         v            v
     Response     PM2 clustered app
                    / | \
               Node Node Node
                    |
               Redis / MongoDB
```

> This is a **conceptual architecture** combining the discussed components, not a claim that every benchmark used this exact topology.

### Nginx reverse proxy

- Routes incoming requests to an upstream app or pool.
- Can terminate TLS, cache eligible responses and apply HTTP compression.
- Does not automatically provide effective bot protection; additional rules/modules/services may be needed.

### Micro-caching

- **Definition:** Store eligible responses for a very short TTL, such as 1–15 seconds.
- **Benefit:** Repeated requests can be served without hitting application code or the database.
- **Trade-off:** Users may receive stale data until expiry.
- **Safety:** Cache keys must account for query parameters and relevant request variation. Never inadvertently share private, authenticated or user-specific responses.
- **Nuance:** Nginx proxy cache commonly stores response bodies on disk with memory used for metadata; it is not necessarily an all-in-memory cache.

### Compression

- **Gzip/Brotli:** Reduce bytes transferred, potentially improving client-perceived download times.
- **Cost:** Compression consumes CPU, especially at high compression levels and RPS.
- **Decision:** Compare payload sizes, network constraints, CPU saturation and end-to-end latency. The “20 GB → 2–3 GB” example in the supplied notes is workload-dependent, not a general guarantee.

## 6. Why adding workers can hurt [38:05–42:25]

Possible causes of the reported ten-worker slowdown:

1. More runnable processes competing for CPU time.
2. More context switches, garbage collectors and memory pressure.
3. Database/Redis connection contention or network saturation.
4. CPU quotas, shared-host load or load-generator limits.
5. A fixed downstream bottleneck that additional app workers cannot remove.

**Practical rule:** Sweep worker counts (1, 2, 4, 6, 8, …) and select the smallest count that meets throughput, P95/P99 latency and error-rate targets with headroom. Do not infer a universal worker count from one benchmark.

## 7. Engineering decision guide

| If you observe… | Investigate first… | Candidate action |
|---|---|---|
| One Node JS thread pegged, other cores idle | CPU-heavy synchronous code | Worker threads, clustering, offload computation |
| High RPS but poor P99 | Queues, GC, DB tail latency | Profile bottleneck; cap concurrency |
| Repeated identical public reads | Cache hit ratio, freshness needs | Nginx micro-cache / application cache |
| Network transfer dominates | Response size and link speed | Compression, pagination, smaller payloads |
| More PM2 workers reduce RPS | CPU contention, memory, downstream saturation | Reduce worker count; retest |
| One host is a reliability risk | Failure modes and recovery time | Horizontal redundancy and load balancing |

## 8. Interview-ready Q&A

**Q1. Why doesn't one Node.js server use all cores for JavaScript?**  
A Node.js process has one main JavaScript event-loop thread. Use multiple processes or worker threads to run JavaScript in parallel; asynchronous I/O is a separate form of concurrency.

**Q2. Why can `pm2 -i max` perform worse than fewer workers?**  
Worker overhead and shared resource contention can exceed the benefit of additional parallelism. Benchmark the worker count rather than maximizing it blindly.

**Q3. Why might Fastify outperform Express?**  
Its low-overhead routing and schema-aware serialization can reduce request-processing costs. The gain depends on endpoint behavior and environment.

**Q4. Why is Nginx caching sometimes dramatically faster?**  
A cache hit can bypass application logic and database queries. It is an architectural reduction in work, not merely a faster framework.

**Q5. What is the compression trade-off?**  
Fewer bytes over the network versus CPU time to compress them. Measure both throughput and user-visible latency.

**Q6. Is Go always faster than Node.js?**  
No. The reported Go test performed better under its particular workload and concurrency setup; choose based on workload, development needs and measured results.

**Q7. Does high Linux load average mean CPU utilization is 600%?**  
No. Load average is an average count of runnable or uninterruptible tasks over time. Compare it with available logical CPUs and inspect CPU/I/O metrics.

## 9. Active recall and practice

**Explain without notes:**
1. What changes when you add CPU cores to a host but keep one Node.js process?
2. Why is 60k cached RPS not directly comparable to 15k uncached RPS?
3. What would you measure before changing PM2 from 4 to 10 workers?
4. When would a 15-second TTL be unsafe?
5. How would you distinguish a CPU bottleneck from a database bottleneck?

**Mini-lab:**
1. Implement the same simple API in Express and Fastify.
2. Run k6 at several concurrency levels and record RPS, P95, errors, CPU and memory.
3. Repeat with PM2 worker counts of 1, 2, 4 and 8.
4. Add Nginx and test uncached requests separately from cache hits.
5. Enable gzip, repeat tests, and document the latency/CPU/bandwidth trade-off.

## 10. One-minute revision card

```text
Scale up       = bigger machine
Scale out      = more machines
Node event loop= one main JS thread per process
PM2 cluster    = multiple Node processes
k6             = load generation
RPS            = throughput
P95 / P99      = tail latency
Nginx          = reverse proxy / caching / compression
Micro-cache    = faster repeat reads, with staleness risk
More workers   = useful until contention/bottlenecks dominate
```

**Bottom line:** *Optimize the bottleneck, not the hardware specification.* Establish a baseline, make one change at a time, and validate throughput, tail latency, errors and infrastructure cost.

---

### Source and verification notes

- [Lecture video](https://www.youtube.com/watch?v=PMD1fat6QdY&list=PLSvDIozU0pRk&index=4)
- [Series repository](https://github.com/xDAnkit/system-design-journey)
- Exact benchmark values are taken from the **user-provided Gemini Notebook notes** and should be checked against the original lecture or benchmark logs before citation.
