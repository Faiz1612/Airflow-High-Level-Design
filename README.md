# Airflow - High Level Design

## Functional Requirements

### Core Requirements

* Users should be able to schedule jobs:

  * For immediate execution
  * At a specific future time
  * On a recurring schedule (e.g., every day at 10:00 AM)
* Users should be able to monitor the status of their scheduled jobs.

---

## Non-Functional Requirements

1. **High Durability**

   * User jobs must not be lost due to system or infrastructure failures until the job has been successfully completed.

2. **Strong Consistency for Status API**

   * The status API should always reflect the latest and exact state of the job execution.

3. **Low Latency**

   * API response time should remain below **200 ms**.

4. **Execution SLA**

   * Jobs should start execution within **2 seconds** of their scheduled time.

5. **Peak Traffic Handling**

   * The system should be capable of handling traffic spikes and large volumes of scheduled jobs.

---

## Core Entities

### User

Represents a system user who creates and monitors jobs.

### Job

Represents a scheduled task along with its execution metadata.

---

## APIs

### 1. Create Job

**Endpoint**

```http
POST /v1/job
```

**Request**

```json
{
  "name": "Daily Report Generation",
  "scheduled_time": "2026-06-15T10:00:00Z",
  "api_to_hit": "https://example.com/report"
}
```

**Response**

Returns the created job object.

---

### 2. Get Job Status

**Endpoint**

```http
GET /v1/job/{job_id}
```

**Response**

Returns one of the following statuses:

```text
PENDING
ONGOING
FAILED
COMPLETED
```

---

# High-Level Design

## Q: How will jobs be scheduled and executed?

### Initial Design

1. A user sends a job creation request.
2. The request passes through a Load Balancer and Authentication layer.
3. The Job Service performs basic validation and stores the job in the Job Database with status `PENDING`.
4. A Scheduler Service runs periodically (e.g., every 30 seconds).
5. The Scheduler queries the Job Database for jobs whose scheduled execution time has arrived.
6. Eligible jobs are executed.
7. Based on the execution outcome, the Scheduler updates the job status in the database.

This provides a simple and straightforward scheduling mechanism that can be enhanced further as scale increases.

---

## Q: How will users view the status of their scheduled or executed jobs?

1. Users call the Job Service using the Job Status API.
2. The Job Service retrieves the latest status from the database.
3. Job status updates are continuously performed by the Scheduler Service during execution.

This allows users to track the current state of their jobs at any time.

---

## Q: How can we scale job execution to support up to 10,000 concurrent jobs?

### Scaling Strategy

Instead of processing jobs one at a time, the Scheduler can fetch multiple eligible jobs using range queries and push them into a queue.

Key optimizations include:

* Horizontally scale both Job Service and Scheduler Service instances.
* Batch-fetch jobs that are ready for execution instead of processing individual records.
* Distribute workload across multiple Scheduler instances.
* Enable auto-scaling using cloud-native services (e.g., AWS Auto Scaling).

### Redis-Based Coordination

Redis can be used as a coordination layer for scheduling.

Since Redis is single-threaded, operations on a particular data structure are executed atomically. One Scheduler instance can retrieve and remove eligible jobs, ensuring that other Scheduler instances do not process the same job simultaneously.

This helps prevent duplicate execution while supporting horizontal scaling.

---

## Q: How should we handle retrying jobs that fail during execution?

### Retry Mechanism Using SQS

Amazon SQS can be used to provide reliable retry capabilities.

1. Scheduler instances consume jobs from the queue.
2. If processing succeeds, the message is acknowledged and removed.
3. If a Scheduler instance crashes or job execution fails, the message becomes visible again after the configured visibility timeout.
4. Another Scheduler instance can then pick up the same job for reprocessing.

### Retry Policy

* Maintain a retry counter for each job.
* Retry failed jobs up to a predefined limit (e.g., 3–5 attempts).
* If all retry attempts fail, update the job status to `FAILED`.

This approach provides fault tolerance and ensures reliable job execution even in the presence of service failures.

### Final Design
![HLD](https://github.com/user-attachments/assets/c74d5b74-8551-4eb7-83f9-ef468a2c556a)
