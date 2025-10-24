---
layout: page
title: Cronq - Cron-as-a-Service in Go
permalink: /projects/cronq

---
layout: page
title: "Cronq - Cron-as-a-Service in Go"
permalink: /projects/cronq/
---

# Cronq - Cron-as-a-Service

A distributed job scheduler written in Go.  
It runs HTTP callbacks at scheduled times, similar to a managed cron service.

---

## 1. Motivation - What is a Cron Job?

Before moving into Go and containers, I wanted to deeply understand the **principle of scheduling**.  
In Linux, `cron` is a daemon that runs background tasks at fixed times using entries like:

```text
*/5 * * * * /usr/bin/backup.sh
````
> "Run /usr/bin/backup.sh every 5 minutes, no matter what day or hour it is."

It’s simple, reliable - but local.

In a **cloud-native world**, we want something similar:

* trigger **HTTP callbacks**, not shell scripts
* scale horizontally
* persist jobs in a **database**, not in `/etc/crontab`
* and keep full observability (logs, metrics, retries)

---

## 2. From Local Cron to Cloud Architecture

In a monolithic system, cron is fine.
But in a distributed system (microservices, Kubernetes, serverless, etc.),
you want a **central scheduler** that:

* stores jobs in a database (PostgreSQL)
* determines when they are due
* dispatches execution tasks via a queue (Redis)
* executes them via workers
* and retries on failure

---

## 3. Starting Small - The First Go Service

To begin, I built the API service with a simple health endpoint:

```go
// internal/api/handlers/health.go
package handlers

import (
    "net/http"
)

func HealthzHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
}
```

Registered via the `chi` router:

```go
router.Get("/healthz", handlers.HealthzHandler)
```

When I run the API:

```bash
go run ./services/api/
```

And hit:

```bash
curl http://localhost:8080/healthz
```

I get:

```text
ok
```

### Why this first?

Because before building complex logic, it’s crucial to see something running and responding -
this gives you a working foundation and a positive feedback loop.

---

## 4. Adding Persistence - Why PostgreSQL?

The next step was defining how to store jobs.

I chose **PostgreSQL** because:

* It’s reliable and battle-tested
* Has strong ACID guarantees
* Integrates beautifully with Go’s `pgxpool` driver

Example `jobs` table:

```sql
CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    name VARCHAR(128) UNIQUE NOT NULL,
    cron VARCHAR(64),
    run_at TIMESTAMPTZ(0),
    callback_url TEXT NOT NULL,
    updated_at TIMESTAMPTZ(0) NOT NULL DEFAULT NOW(),
    max_retries SMALLINT NOT NULL DEFAULT 10 CHECK (max_retries >= 0),
    retry_backoff_ms INTEGER NOT NULL DEFAULT 5000 CHECK (retry_backoff_ms >= 0),
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    CONSTRAINT chk_cron_xor_runat CHECK ((cron IS NULL) <> (run_at IS NULL))
);
```

### Design insight

`cron` and `run_at` are mutually exclusive -
one for repeating, one for one-time jobs.
The constraint `chk_cron_xor_runat` enforces this rule directly in the DB layer.

---

## 5. Why Docker Now Makes Sense

At this point, I had my API and a database schema -
but I didn’t want to install PostgreSQL locally and clutter my system.

That’s where **Docker Compose** comes in:

**coming soon...**

---