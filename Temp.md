                   ┌─────────────────────┐
                   │     CONTROL PLANE    │
                   │ Workload / Metrics   │
                   │ Prediction Quality   │
                   │ Optimizer / Policy   │
                   └──────────┬──────────┘
                              │
                              │ reconfigure
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ SUBGRAPH 1 — DATA INGESTION                                  │
│                                                              │
│ Crawl Task → Kafka → Crawler Workers → Reddit / Threads      │
│                         ↓                                    │
│                     Raw Data                                 │
└─────────────────────────┬────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ SUBGRAPH 2 — KAFKA INFRASTRUCTURE                            │
│                                                              │
│ Brokers / Partitions / Replication / Producer / Consumer     │
│                                                              │
│ Metrics:                                                     │
│ • Throughput                                                  │
│ • Consumer Lag                                               │
│ • Queueing Time                                              │
│ • Network I/O                                                │
└─────────────────────────┬────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ SUBGRAPH 3 — FLINK REALTIME PROCESSING                       │
│                                                              │
│ Kafka → Network Shuffle → Window → Aggregation → Emit Result │
│                                      ↓                       │
│                                Feature Result                │
│                                                              │
│ Metrics:                                                     │
│ • Processing Latency                                         │
│ • Shuffle Time                                                │
│ • Window Completion Time                                     │
│ • CPU / Memory                                               │
└─────────────────────────┬────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ SUBGRAPH 4 — REALTIME PREDICTION                             │
│                                                              │
│ Feature Result → Lightweight Prediction Model → Result       │
│                                                              │
│ Measure:                                                     │
│ • Prediction Accuracy                                        │
│ • Prediction Latency                                         │
│ • Data Freshness                                             │
│ • Deadline Hit / Miss                                        │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          │ feedback
                          ↓
                ┌───────────────────────┐
                │ ADAPTIVE CONTROLLER   │
                │                       │
                │ Predict workload/QoS  │
                │ + Optimize config     │
                └──────────┬────────────┘
                           │
                           │
                     reconfigure
                           │
             ┌─────────────┴─────────────┐
             ↓                           ↓
        Kafka config                Flink config