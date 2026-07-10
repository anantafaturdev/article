---
title: "Research: Cost-Optimization RDS for Staging Environment"
slug: research-cost-optimization-rds-for-staging-environment
date_published: 1970-01-01T00:00:00.000Z
date_updated: 2026-03-09T02:28:05.000Z
draft: true
---

## Summary

A comparative performance analysis between AWS RDS `db.t3.large` (Intel Xeon) and `db.t4g.large` (Graviton2/ARM) PostgreSQL instances demonstrates that **Graviton2 provides significantly better performance at lower cost**. For most realistic workloads, t4g.large delivers **20-40% higher throughput** with **15-30% lower latency** at scale, while being **~20% cheaper** based on AWS pricing.

## Performance Benchmark Results

### Key Performance Metrics (at 32 concurrent clients)

Workload

Metric

t3.large (Intel)

t4g.large (Graviton2)

Improvement

**Read-Write (default)**

TPS 

866 

1,115 

**+28.8%**

 

Avg Latency 

36.9ms 

28.6ms 

**-22.5%**

**Read-Only (select)**

TPS 

6,435 

8,230 

**+27.9%**

 

Avg Latency 

4.94ms 

3.85ms 

**-22.1%**

**Write-Intensive (update)**

TPS 

1,212 

1,579 

**+30.3%**

 

Avg Latency 

26.3ms 

20.2ms 

**-23.2%**

Figure
![](__GHOST_URL__/content/images/2026/02/image-5.png)s
### Performance Scaling Analysis

**Graviton2 shows superior scaling characteristics:**

1. **Better concurrency handling**: t4g.large maintains lower latency increase as client load grows
2. **Higher saturation points**: Maximum TPS achieved at higher client counts
3. **More consistent performance**: Smaller variance between min/max TPS across test iterations

#### Read-Write Workload Scaling

Clients | t3.large TPS | t4g.large TPS | Graviton Advantage
--——|--————|—————|-------------------
16      | 776          | 900           | +16.0%
32      | 866          | 1,115         | +28.8%
64      | 977          | 1,250         | +27.9%
128     | 1,069        | 1,358         | +27.0%
`192     | 1,038        | 1,357         | +30.7%`

### Latency Analysis

Graviton2 consistently delivers lower latency, particularly under higher loads:

- At 64 clients: **51.1ms vs 65.4ms** (22% better)
- At 128 clients: **93.7ms vs 119.3ms** (21% better)
- At 192 clients: **140.9ms vs 184.1ms** (23% better)

## Cost-Performance Analysis

### AWS Pricing (ap-southeast-3 Region)

- **t3.large**: **$0.252/hour**
- **t4g.large**: **$0.226/hour**
- **Cost Savings**: **10.3%** lower hourly cost

### Cost-Per-TPS Efficiency (Updated)

Workload

Peak TPS

t3.large Cost/TPS

t4g.large Cost/TPS

Efficiency Gain

**Read-Write**

t3: 1,069
t4g: 1,358

**$0.236/million**
($0.252×3600/1069)

**$0.150/million**
($0.226×3600/1358)

**+57.3%**

**Read-Only**

t3: 6,511
t4g: 8,231

**$0.139/million**
($0.252×3600/6511)

**$0.099/million**
($0.226×3600/8231)

**+40.8%**

**Write-Intensive**

t3: 1,504
t4g: 1,937

**$0.603/million**
($0.252×3600/1504)

**$0.420/million**
($0.226×3600/1937)

**+43.6%**

**Formula**: `(Instance Cost per Hour × 3600 seconds) / Peak TPS = Cost per million transactions`

## Benchmark Methodology

### Test Configuration

- **Database**: PostgreSQL 17.4, 10GB dataset (scale factor 500)
- **Duration**: 120 seconds per test, 3 iterations per configuration  
- **Concurrency**: 1 to 192 concurrent clients
- **Workloads**:
- `default`: TPC-B like read/write mix (`pgbench` default)
- `select-only`: Read-intensive operations (`pgbench -S`)
- `simple-update`: Write-intensive operations (`pgbench -N`)

### Actual Benchmark Commands Used

**Database Setup (per instance):**

`*# Full initialization with 10GB dataset*
pgbench -h <hostname> -U benchmark -i -s 500 --unlogged -I dtgp benchmarkdb`

**Benchmark Execution Script Logic:**

`*# For each workload and client count (1-192), 3 iterations:*
case $workload in
  "default")    BENCH_OPTS="" ;;
  "select-only") BENCH_OPTS="-S" ;;
  "simple-update") BENCH_OPTS="-N" ;;
esac
*# Thread calculation: clients/8, min 1, max 16*
threads=$(( clients / 8 ))
[ $threads -lt 1 ] && threads=1
[ $threads -gt 16 ] && threads=16
*# Actual execution command:*
pgbench -h "$DB_HOST" -U "$DB_USER" \
  -c "$clients" -j "$threads" -T 120 \
  -M prepared -r -P 10 $BENCH_OPTS \
  "$DB_NAME"`

## Conclusion

**Graviton2-based RDS instances (t4g.large) provide superior price-performance for PostgreSQL workloads.** The benchmark data clearly demonstrates:

1. **Better performance**: 28-30% higher throughput across workloads
2. **Lower latency**: 20-25% faster response times under load
3. **Cost efficiency**: 48-50% better cost per transaction

**Recommendation**: Proceed with migration of **staging environments** to Graviton2 instances.

 

Raw Results;

[https://github.com/anantaonrey/devops-internship/tree/master/rds-benchmark/results/t3-large](https://github.com/anantaonrey/devops-internship/tree/master/rds-benchmark/results/t3-large)

[https://github.com/anantaonrey/devops-internship/tree/master/rds-benchmark/results/t4g-large](https://github.com/anantaonrey/devops-internship/tree/master/rds-benchmark/results/t4g-large)
