---
title: True Social Metrics Internal Architecture
permalink: /dev/true-social-metrics-internal-architecture/
---

# True Social Metrics Internal Architecture

True Social Metrics looks like a pipeline that separates ingestion, filtering, and aggregation into different worker boundaries instead of pushing everything into one service. That is a good fit for social analytics because write volume and query volume usually scale differently, and each stage has a very different responsibility.

<img src="/assets/images/true-social-metrics-architecture.svg" alt="True Social Metrics architecture" class="img-fluid">

The architecture in the diagram breaks into four main zones: `worker-input`, `MongoDB`, `worker-connector`, and `worker-aggregator`. The interesting part is that the system is not just a straight request-response application. It is a queue-driven pipeline where worker groups can scale independently and communicate asynchronously.

## End-To-End Flow

There are really two flows in the system.

The first flow starts from the incoming social data stream. `worker-input` receives those events and writes them into MongoDB. This is the ingestion side of the platform. Its job is not to answer the web application directly. Its job is to absorb a high-volume stream, normalize it enough to persist safely, and keep storage up to date.

The second flow starts from the web side. A web request does not appear to hit MongoDB directly in the diagram. Instead it goes through `worker-aggregator`, then `worker-connector`, and only then into MongoDB. That tells us the read path is mediated by background-style worker boundaries, likely using queues and asynchronous tasks to isolate expensive data preparation from the user-facing layer.

## User Request Sequence

The architecture diagram shows the static boundaries, but the request sequence makes the runtime behavior much easier to understand.

<img src="/assets/images/true-social-metrics-user-sequence.svg" alt="True Social Metrics user sequence" class="img-fluid">

The request starts in `Web`, which places an asynchronous job into the queue. `Aggregator` picks that job up and checks a local cache first. This is an important optimization because many analytics requests are repetitive. If the result is already cached, `Aggregator` can return the data immediately without touching the deeper pipeline.

The cache hit path is the fast path:

1. the user opens a page
2. `Web` enqueues the request
3. `Aggregator` picks it up
4. `Aggregator` checks local cache
5. cached data is returned back to `Aggregator`
6. `Aggregator` returns the result to `Web`

The cache miss path is longer, but it keeps the same async pattern:

1. `Aggregator` sees that local cache does not have the data
2. it enqueues a load request
3. `Connector` picks that job up from the queue
4. `Connector` loads the data from MongoDB
5. the loaded result is sent back through the queue
6. `Aggregator` receives it, updates local cache, and returns the result

This sequence explains why the system can handle load better than a fully synchronous chain. Cache hits never need to touch storage. Cache misses still move through queues, so spikes can be buffered between boundaries instead of immediately overwhelming every service at once.

## Input

`worker-input` is the ingestion boundary. It receives the social data stream and stores raw data in the main storage cluster.

That makes this worker group responsible for the high-throughput part of the system:

- receiving social events from external streams
- normalizing or validating them enough to be stored reliably
- writing raw records, profiles, and snapshots into MongoDB
- scaling horizontally when the input rate spikes

The diagram explicitly shows autoscaled input workers and mentions `HPA / ECS autoscale by queue size`. That is a strong sign the system is built to handle uneven social traffic, bursty imports, or source-specific spikes without coupling ingestion capacity to the rest of the application.

## Storage

MongoDB sits in the middle as the main storage cluster. According to the diagram, it stores raw data, profiles, snapshots, aggregates, and cache.

MongoDB was selected for a few practical reasons.

First, this is a write-heavy system where writes are more important than reads. Social streams generate a constant flow of incoming records, and the platform has to persist them reliably before anything else happens. In that kind of workload, optimizing the write path matters more than building around a classic read-first database model.

Second, horizontal scaling is critical. If the system processes terabytes of data per day, storage cannot depend on scaling only upward with a bigger single machine. It needs a storage model that can scale across nodes as data volume and write throughput keep growing.

It is also a practical choice for a social metrics product because the incoming data is typically semi-structured, source schemas evolve over time, and profile or post payloads often differ across networks. A document store lets the platform persist source-shaped data while still supporting later processing stages.

In this setup, MongoDB is not only the landing zone for incoming events. It is also the shared state for downstream workers. `worker-connector` reads from it, applies first-stage filtering, and prepares data for the next layer.

## Connector

`worker-connector` is the boundary between stored raw data and higher-level metric production.

Based on your description and the diagram, this layer:

- gets data from MongoDB
- performs the first filtering pass
- extracts the subset of records relevant for the next processing stage
- sends prepared work forward to `worker-aggregator`

This separation matters for another reason too. `Aggregator` can load many sources asynchronously through connectors in parallel. That means it does not have to block on one large source before it can continue working on others.

The practical advantage is that the aggregator deals with prepared source-level units of work instead of raw post volume directly. In other words, it should not matter much whether one source produced two or three posts in the last 24 hours or one thousand posts in the same period. `Connector` absorbs the storage read and first filtering complexity, and `Aggregator` can keep working with a more stable abstraction for downstream analytics.

If filtering and source-specific reading logic live in `worker-connector`, then `worker-aggregator` can stay focused on business metrics instead of worrying about storage structure or raw payload cleanup. That usually makes the aggregation layer simpler, more reusable, and easier to scale.

## Aggregator

`worker-aggregator` consumes the prepared outputs from `worker-connector` and builds the actual aggregated views used by the web application.

This is where the platform likely computes the analytics objects people care about:

- grouped metrics over time
- rollups by profile, campaign, or source
- derived counters and summary views
- cacheable response shapes for the web layer

