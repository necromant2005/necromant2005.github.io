---
title: True Social Metrics Internal Architecture
permalink: /dev/true-social-metrics-internal-architecture/
---

# True Social Metrics Internal Architecture

If I explain True Social Metrics in a simple way, it is basically a pipeline. We did not put everything into one giant service. Instead, we split ingestion, filtering, and aggregation into separate workers. For social analytics that just makes life easier, because writes, reads, and heavy analytics do not scale the same way at all.

<img src="/assets/images/true-social-metrics-architecture.svg" alt="True Social Metrics architecture" class="img-fluid">

The main parts in the diagram are `Input`, `Storage`, `Connector`, and `Aggregator`. The important thing here is that this is not some simple request-response app where one request goes straight through the whole stack. It is queue-driven, async, and each worker group can scale on its own.

## End-To-End Flow

There are really two big flows in the system. The first one starts from the incoming social stream. `worker-input` takes those events and writes them into MongoDB. This part is about ingestion only. It is not trying to answer web requests. Its job is just to eat a lot of incoming data, normalize it a bit, and store it safely.

The second flow starts from the web side. A web request does not go straight into MongoDB. It first goes through `worker-aggregator`, then `worker-connector`, and only then reaches storage. That is important because expensive data loading and preparation are isolated from the user-facing part of the app.

## Input

`worker-input` is the ingestion part. It receives the social stream and stores raw data in the main storage cluster. So this worker group is responsible for the high-throughput part of the whole system:

- receiving social events from external streams
- normalizing or validating them enough to be stored reliably
- writing raw records, profiles, and snapshots into MongoDB
- scaling horizontally when the input rate spikes

The diagram shows autoscaled input workers and mentions `HPA / ECS autoscale by queue size`. In practice that means the system is ready for ugly traffic patterns, burst imports, and random spikes without forcing the whole app to scale together.

## Storage

MongoDB is in the middle and acts as the main storage cluster. It keeps raw data, profiles, snapshots, aggregates, and cache.

We picked MongoDB for a few very practical reasons:

- writes matter more than reads here, because social streams constantly push new data in and we need to save it reliably first
- horizontal scaling is critical, because at terabytes per day you cannot keep solving the problem by moving to a bigger single machine

It also fits social data pretty naturally. Payloads are semi-structured, schemas change over time, and every network has its own weird shape. A document store makes that much less painful.

In this setup MongoDB is not just the landing zone for incoming events. It is also the shared state for the next layers. `worker-connector` reads from it, does the first filtering pass, and prepares data for the next step.

## Connector

`worker-connector` sits between stored raw data and the higher-level metric logic. Its job is pretty straightforward:

- gets data from MongoDB
- performs the first filtering pass
- extracts the subset of records relevant for the next processing stage
- sends prepared work forward to `worker-aggregator`

This layer exists for a very important reason. `Aggregator` can load many sources asynchronously through connectors in parallel. So it does not need to block on one huge source before continuing with everything else.

The nice part is that `Aggregator` deals with prepared source-level work instead of raw post volume directly. So it should not matter too much whether one source had two posts in the last 24 hours or one thousand. `Connector` absorbs the storage reads and the first filtering complexity, and `Aggregator` gets something much cleaner to work with.

Because of that split, `Aggregator` can stay focused on business metrics instead of worrying about raw payload cleanup or storage shape. That makes the aggregation layer much simpler and easier to scale.

## Aggregator

`worker-aggregator` takes the prepared outputs from `worker-connector` and builds the actual aggregated views that the web app needs. This is where the product-facing analytics are built:

- grouped metrics over time
- rollups by profile, campaign, or source
- derived counters and summary views
- cacheable response shapes for the web layer

`Aggregator` also exists for a very practical reason. Some clients have thousands of social accounts tracked over years. We simply cannot afford to load that amount of raw history on every request.

So instead of reloading huge historical datasets again and again, this layer builds reusable aggregated views that are much cheaper to serve. The system can answer from prepared summaries, cacheable result shapes, and already-computed analytics slices.

Because this worker group sits closest to the web boundary, it is the right place to turn filtered source data into product-facing numbers. And since aggregators are autoscaled too, this part can grow independently from ingestion or storage reading.

## User Request Sequence

The architecture diagram shows the boxes, but the sequence diagram shows what really happens at runtime.

<img src="/assets/images/true-social-metrics-user-sequence.svg" alt="True Social Metrics user sequence" class="img-fluid">

The request starts in `Web`, which puts a task into the queue. Then `Aggregator` picks it up and first checks local cache. This part matters a lot, because analytics requests are often repetitive. If we already have the answer cached, we can return it fast and not wake up the deeper part of the pipeline at all.

The cache hit path is the fast path:

1. the user opens a page
2. `Web` enqueues the request
3. `Aggregator` picks it up
4. `Aggregator` checks local cache
5. cached data is returned back to `Aggregator`
6. `Aggregator` returns the result to `Web`

The cache miss path is longer, but the idea is still the same:

1. `Aggregator` sees that local cache does not have the data
2. it enqueues a load request
3. `Connector` picks that job up from the queue
4. `Connector` loads the data from MongoDB
5. the loaded result is sent back through the queue
6. `Aggregator` receives it, updates local cache, and returns the result

This is one of the reasons the system handles load pretty well. Cache hits never touch storage. Cache misses still move through queues, so spikes can be buffered between boundaries instead of immediately hitting every service at once.

