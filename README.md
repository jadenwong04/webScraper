# Building Scalable and Fault-Tolerant Web Scrapers

---

## Introduction
Data is an important foundation for any type of project — whether it's powering a machine learning model, driving business intelligence dashboards, or monitoring real-world trends.

Yet, in most projects, acquiring the right data is often the most challenging step. The difficulties stem from two key factors:
- **Volume** — Do we have sufficient data to produce statistically meaningful insights?
- **Coverage** — Is the data representative enough to avoid bias and capture the full scope of the problem?

> This is where web scraping becomes essential — it allows us to control what data we collect and how we collect it, all at scale.

---

## Problem
Web scraping may sound trivial at first — "Just extract content from an HTML page". However, even a seemingly simple architecture can quickly render your scraper unusable when faced with real-world conditions.

### 1. Throughput
In most cases, websites do not publicly present all of their data on a single endpoint — as a series of interactions are required to expose a partial / specific-category of the complete data.

#### Scenario: Analyzing bus ticket pricing in Malaysia
![Landing Page Screenshot](images/entry.png)
Ticket data are commonly categorized based on the origin, destination and departmentDate.

Let's say we want to scrape for all available bus tickets on this specific day.

**Workload Analysis:**

![Estimated Workload Function](images/workload.png)
- Number of supported Points-of-Interests (N): 502

- Total workload (worst-case): $N * (N-1) = 502 * 501 = 251,502$ requests!

**Trivial Workflow:**
```python
import requests

session = requests.Session()

# Assuming that each category (origin -> destination) is already acquired.
for origin, destination in category_data:
    payload = {"origin": origin, "destination": destination}
    try:
        response = session.post(ENDPOINT, data=payload, timeout=10)
        response.raise_for_status()
        append_to_log(response.json())
    except requests.RequestException as e:
        print(f"Error fetching {origin} → {destination}: {e}")

session.close()
```

**Analysis:**

![Result Success/Failure Distribution](images/analysis.png)

After 8 hours of continuous execution:
- ~22,000 succeeded
- ~3,000 timed out
- ~226,000 never executed

> The scraper spent an entire workday processing less than 10% of its total workload.

### 2. Availability
As execution progressed, we can observe from the analysis that a large distribution of failed requests occurred during end of the execution.

> This relationship aligns with a technique called **rate-limiting** which are used to throttle excessive traffic coming from the same origin.

---

## Solution #1: Concurrency
At surface, the workload appears to be inherently sequential. Each category of data is obtained through independent HTTP calls. Therefore, algorithmic optimizations are limited here.

Let's consider optimizations on the system-level.

**Question**:
Are there dependencies between each request?

**Answer**:
In our scenario, each ticket are independent of one another and can be executed separately.

With that in mind, let's introduce a preprocessing step.

### Preprocessing
The goal of this step is to bundle all time-consuming modular operations so we that we can parallelize them later on. In our case, we will construct all combinations of ticketing requests in advance.

**Concerns**
- **Would this be applicable when dependencies exists?**
> Depending on the extent/level of dependencies. If endpoints are deeply rooted across many different branches (with respect to N), the preprocessing step may have to be decomposed into separate steps/layers. In essence, it is important to analyze the tradeoff between preprocessing overhead and theoretical speed-up.

### Choosing a Model
Network requests are I/O bound tasks, where the primary bottleneck is I/O Latency, not CPU computation.

![Request Execution](images/requestExecution.png)

In Python, we have a few concurrency model to choose from:
- Multi-processing
- Multi-threading
- Asynchronous I/O

#### 1. Multi-processing

**Verdict:** Not ideal.

**Resource Overhead**

Each process requires its own memory space and interpreter instance. In a workload requiring high concurrency, this can lead to excessive resource usage which are undesirable when we are working with limited resources.

**Data Ingestion Challenges**

Process have their own memory space. Ingesting scraped data into a data-source requires careful consideration.

- **Individual Ingestion:** Every process handles its own data persistence. While frequent writes are necessary to ensure data is backed up during long runs, this often leads to race conditions. Without locking mechanisms, multiple processes attempting to write to the same data source can cause data corruption.
- **Bundled Ingestion:** Requires Inter-Process Communication to aggregate data which contributes to overhead issue.

#### 2. Multi-threading

**Verdict:** Viable.

**I/O Concurrency**

The Global Interpreter Lock (GIL) is frequently misunderstood as a total blocker for concurrency. In reality, GIL is released during I/O-bound tasks, allowing multiple threads to wait for network responses in parallel. This enables threading to achieve high throughput for web scraping without the overhead drawback of multiprocessing.

**Context Switching and Synchronization**

Despite being lighter than processes, threads are managed by the OS. Scaling to thousands of concurrent requests results in high context-switching overhead where CPU time is wasted on thread management.

#### 3. Asynchronous I/O (asyncio)

**Verdict**: Optimal.

