# Round-Robin Job Scheduler

A fair scheduling solution for distributed job processing that prevents large jobs from blocking smaller ones.

---

## Problem Statement

In traditional job processing systems, jobs are processed in order of arrival (FIFO). This creates a **head-of-line blocking** problem:

```
Without Round-Robin:
┌─────────────────────────────────────────────────────────┐
│ Large Job (50 tasks) ███████████████████████████.....   │
│ Small Job (5 tasks)  ···························█████   │
│                                                         │
│ Small job waits until large job completes entirely      │
└─────────────────────────────────────────────────────────┘
```

**Impact:**
- Small jobs experience disproportionately long wait times
- System appears unresponsive to users with small requests
- Resource utilization is technically high, but user satisfaction is low

---

## Solution Overview

**Round-Robin scheduling** distributes worker time fairly across all active jobs, ensuring every job makes progress regardless of size.

```
With Round-Robin:
┌─────────────────────────────────────────────────────────┐
│ Large Job (50 tasks) █·█·█·█·█·█·█·█·█·█·█·█·█·█·█·█·█  │
│ Small Job (5 tasks)  ·█·█·█·█·█                         │
│                                                         │
│ Small job completes while large job is still running    │
└─────────────────────────────────────────────────────────┘
```

---

## Key Components

### 1. Job Queue (FIFO with Rotation)

A circular queue that rotates jobs after each task completion:

```
Initial:  [Job-A] → [Job-B] → [Job-C]
                ↓
After A processes one task:
          [Job-B] → [Job-C] → [Job-A]
                ↓
After B processes one task:
          [Job-C] → [Job-A] → [Job-B]
```

**Implementation:**
- `LPOP` (remove from front) to get next job
- `RPUSH` (add to back) after processing one task
- Jobs are removed entirely when all tasks complete

### 2. Worker Pool (Limited Concurrent Workers)

A fixed pool of workers that process tasks:

```
┌─────────────────────────────────────────┐
│  Worker Pool (12 max)                   │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐     │
│  │ W1 │ │ W2 │ │ W3 │ │ W4 │ │ W5 │     │
│  │busy│ │busy│ │idle│ │busy│ │idle│     │
│  └────┘ └────┘ └────┘ └────┘ └────┘     │
│  Active: 3/12                           │
└─────────────────────────────────────────┘
```

**Worker Lifecycle:**
1. Claim a slot (atomic increment)
2. Pick task from front of queue
3. Process task
4. Release slot (atomic decrement)
5. Trigger next worker if queue not empty

### 3. Event-Driven Orchestration

The system is driven by two events:

| Event | Trigger | Action |
|-------|---------|--------|
| **JobCreated** | New job submitted | Add to queue, spawn workers up to limit |
| **WorkerRelease** | Task completes | Free slot, spawn next worker if available |

```
   JobCreated Event                    WorkerRelease Event
         │                                    │
         ▼                                    ▼
   ┌───────────┐                        ┌───────────┐
   │ Add job   │                        │ Free slot │
   │ to queue  │                        │           │
   └─────┬─────┘                        └─────┬─────┘
         │                                    │
         ▼                                    ▼
   ┌───────────────────────────────────────────────┐
   │         Spawn workers if:                     │
   │         - Slots available (< max)             │
   │         - Jobs in queue                       │
   └───────────────────────────────────────────────┘
```

---

## How It Works

### Step-by-Step Flow

```
1. USER SUBMITS JOB
   ─────────────────────────────────────────────────────
   User submits "Job-A" with 50 tasks

2. JOB ENTERS QUEUE
   ─────────────────────────────────────────────────────
   Queue: [Job-A]

3. WORKERS SPAWN
   ─────────────────────────────────────────────────────
   Workers claim slots and start processing Job-A tasks
   Active Workers: 10/12

4. NEW JOB ARRIVES
   ─────────────────────────────────────────────────────
   User submits "Job-B" with 5 tasks
   Queue: [Job-A] → [Job-B]

5. ROUND-ROBIN KICKS IN
   ─────────────────────────────────────────────────────
   When a worker finishes a Job-A task:
   - Job-A rotates to back of queue
   - Next worker picks Job-B (now at front)

   Queue: [Job-B] → [Job-A]

6. FAIR DISTRIBUTION
   ─────────────────────────────────────────────────────
   Tasks alternate: A, B, A, B, A, B...
   Job-B completes after ~10 task cycles
   Job-A continues until all 50 tasks done
```

### State Diagram

```
                    ┌──────────────┐
                    │   PENDING    │
                    │  (in queue)  │
                    └──────┬───────┘
                           │
                    Worker claims task
                           │
                           ▼
                    ┌──────────────┐
                    │  PREPARING   │
                    │ (claimed)    │
                    └──────┬───────┘
                           │
                    Worker starts
                           │
                           ▼
                    ┌──────────────┐
                    │   RUNNING    │
                    │ (processing) │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
        Success                      Failure
              │                         │
              ▼                         ▼
       ┌──────────────┐          ┌──────────────┐
       │   SUCCESS    │          │   FAILED     │
       │  (complete)  │          │  (error)     │
       └──────────────┘          └──────────────┘
```

---

## Benefits

### 1. Fairness
Every job gets equal opportunity to make progress, regardless of when it was submitted or how large it is.

### 2. Responsiveness
Small jobs complete quickly even when large jobs are in the system. Users see results faster.

### 3. Predictable Progress
All jobs show steady progress rather than some completing instantly while others wait indefinitely.

### 4. Scalability
The worker pool can be adjusted based on system capacity. More workers = faster overall throughput.

---

## Trade-offs

### 1. Large Job Completion Time
Large jobs take slightly longer to complete because they share worker time with other jobs.

```
Without Round-Robin: Large job = 50 task-cycles
With Round-Robin:    Large job = 50 task-cycles + overhead from rotation
```

**Mitigation:** The overhead is minimal (typically <5%) and the fairness benefit outweighs the cost.

### 2. Queue Management Overhead
Maintaining the round-robin queue adds some complexity and operations.

**Mitigation:** Using a cache (like Redis) with O(1) LPOP/RPUSH operations keeps overhead negligible.

### 3. Memory Usage
The queue must track all active job IDs.

**Mitigation:** Only job IDs are stored, not the full job data. Memory usage is minimal even with thousands of jobs.

---

## Comparison Summary

| Aspect | Without Round-Robin | With Round-Robin |
|--------|---------------------|------------------|
| Small job wait time | Potentially very long | Proportional to job size |
| Large job completion | Fastest possible | Slightly longer |
| User experience | Frustrating for small jobs | Fair for everyone |
| Implementation | Simple FIFO | Slightly more complex |
| Throughput | Same | Same |

---

## Visual Demonstration

Open `demonstration.html` in a browser to see an interactive animation of the round-robin scheduler in action. You can:

- Add jobs of different sizes
- Watch workers process tasks in round-robin order
- Compare side-by-side with traditional FIFO processing
- Adjust simulation speed

