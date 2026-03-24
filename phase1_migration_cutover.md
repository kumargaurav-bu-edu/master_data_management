# Phase 1: Migration & Cutover Timeline

> Extracted from [phase1_data_ingest_flow.md](phase1_data_ingest_flow.md)

```mermaid
gantt
  title Phase 1: Migration & Cutover Plan
  dateFormat YYYY-MM-DD
  axisFormat %b %d

  section Data Migration
    Export legacy BQ + Elastic records       :m1, 2026-04-01, 7d
    Transform to new schema via normalizer   :m2, after m1, 7d
    Generate panw_customer_id for all records:m3, after m1, 7d
    Load into new BQ + Vector DB             :m4, after m2, 7d
    Validate: counts + 500 record spot-check :m5, after m4, 3d

  section Shadow Mode
    Enable dual-write to old + new pipelines    :s1, after m5, 7d
    Enable dual-read: Elastic vs FSAII          :s2, after s1, 7d
    Automated comparison: precision, recall, latency :s3, after s1, 14d
    Success criteria check: 2 consecutive weeks :crit, s4, after s3, 7d

  section Gradual Cutover
    Route 10pct read traffic to FSAII          :c1, after s4, 3d
    Route 25pct read traffic                   :c2, after c1, 2d
    Route 50pct read traffic                   :c3, after c2, 2d
    Route 100pct read traffic                  :crit, c4, after c3, 3d
    Stop writes to Elasticsearch               :c5, after c4, 2d

  section Decommission
    Elevated monitoring period                 :d1, after c5, 14d
    Archive Elastic indexes to cold storage    :d2, after d1, 3d
    Shut down Elasticsearch cluster            :crit, d3, after d2, 1d
    Remove dual-write code paths               :d4, after d3, 5d
```