> **Key Concepts**
> 1. Event Loop: Manages scheduled coroutines and their execution.
> 2. Coroutines: Function that can pause and resume their executions.

In essence, this model achieves concurrency via an event-driven architecture where coroutines can yield control back to the event loop during I/O operations.

**Resource Overhead**

An asyncio event loop operates on a single thread. Coroutines share memory space, enabling thousands of concurrent connections without thread/process overhead.

**Context Switching**

Implicit context switching — concurrency behavior can be manually controlled using await `await` statements as opposed to OS controlled with processes and threads.

In addition, resolving race conditions will be easier because behavior of context switching is traceable.

---

## Solution #2: Distribution

![Too Many Requests Error](images/error429.png)

Concurrency amplifies rate-limiting issues (HTTP 429: Too Many Requests). To address this, we need a way of distributing requests across multiple sources.

In this article, we will document the use of **Proxy Servers**.

> A proxy server is an intermediary that forwards requests
between clients and servers, acting on behalf of the
client to access resources.

Going along with the theme of limited resources, we will be using public proxy servers.
- **Advantage:** Free and abundant, enabling our scrapers to scale easily.
- **Disadvantage:** Performance and reliability degrade over time due to overuse and latency.

### Challenge Analysis
1. **Problem:** A large distribution of proxies are unusable causing failed requests to fail which bottlenecks the throughput.
   - **Solution:** Preprocessing Proxy Servers Asynchronously.
2. **Problem:** Proxies servers are not long-lasting, we need a way of tracking usable ones and revoking expired ones.
   - **Solution:** Managing Proxy Servers.

### Preprocessing Proxy Servers Asynchronously

```mermaid
flowchart LR
    A[Acquired Public Proxies] --> Funnel
    B[Root URL] --> Funnel

    Funnel{ }

    subgraph Validation[Validation Pipeline]
        direction LR
        V1[Not in blacklist] --> V2[Request Execution<br/><i>timeout: x s</i>] --> V3[Success within y retries]
    end

    D[(Push to Proxy Pool)]

    Funnel --> Validation
    V3 --> D

    note1[<b>Note:</b> Skip known bad proxies<br/>from blacklist file]
    note2[<b>Note:</b> Verify proxy can reach<br/>target domain]
    note3[<b>Note:</b> Retry on transient<br/>failures before discarding]

    V1 -.- note1
    V2 -.- note2
    V3 -.- note3

    style Funnel fill:#fff,stroke:#333,stroke-width:2px
    style Validation fill:#f9f9f9,stroke:#333,stroke-width:1px
    style note1 fill:#fff9c4,stroke:#ddd,stroke-width:1px
    style note2 fill:#fff9c4,stroke:#ddd,stroke-width:1px
    style note3 fill:#fff9c4,stroke:#ddd,stroke-width:1px
```

The validation step is straightforward, we filter out proxy servers that cannot access the root endpoint of the website.

However, this additional step can be very time-consuming and hog the main scraper's execution.

Therefore, we need an efficient way of integrating this.

#### Integration

```mermaid
sequenceDiagram
autonumber
participant P as Preprocessing<br/>Process
participant D as Database
participant Q as Validated<br/>Proxy Queue
participant S as ScraperScript
participant E as EventLoop
participant PP as ProxyPool
participant AP as Async Proxy<br/>Preprocessor
participant W as Website

    rect rgb(200, 220, 255)
        note over P,D: Preprocessing Stage
        P->>W: Crawl website for request metadata
        P->>P: Parse & construct metadata combinations
        P->>D: Store in category_table
    end

    rect rgb(220, 255, 200)
        note over S,PP: Main Scraping Stage
        S->>D: Fetch request metadata
        D-->>S: Return metadata list

        S->>S: Construct request objects
        S->>E: Schedule coroutines

        loop EventLoop processes coroutines
            S->>PP: get_proxy()

            opt Pool size < threshold
                PP->>AP: Trigger refresh
                activate AP
                AP->>W: Scrape & validate proxies
                AP->>Q: Push to validated queue
                AP-->>PP: Refresh complete
                deactivate AP

                PP->>PP: Lock pool
                PP->>Q: Fetch validated proxies
                PP->>PP: Update pool
                PP->>S: Release lock
            end

            PP-->>S: Return proxy

            S->>W: Execute request via proxy
            alt Success
                W-->>S: Response
                S->>S: Process & store data
            else Rate Limited / Timeout
                W-->>S: 429 / Timeout
                S->>PP: remove_proxy()
                S->>E: Retry with new proxy
            end
        end
    end
```

#### Why this solves the time-consuming aspect

The async preprocessor transforms a blocking bottleneck into a non-blocking background task.

| **Aspect** | **Synchronous Approach** | **Async Preprocessor**                                                    |
|------------|-------------------------|---------------------------------------------------------------------------|
| **Timing** | Validate proxies during scraping (blocking) | Validate on-demand when pool is low (avoid unnecessary upfront work)      |
| **Execution** | Sequential validation | Parallel validation via a separate event loop                             |
| **Impact** | Scraper waits for each validation | Scraper continues while validation runs in background (Producer-Consumer) |

