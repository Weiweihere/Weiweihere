# Do You Really Need AWS RDS? A Practical Guide for Small Projects

When a cloud bill grows, the first instinct is often to blame the service itself.

Recently, I reviewed the AWS cost of a small internal project. The total bill had been averaging around **€250 per month**, and **Amazon RDS** was the largest cost driver. The database was running PostgreSQL on **db.t4g.large (2 vCPU / 8 GiB RAM)**.

At first glance, the obvious question was:

> Should we move the database from RDS to PostgreSQL running directly on EC2?

But before changing the architecture, I checked the workload.

Over roughly six months, the database showed:

- **CPU utilization:** mostly **3.5–4.5%**, with peaks around **6–6.5%**
- **Freeable memory:** mostly **4.6–5.1 GiB**
- **Database connections:** generally low
- **Read/write IOPS:** low
- **Disk queue depth:** close to zero for most of the period
- **Query latency:** low and stable

The problem was not necessarily RDS.

The database was simply **over-provisioned**.

## 1. Before replacing RDS, check whether you are over-provisioned

Managed services are more expensive than self-managed infrastructure for a reason: part of the price pays for reduced operational work.

RDS can take care of, or simplify:

- backups
- patching
- monitoring
- recovery
- failover options
- database configuration
- integration with other AWS services

Running PostgreSQL on EC2 can reduce infrastructure cost, but it also moves these responsibilities back to you.

So before asking:

> “Is RDS too expensive?”

ask:

> “Am I paying for an RDS instance much larger than my workload actually needs?”

In my case, downsizing from **db.t4g.large (8 GiB RAM)** to **db.t4g.medium (4 GiB RAM)** was the more reasonable first step.

## 2. The metrics I would check before downsizing RDS

A single CPU screenshot is not enough. Database workloads can be memory-, I/O-, or connection-bound even when CPU usage is low.

I would look at at least one to several weeks of CloudWatch history, and preferably longer if the workload has monthly or irregular jobs.

### CPUUtilization

If CPU stays very low for months and even peak periods remain far below the instance capacity, the database may have excess compute capacity.

In this case, utilization stayed around 4% for most of six months.

### FreeableMemory

This is particularly important for PostgreSQL.

A large amount of freeable memory over a long period is a strong sign that the current RAM allocation may be larger than necessary. However, it should not be interpreted as:

```
required memory = total RAM - freeable memory
```

PostgreSQL and the operating system use memory for caching, so memory behavior changes after resizing.

### DatabaseConnections

Check both average and peak connections.

If connection usage is far below the instance limit, connection capacity is probably not the reason you need a larger instance.

If Lambda or another serverless workload connects to the database, also check whether you are using **RDS Proxy** or another connection pool.

### ReadIOPS / WriteIOPS

Low I/O numbers suggest the database is not under meaningful storage pressure.

### DiskQueueDepth

A queue consistently close to zero means storage operations are generally not waiting for capacity.

### SwapUsage

After downsizing, this becomes one of the most important metrics.

If swap usage grows significantly, memory pressure may be too high and the smaller instance may not be appropriate.

### Query latency and errors

Infrastructure metrics are useful, but the application matters more.

After a resize, monitor:

- query latency
- Lambda/database errors
- timeouts
- application response time

A cheaper database is not useful if it makes the application unreliable.

## 3. A simple way to choose an RDS instance

I use a very simple mental model:

| Symptom | What to investigate |
| --- | --- |
| CPU consistently high | More vCPU / query optimization |
| Memory consistently tight | More RAM / query and connection tuning |
| High I/O or queue depth | Storage type, IOPS, query patterns |
| Too many connections | Pooling, RDS Proxy, application connection handling |
| CPU, memory, I/O and connections all low | Consider downsizing |

The last case is easy to miss.

Cloud infrastructure can make over-provisioning feel harmless because resizing later is easy. But unused capacity is still billed every hour.

## 4. RDS vs. PostgreSQL on EC2

For a personal project or small team, the choice is not simply “managed = expensive” and “EC2 = cheap.”

| | Amazon RDS | PostgreSQL on EC2 |
| --- | --- | --- |
| Initial setup | Easier | More manual |
| Backups | Managed options | You manage them |
| Patching | Simplified | You manage OS + database |
| Monitoring | Strong AWS integration | You configure it |
| High availability | Easier to configure | More complex |
| Control | Less | More |
| Infrastructure cost | Usually higher | Can be lower |
| Operational burden | Lower | Higher |

### RDS is often a good fit when:

- the database contains important business data
- recovery and backups matter
- several people depend on the application
- you do not want to maintain PostgreSQL and the underlying OS
- downtime has a meaningful cost

### Self-managed PostgreSQL on EC2 can make sense when:

- it is a personal or experimental project
- cost is the main constraint
- you are comfortable maintaining PostgreSQL
- you can tolerate some downtime
- you have a clear backup and recovery plan
- you genuinely need the additional control

## 5. The lesson from this case

The most useful lesson for me was not:

> “RDS is too expensive.”

It was:

> **Before replacing a managed service with cheaper infrastructure, first check whether you are simply over-provisioned.**

In this case, moving directly from RDS to EC2 would have introduced additional operational work before addressing the simpler problem: the database instance was much larger than the observed workload required.

The safer sequence is:

1. Measure the workload over a meaningful period.
2. Right-size the existing managed service.
3. Monitor the smaller configuration.
4. Only then compare the remaining RDS premium with the operational cost of managing the database yourself.

For small teams, that distinction matters. The cheapest infrastructure is not always the lowest-cost system once maintenance time and reliability are included.

---

*Note: AWS pricing varies by region, database engine, deployment model, storage configuration, and time. The numbers above describe one real workload pattern and should not be treated as universal sizing thresholds.*
