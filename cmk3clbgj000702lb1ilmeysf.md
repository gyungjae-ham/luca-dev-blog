---
title: "What Happens to the Thread Count When Running a JAR on Linux?"
seoTitle: "JAR"
seoDescription: "JAR"
datePublished: Wed Jan 07 2026 01:36:45 GMT+0000 (Coordinated Universal Time)
cuid: cmk3clbgj000702lb1ilmeysf
slug: what-happens-to-the-thread-count-when-running-a-jar-on-linux
tags: threads, jar

---

* JVM threads are mapped directly to OS threads.
    

* This is known as the **Native Thread Implementation** or the **1:1 Threading Model**.
    

### Thread Pool in Spring Boot

* By default, Spring Boot creates a thread pool when using an embedded Tomcat server.
    
* The default configuration allows for a maximum of **200 threads**.
    
* Each incoming request is handled by a worker thread from this pool.
    

---

## Q1. If I spin up a container using a Docker image, does that container get assigned processes and threads from the host OS?

### 1\. Relationship Between Containers and Processes

* A Docker container runs as an isolated group of processes on the host OS.
    
* It leverages Linux **namespaces** and **cgroups** to isolate processes and limit resource usage.
    
* In reality, it shares the host OS kernel.
    

### 2\. In the Case of Spring Boot Applications

* The JVM running inside the container is also a process on the host OS.
    
* JVM threads are still mapped to the host OS's native threads.
    
* However, they are subject to the container's resource constraints (CPU, Memory, etc.).
    

### 3\. Verification Commands

Bash

```dockerfile
# Check container processes from the host OS
docker top [container-id]

# Check processes inside the container
docker exec [container-id] ps -ef

# Check specific threads
docker exec [container-id] ps -eLf
```

### 1\. Container Structure

* Each container includes its own application, libraries, and runtime.
    
* Containers remain isolated from one another.
    

### 2\. Namespace Isolation

* **PID Namespace**: Isolates Process IDs.
    
    * A process may have PID 1 inside the container but appear as PID 1234 on the host system.
        
* **Network Namespace**: Isolates the network stack.
    
    * Provides each container with an independent network stack (interfaces, IP addresses, routing tables, port numbers, iptables rules).
        
    * When a container is created, Docker generates a **veth (virtual ethernet)** pair.
        
    * One end resides in the host's network namespace, while the other (eth0) resides in the container's.
        
    * These interfaces act like a pipe to forward traffic.
        
* **Port Mapping**
    
    * `-p 8080:80`
        
* **Mount Namespace**: Isolates filesystem mount points.
    
* **User Namespace**: Isolates User and Group IDs.
    

### 3\. Resource Control (cgroups)

* Manages CPU, Memory, and I/O usage.
    

### 4\. Kernel Sharing

* All containers share the same host OS kernel and access system resources through it.
    

---

## Q2. In a Kotlin Spring Boot environment, are the threads in the Dispatchers pool used by Coroutines also OS threads?

### 1\. Dispatcher Thread Pools

* **Dispatchers.Default**: Uses a thread pool proportional to the number of CPU cores (Minimum 2, Maximum: CPU cores + 1).
    
