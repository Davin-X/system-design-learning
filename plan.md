system-design/
│
├── README.md                      # How to study + interview strategy
│
├── 00_basics/                     # FOUNDATION (must not skip)
│   ├── what_is_system_design.md
│   ├── client_server_model.md
│   ├── http_https.md
│   ├── rest_api_design.md
│   ├── latency_vs_throughput.md
│   └── scalability_basics.md
│
├── 01_networking/
│   ├── dns.md
│   ├── load_balancer.md
│   ├── reverse_proxy.md
│   ├── cdn.md
│   └── api_gateway.md
│
├── 02_databases/
│   ├── sql_vs_nosql.md
│   ├── indexing.md
│   ├── transactions_acid.md
│   ├── isolation_levels.md
│   ├── replication.md
│   ├── sharding.md
│   └── schema_design_examples.md
│
├── 03_caching/
│   ├── why_caching.md
│   ├── cache_aside.md
│   ├── write_through_write_back.md
│   ├── cache_invalidation.md
│   ├── ttl_and_eviction.md
│   └── redis_design.md
│
├── 04_scalability/
│   ├── vertical_vs_horizontal_scaling.md
│   ├── stateless_services.md
│   ├── auto_scaling.md
│   ├── rate_limiting.md
│   └── backpressure.md
│
├── 05_messaging_async/
│   ├── sync_vs_async.md
│   ├── message_queues.md
│   ├── event_streaming.md
│   ├── kafka_fundamentals.md
│   ├── delivery_semantics.md
│   └── idempotency.md
│
├── 06_consistency_reliability/
│   ├── cap_theorem.md
│   ├── strong_vs_eventual_consistency.md
│   ├── retries_timeouts.md
│   ├── circuit_breaker.md
│   └── fault_tolerance.md
│
├── 07_distributed_systems/
│   ├── leader_election.md
│   ├── consensus_basics.md
│   ├── heartbeats.md
│   ├── clock_skew.md
│   └── distributed_locks.md
│
├── 08_observability/
│   ├── logging.md
│   ├── metrics.md
│   ├── tracing.md
│   ├── alerting.md
│   └── slos_slas.md
│
├── 09_design_patterns/           # VERY IMPORTANT
│   ├── microservices.md
│   ├── saga_pattern.md
│   ├── cqrs.md
│   ├── event_driven_architecture.md
│   ├── bulkhead_pattern.md
│   └── strangler_pattern.md
│
├── 10_case_studies/              # INTERVIEW CORE
│   ├── url_shortener.md
│   ├── notification_system.md
│   ├── rate_limiter.md
│   ├── chat_application.md
│   ├── ecommerce_system.md
│   └── file_storage_system.md
│
├── 11_backend_focus_java/        # YOUR EDGE
│   ├── spring_boot_architecture.md
│   ├── api_versioning.md
│   ├── database_connection_pooling.md
│   ├── thread_management.md
│   └── async_processing_spring.md
│
├── 12_interview_preparation/     # FINAL STAGE
│   ├── how_to_approach_design.md
│   ├── clarifying_questions.md
│   ├── common_followups.md
│   ├── tradeoff_answers.md
│   ├── mock_interviews.md
│   └── checklist_before_interview.md
│
└── resources/
    ├── diagrams/
    ├── cheatsheets/
    └── further_reading.md