> The scraper maintains high throughput because proxy validation no longer competes with request execution. The validated queue acts as a buffer, ensuring a steady supply of working proxies without blocking the main scraping loop.

### Managing Proxy Servers

With validated proxies in hand, the challenge shifts to efficient utilization — maximizing throughput while handling inevitable failures.

#### Design Requirements

**1. Even Load Distribution**

To prevent rate-limiting, proxies must be used evenly — each proxy should handle roughly the same number of requests before any repetition.

```mermaid
flowchart LR
    subgraph State1[Request 1]
        P1[Proxy 1] --> P2[Proxy 2]
        P2 --> P3[Proxy 3]
        P3 --> P4[Proxy 4]
        P4 --> P1
        style P1 fill:#bbf,stroke:#333,stroke-width:3px
        style P2 fill:#fff,stroke:#333
        style P3 fill:#fff,stroke:#333
        style P4 fill:#fff,stroke:#333
    end

    subgraph State2[Request 2]
        Q1[Proxy 1] --> Q2[Proxy 2]
        Q2 --> Q3[Proxy 3]
        Q3 --> Q4[Proxy 4]
        Q4 --> Q1
        style Q1 fill:#fff,stroke:#333
        style Q2 fill:#bbf,stroke:#333,stroke-width:3px
        style Q3 fill:#fff,stroke:#333
        style Q4 fill:#fff,stroke:#333
    end

    State1 -.->|Advance pointer| State2
```

---

#### Choice of Data Structure

The **Circular Rotation** behavior is not the sole factor that determines the data structure that we use. 

Let's take a look at the design comparisons.

**1. Array-based Proxy Pool**

```mermaid
flowchart LR
    subgraph Before[Before]
        direction LR
        B1["Proxy 1"] --- B2["Proxy 2 ❌"] --- B3["Proxy 3"] --- B4["Proxy 4"]
        style B2 fill:#f99,stroke:#333
    end

    subgraph After[After]
        direction LR
        C1["Proxy 1"] --- C2["Proxy 3"] --- C3["Proxy 4"] --- C4[" "]
        style C4 fill:#eee,stroke:#333
    end

    Before -->|Shift elements left| After
```
This design supports both the load balancing and ingestion behavior.

However, if we referred back to the drawbacks of using public proxy servers, we know that they are likely to fail overtime. Therefore, the deletion of faulty proxies must also be considered in our design.

> With the Array implementation, removing a failed proxy requires shifting all subsequent elements. Consequently, this will decrease concurrency due to increased lock contention.

**2. Linked-List-based Proxy Pool**

```mermaid
flowchart LR
    subgraph Before[Before]
        L1[Proxy 1] <--> L2[Proxy 2 ❌] <--> L3[Proxy 3] <--> L4[Proxy 4]
        L4 <--> L1
        style L2 fill:#f99,stroke:#333,stroke-dasharray: 5 5
    end

    subgraph After[After]
        L5[Proxy 1] <--> L6[Proxy 3] <--> L7[Proxy 4]
        L7 <--> L5
        style L5 fill:#e8f5e9,stroke:#2e7d32
        style L6 fill:#e8f5e9,stroke:#2e7d32
        style L7 fill:#e8f5e9,stroke:#2e7d32
    end

    Before -.->|Unlink| After
```

By incorporating a `previous` and `next` pointer for each node, we can enable constant time proxy removals. Thereby, reducing lock contention and increasing concurrency. 

In addition to that, the circular behavior can be replicated more easily as with pointer transitions instead of indexing updates.

The trade-off for the increased concurrency is additional pointer references for each node.

---

#### Adaptive Concurrency

With a circular proxy pool that can indefinitely distribute even workload with available proxies. **How can we control the number of executing requests aka Concurrency?**

| Concurrency Limit | Result                                                        |
|-------------------|---------------------------------------------------------------|
| Too Small         | Proxies sit idle and we don't reap the benefits of concurrency |
| Too Large         | Risk of proxies rate limiting early.                          |

**Solution: Dynamic Semaphore**

A standard semaphore has a fixed limit set at initialization. But in our proxy pool, the size is constantly changing — new proxies or revoked proxies.

```mermaid
flowchart TB
    A[Pool grows] --> B[Semaphore expands]
    B --> C[More concurrency]
    
    D[Pool shrinks] --> E[Semaphore contracts]
    E --> F[Prevent overload]
    
    style A fill:#e3f2fd,stroke:#1565c0
    style D fill:#ffebee,stroke:#c62828
```

Whenever there is a change to the size of the proxy pool, the limit recalculates as `pool_size × speedup_factor`.

> This creates a self-regulating system: concurrency expands to seize opportunity, contracts to prevent overload, and never blocks requests that already hold permits.

---