Because this worker group sits closest to the web boundary, it is a good place to turn filtered source data into product-facing numbers. The diagram shows multiple autoscaled aggregator workers, which suggests aggregation jobs are parallelizable and can be expanded independently from ingestion or connector throughput.

## Why The Separation Works

The cleanest part of this design is that each worker boundary has one primary concern.

- `worker-input` handles writes from the external social stream.
- `worker-connector` reads from MongoDB and performs the first filtering stage.
- `worker-aggregator` consumes connector outputs and builds higher-level aggregates.

There is also a very practical infrastructure reason for using workers this way: they are simple Docker images with no local durable data. That makes them easy to scale horizontally because new instances do not need state migration, local synchronization, or special bootstrap logic beyond joining the queue-driven workflow.

The local cache does not break that model because it is disposable. If a worker disappears, the cache can be reloaded. That means the system can treat workers as replaceable compute units instead of stateful machines, which is exactly what you want when scaling up quickly during traffic spikes.

That kind of split gives the platform a few practical benefits:

1. Independent scaling. A spike in incoming social events does not require the same scaling pattern as a spike in report generation.
2. Better fault isolation. If aggregation slows down, ingestion can still continue storing raw data.
3. Cleaner ownership. Each worker boundary can evolve around one type of problem instead of mixing ingestion, filtering, and aggregation in one code path.
4. Simpler queue semantics. Each stage can acknowledge work only after it finishes its own responsibility.

## Communication Protocols

The diagram notes `RabbitMQ async communication / queue protocol between worker boundaries`. That is important because it explains how the system avoids tight coupling between the stages.

RabbitMQ likely acts as the handoff mechanism between worker groups:

- web-related requests or jobs are handed to aggregators
- aggregators dispatch downstream work to connectors
- each boundary can process jobs at its own pace

This gives the architecture buffering and elasticity. When one stage is slower, the queue can absorb pressure temporarily instead of forcing synchronous timeouts across the entire chain.

The queue is also important operationally. Because work is buffered between boundaries, workers can be updated and replaced without breaking the application. One group of workers can be drained, a new release can be started, and the system continues processing through the queue instead of depending on one fixed in-memory execution chain.

Queue size also becomes a practical scaling signal. If the queue grows, the system knows it needs more worker capacity. If the queue stays small or drains quickly, it can scale back down. That makes the queue not just a transport mechanism, but also one of the main control points for autoscaling decisions.

At the transport layer, the main protocol is `gRPC`. That is a sensible choice here because it reduces overhead compared with more verbose protocols and works well for internal service-to-service communication. In a pipeline like this, `gRPC` is especially useful for compression and for transferring deltas instead of repeatedly sending full payloads. That matters when the system is moving large amounts of analytics data internally and wants to keep both latency and bandwidth under control.

## Final Thought

What makes this architecture effective is not just the choice of MongoDB or RabbitMQ. It is the decision to treat ingestion, filtering, and aggregation as separate worker responsibilities. For a social analytics system like True Social Metrics, that is a sensible way to keep the pipeline scalable while still letting the web layer depend on prepared, aggregated results instead of raw social data.

## Main Scaling Challenges

The biggest operational challenge in a system like this is not average load. It is unpredictable load with spikes that can easily jump 10x above normal traffic.

There are two obvious cases.

The first case is a real-world event spike. If something major happens, like the start of a war in Iran or another globally visible breaking event, social platforms explode with activity. That means much more incoming stream volume, more writes into `worker-input`, more raw data in MongoDB, and then more downstream work for filtering and aggregation. The platform has to absorb that surge without losing data and without forcing every part of the system to scale in lockstep.

The second case is a business reporting spike. At the end of the month, customers usually need more analytics data, more summaries, and more reporting jobs at the same time. This is not caused by the public data stream itself. It is caused by many users asking for aggregates and dashboards in the same reporting window. That puts more pressure on `worker-aggregator` and `worker-connector`, even if ingestion volume is relatively stable.

This is exactly why the worker boundaries matter. A 10x spike in social input should mostly scale `worker-input`. A 10x spike in reporting demand should mostly scale `worker-aggregator` and `worker-connector`. By keeping those concerns separate and using queues between them, the platform can react to different spike patterns without turning the whole architecture into one large bottleneck.

## Cost-Aware Scaling

The scaling model is not only about performance. It is also about cost.

For the base load, the system can run on a minimal amount of reserved AWS instances. That gives predictable baseline capacity for the normal steady-state workload.

When traffic spikes, scaling prefers spot instances first because they can be around 10x cheaper than regular on-demand capacity. That makes them a very effective first layer for burst scaling during event-driven traffic or reporting peaks.

If spot capacity is not available, the system can then fall back to general on-demand instances. So the capacity strategy is layered:

1. reserved instances for the steady baseline
2. spot instances first for cheap burst capacity
3. on-demand instances as the fallback when spot supply is constrained

That matches the rest of the architecture well. Stateless workers, disposable cache, and queue-based buffering make it much easier to use a mixed instance strategy without turning infrastructure changes into application risk.

## High Availability

Everything in the system is deployed with at least double capacity for high availability. In practice, that means critical components are not running as single instances. There is always redundancy, so if one instance fails, another one is already there to keep processing traffic.

This matters for both reliability and operations. It reduces the chance that one worker loss, one host failure, or one rollout issue turns into visible downtime. Combined with queues, disposable cache, and stateless workers, that redundancy helps the platform keep working even while instances are being replaced, scaled, or interrupted.