## Why The Separation Works

The nice part of this design is that every worker group has one main job:

- `worker-input` handles writes from the external social stream.
- `worker-connector` reads from MongoDB and performs the first filtering stage.
- `worker-aggregator` consumes connector outputs and builds higher-level aggregates.

There is also a super practical infrastructure reason for doing it this way: workers are just simple Docker images with no local durable data. That makes them easy to scale horizontally, because a new instance does not need state migration or some special local setup. It just starts and joins the workflow.

Local cache does not really break that model because it is disposable. If a worker dies, we can just reload the cache. So workers stay replaceable compute units instead of becoming precious stateful machines.

That split gives a few very practical benefits:

1. Independent scaling. A spike in incoming social events does not require the same scaling pattern as a spike in report generation.
2. Better fault isolation. If aggregation slows down, ingestion can still continue storing raw data.
3. Cleaner ownership. Each worker boundary can evolve around one type of problem instead of mixing ingestion, filtering, and aggregation in one code path.
4. Simpler queue semantics. Each stage can acknowledge work only after it finishes its own responsibility.

## Communication Protocols

The diagram mentions `RabbitMQ async communication / queue protocol between worker boundaries`. This part is really important, because it is what keeps the whole system loosely coupled.

RabbitMQ is basically the handoff layer between worker groups:

- web-related requests or jobs are handed to aggregators
- aggregators dispatch downstream work to connectors
- each boundary can process jobs at its own pace

This gives the system buffering. If one stage gets slower, the queue can absorb the pressure for a while instead of making the whole chain fail synchronously.

The queue also makes operations much easier. Because work is buffered between boundaries, we can release new worker versions without breaking the app. One group can be drained, a new one can start, and the system keeps moving.

Queue size is also one of the easiest signals for scaling. If the queue grows, we know we need more workers. If it stays small or drains fast, we can scale down. So the queue is not only transport, it is also one of the main control points for autoscaling.

At the transport level we use `gRPC`. That makes sense for internal service-to-service communication because it is efficient, supports compression well, and is good for sending deltas instead of full payloads over and over. When you move a lot of analytics data internally, that matters a lot.

## Main Scaling Challenges

The hardest part in a system like this is not average load. It is unpredictable spikes that can suddenly go 10x above normal traffic. There are two very obvious cases. The first one is a real-world event spike. If something big happens, like war starting in Iran or some other major global event, social platforms explode. That means way more incoming stream volume, way more writes into `worker-input`, and then more downstream work for filtering and aggregation. The platform has to absorb that without losing data and without scaling everything blindly.

The second one is a business reporting spike. At the end of the month, customers suddenly want more analytics, more summaries, and more reports at the same time. This spike does not come from the social stream itself. It comes from many users asking for data in the same reporting window. So pressure goes mostly to `worker-aggregator` and `worker-connector`, even if ingestion is calm.

That is exactly why the worker split matters. A 10x spike in social input should mostly scale `worker-input`. A 10x spike in reporting demand should mostly scale `worker-aggregator` and `worker-connector`. We do not want one giant bottleneck service trying to handle both patterns.

## Predictive Loading

The system is also smart about when it prepares data. It does not only wait until a user clicks a button. It tries to predict demand and move work earlier when possible.

- start loading user data while the user is still signing in, so before the dashboard is even opened, tasks can already be sitting in the queue
- use historical behavior, so if we know a customer usually needs data at the end of the month, we can preload it before they ask
- run heavier analysis during low-load periods and then recalculate only the diff on a live request, keeping expensive work away from peak traffic

These are simple tricks, but together they help a lot. Users get data faster, and the system avoids some nasty spikes because part of the work is already done before the hot request path even starts.

## Cost-Aware Scaling

Scaling here is not only about performance. It is also about cost. For the base load, we keep a minimal amount of reserved AWS instances. That gives predictable baseline capacity for normal traffic.

When traffic spikes, we try spot instances first because they can be around 10x cheaper than normal on-demand capacity. That makes them a very good first layer for burst scaling.

If spot capacity is not available, then we fall back to normal on-demand instances. So the strategy is:

1. reserved instances for the steady baseline
2. spot instances first for cheap burst capacity
3. on-demand instances as the fallback when spot supply is constrained

This fits the rest of the architecture really well. Stateless workers, disposable cache, and queue buffering make it much easier to use this mixed capacity model safely.

## High Availability

Everything in the system is deployed at least doubled for high availability. We do not want critical parts running as single instances. There is always redundancy, so if one instance dies, another one is already there.

This matters both for reliability and for operations. One worker crash, one host issue, or one bad rollout should not become visible downtime. Together with queues, disposable cache, and stateless workers, this redundancy helps the system keep working even while things are being replaced or interrupted.

## Final Thought

What I like about this architecture is that it is practical at every level. Workers are simple and stateless, so they are easy to scale and replace. Queues give buffering, safer deploys, and a very clear signal for when to scale up or down. `Input`, `Connector`, and `Aggregator` each do one job, so the system can handle very different load patterns without turning into one giant bottleneck.

On top of that, the platform is not only reactive, it is also proactive. It can preload data during sign-in, warm things up before predictable reporting periods, and do heavier analysis during quieter hours so user requests stay lighter. Add spot-first scaling, reserved baseline capacity, and everything deployed with redundancy, and the result is a system that is built not just to work, but to keep working under pressure.