* [**Dispatchers.IO**](http://Dispatchers.IO): Uses a shared thread pool (defaults to up to 64 threads).
    
* All of these are indeed **native OS threads**.
    

### 2\. Coroutines vs. Threads

* While Coroutines are called "lightweight threads," they are actually units of work executed on top of threads.
    
* Multiple coroutines can run on a single thread (**M:N mapping**).
    
* Switching between coroutines is significantly "lighter" than switching between threads.
    

---

## Q3. When a Coroutine Scope is executed, is a specific thread type assigned? And is switching between coroutines really cheaper than thread context switching?

### 1\. Coroutine Scope and Dispatchers

Kotlin

```kotlin
// Determine which thread pool to use via the Dispatcher
CoroutineScope(Dispatchers.Default).launch {
    // All coroutines in this scope use the Default thread pool
    
    // Switching between coroutines (e.g., during suspension) is extremely lightweight
    val result1 = async { heavyComputation() }
    val result2 = async { anotherComputation() }
}
```

### 2\. Context Switching Cost Comparison

* **Thread Context Switch**: Occurs at the OS level (Expensive).
    
    * Requires saving/loading CPU register states: **Program Counter (PC)**, **Stack Pointer (SP)**, General-purpose registers, and Status registers.
        

> **Thread Switching Process:**
> 
> 1. Thread A is running -&gt; Save Thread A's register values to memory (in the **PCB**).
>     
> 2. Load Thread B's register values from memory.
>     
> 3. Start Thread B execution.
>     

> What is a PCB (Process Control Block)?
> 
> A metadata structure maintained by the OS to manage each process.
> 
> * **Management Info**: PID, Status, Priority, PC, CPU Registers, Scheduling info.
>     
> * **Memory Info**: Allocation, Page/Segment tables.
>     
> * **File/IO Info**: Open files, File descriptors, I/O devices.
>     

* **Memory Map Switching**: Switching the virtual memory address space.
    
    * **TLB Flushing**: Increases page table walks as the translation cache becomes invalid.
        
    * **Increased Cache Misses/Page Faults**: Since the new process needs different data, memory access latency increases.
        
* **CPU Cache Invalidation**: Data in the L1/L2/L3 caches may no longer be valid for the new thread, leading to cache misses.
    
* **Coroutine Switching**:
    
    * Occurs within the same thread.
        
    * Only saves execution state and stack information.
        
    * Extremely lightweight with minimal memory access.
        

Kotlin

```kotlin
// Conceptual structure of Coroutine Suspension (Continuation)
fun example(continuation: ExampleContinuation) {
    when(continuation.label) {
        0 -> {
            continuation.x = 10
            continuation.label = 1 
            delay(1000, continuation)
        }
        1 -> {
            val x = continuation.x // Restore state
            println(x)
            continuation.label = 2
            delay(2000, continuation)
        }
        2 -> {
            val x = continuation.x
            println(x + 1)
        }
    }
}
```

---

## Q4. If I set the Spring Boot Max Thread Pool to 200, are Async and Coroutine threads created separately?

Yes, they use **independent thread pools**:

1. **Tomcat Thread Pool**: `server.tomcat.threads.max=200` (For HTTP requests).
    
2. **Spring @Async Pool**: Configured separately via `ThreadPoolTaskExecutor`.
    
3. **Coroutine Dispatcher Pool**: `Default` and `IO` dispatchers maintain their own pools.
    

Total Thread Count = Tomcat Threads + @Async Threads + Coroutine Threads.

In a Docker environment, you must be cautious as the sum of these pools can lead to heavy resource contention.

---

## Q5. If I run a JAR in Docker with Max Threads at 200, 2 Async threads, and default Coroutine settings, how are Coroutine threads calculated?

* **Tomcat**: Max 200.
    
* **Async**: 2.
    
* **Dispatchers.Default**: `min(Core Count * 2, 128)`.
    
* [**Dispatchers.IO**](http://Dispatchers.IO): Max 64.
    

For an **8-core system**:

* $200 (Tomcat) + 2 (Async) + 16 (Default) + 64 (IO) = 282$ potential threads.
    
    Note: Threads are created/destroyed dynamically based on demand; they aren't all allocated at startup.
    

---

## Q6. What happens to resources if I spin up a second identical container on the same host?

* **Container 1**: Up to 282 threads.
    
* **Container 2**: Up to 282 threads.
    
* **Total**: Up to 564 threads.
    

**Critical Point**: Both containers share the same host CPU and memory. Without limits, they will compete for resources, leading to **resource contention** and performance degradation. It is best practice to define limits:

YAML

```yaml
# docker-compose.yml example
services:
  app1:
    cpus: '4'
    pids_limit: 150
  app2:
    cpus: '4'
    pids_limit: 150
```

---

## Q7. Maximize Performance: Large Server with Many Containers vs. Multiple Small Servers?

<table><tbody><tr><td colspan="1" rowspan="1"><p><strong>Strategy</strong></p></td><td colspan="1" rowspan="1"><p><strong>Pros</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>Many Containers on One Server</strong></p></td><td colspan="1" rowspan="1"><p>Efficient resource sharing, simpler management, lower cost, fast inter-container communication.</p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>Distributed over Small Servers</strong></p></td><td colspan="1" rowspan="1"><p>Better fault isolation, easier individual scaling, less resource contention, hardware redundancy.</p></td></tr></tbody></table>

**Verdict**: It depends. Complementary services should be grouped, while high-load/mission-critical services should be isolated.

---

## Q8. In K8s (1 Master, 3 Nodes), does scaling out mean adding more containers to the same server?

Scaling out in Kubernetes follows two paths:

1. **Sufficient Node Resources**: The K8s Scheduler places new Pods (containers) on existing nodes. Multiple containers will run on one server.
    
2. **Insufficient Node Resources**: Pods enter a **Pending** state. You must add physical/virtual nodes. In cloud environments, **Cluster Autoscaler** can automate this.
    

**Pro-tip**: Always define `resources.requests` and `limits` to make scaling predictable.