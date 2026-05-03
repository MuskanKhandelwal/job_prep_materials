# System Design Fundamentals


## Vertical Scaling

Vertical scaling (also called "scaling up") means making a single server more powerful — adding more CPU, RAM, faster disks, or better network cards to one machine.

Think of it like upgrading your laptop: instead of buying a second laptop, you put more RAM into the one you have. If your web app is slow, you move from a server with 8GB of RAM to one with 64GB.

The appeal is simplicity. Your code doesn't change. There's no coordination between machines, no network latency between servers, no distributed systems headaches. For small to medium applications, this is often the right first move.

The catch: there's a ceiling. You can't keep adding RAM forever, and high-end hardware gets exponentially more expensive. A server with 1TB of RAM costs *far* more than 16 servers with 64GB each. You also have a single point of failure — if that one beefy machine dies, your whole app goes down.

## Horizontal Scaling

Horizontal scaling ("scaling out") means adding *more* servers rather than bigger ones. Instead of one giant machine handling all traffic, you have ten modest machines sharing the load.

This is how Google, Netflix, and most large-scale systems work. It's cheaper at scale (commodity hardware), more fault-tolerant (one server dying doesn't kill the system), and theoretically has no upper limit — you just keep adding boxes.

The cost is complexity. Now you need to figure out: how do incoming requests get distributed across servers? How do servers share state? What happens when one fails mid-request? This is where the rest of the concepts below come in — they're largely tools for making horizontal scaling work.

## Load Balancing

Once you have multiple servers, something needs to decide *which* server handles each incoming request. That's a load balancer — a piece of infrastructure that sits between your users and your servers, distributing traffic.

Common strategies include round-robin (server 1, then 2, then 3, then back to 1), least-connections (send to whichever server is least busy), and IP hash (the same user always hits the same server, which helps with session data).

Load balancers also do health checks — they ping each server periodically and stop routing traffic to any server that's not responding. This is how horizontal scaling achieves fault tolerance: if a server crashes, the load balancer notices and quietly routes around it. Users never know.

A subtlety worth understanding: if your servers store session data in their own memory ("user X is logged in"), then a user bouncing between servers will appear logged out randomly. The fix is either sticky sessions (keep a user pinned to one server) or — better — store session data somewhere external like a shared cache, so any server can handle any request. This is called making your servers "stateless," and it's foundational to scaling.

## Caching

Caching means storing the results of expensive operations so you can reuse them instead of recomputing. If 10,000 users load your homepage and each request hits the database to fetch the same article list, you're doing 10,000x more database work than you need to.

Caches exist at many levels. Browser caches store assets locally so users don't re-download your logo. CDNs (like Cloudflare) cache static content in data centers around the world, so a user in Tokyo gets your CSS file from a Tokyo server, not your origin server in Virginia. Application caches (Redis, Memcached) store computed results — rendered pages, database query results, API responses — in fast in-memory stores.

The hard problem in caching is *invalidation*: when the underlying data changes, how do you make sure the cache doesn't keep serving stale results? Common patterns include time-based expiration (cache for 5 minutes, then refetch), write-through (update cache and database together), and explicit invalidation (when data changes, delete the cache entry). There's a famous quote in computer science: "There are only two hard things: cache invalidation and naming things." It's only half a joke.

## Database Replication

Databases often become the bottleneck before web servers do, because they're stateful and harder to scale horizontally. Replication is one answer: you keep multiple copies of your database in sync.

The most common pattern is primary-replica (historically called master-slave). One database — the primary — handles all writes. Several replicas continuously copy data from the primary and handle reads. Since most applications read far more than they write (think 90% reads, 10% writes for a typical web app), this lets you scale reads dramatically by adding more replicas.

The tradeoff is *replication lag*. When you write to the primary, it takes a fraction of a second (sometimes longer) for that write to propagate to replicas. So a user who posts a comment and immediately refreshes might briefly not see their own comment if the refresh hits a replica that hasn't caught up. Applications either accept this, route the user's own reads to the primary, or use other tricks.

There's also multi-primary replication where multiple nodes accept writes, but this introduces conflict resolution problems (what if two nodes write conflicting values at the same time?) and is much harder to get right.

## Database Partitioning

When your dataset gets large enough that even a single primary can't handle the write load — or simply can't fit on one machine — you partition. Partitioning (also called sharding) splits your data across multiple databases, each holding a subset.

For example, if you have a billion users, you might put users with IDs 1–250M on shard A, 250M–500M on shard B, and so on. Each shard is independent and handles only its slice of the data and traffic. This lets writes scale horizontally — something replication alone can't do.

The decision of *how* to partition is critical and often hard to change later. You partition by a key (user ID, geography, customer ID), and that key determines where data lives. Choose poorly and you get hot spots — one shard handling 80% of traffic while others sit idle. Choose a key tied to a single user's data, and queries that span users (analytics, search) become painful because they have to query every shard and merge results.

Partitioning also makes things like joins, transactions, and foreign key constraints much harder, because the data you need might live on different machines. Many large-scale systems explicitly give up these features in exchange for the scalability gain.

## How they fit together

A typical scaled architecture uses all of these together: users hit a load balancer, which distributes requests to a fleet of horizontally-scaled stateless web servers, which check a cache before reading from a database that's both replicated (for read scaling) and partitioned (for write scaling and storage). Vertical scaling is still happening underneath — each individual server is reasonably beefy — but the overall system grows by adding more boxes, not bigger ones.


Load balancing is essentially the technique that *makes* horizontal scaling work, so we'll see them weave together throughout.

# Part 1: Horizontal Scaling

## The core idea, revisited

Horizontal scaling means handling more load by adding more machines rather than upgrading one machine. But the phrase hides a lot of subtlety. The real question horizontal scaling forces you to answer is: *how do many machines coordinate to look like one system to the user?*

If you just spin up ten copies of your web server, you immediately face questions like: which server handles a given request? If a user logs in on server 3, why does server 7 not know they're logged in? If two servers try to update the same record, who wins? Horizontal scaling is less about "adding servers" and more about *redesigning your system so that adding servers actually helps*.

## Stateless vs stateful services

The single most important concept in horizontal scaling is **statelessness**. A stateless server holds no information between requests — every request contains everything needed to process it, and any server in the fleet can handle any request interchangeably.

Compare two designs for a login system:

**Stateful (hard to scale):** When you log in, the server stores `{user: alice, sessionId: abc123}` in its own memory. Future requests with cookie `abc123` work only if they hit *that specific server*. Add a second server and half your users randomly appear logged out.

**Stateless (scales easily):** When you log in, the server creates a signed token (like a JWT) containing your user info and hands it to you. The server stores nothing. Every future request includes the token, and *any* server can verify it independently. Add a hundred servers — doesn't matter, they're all interchangeable.

The pattern generalizes: push state out of your application servers and into shared infrastructure (databases, caches, object storage). Application servers become disposable workers. This is why "make your services stateless" is repeated like a mantra in scaling discussions.

## What horizontal scaling actually looks like in practice

A typical web tier might have an auto-scaling group: a pool of identical servers behind a load balancer, with rules like "if average CPU exceeds 70% for 5 minutes, add 2 more servers; if it drops below 30%, remove 2." The number of servers breathes with traffic. Black Friday spike? Pool grows from 20 to 200. Quiet Tuesday at 3am? Back down to 20. You only pay for what you use.

For this to work cleanly, server startup must be fast and automated. New servers need to come online, register themselves with the load balancer, pass health checks, and start taking traffic — ideally in under a minute. This is why containerization (Docker) and orchestration (Kubernetes) became dominant: they make spinning up identical, ready-to-serve instances trivial.

## What horizontal scaling doesn't fix

It's tempting to think horizontal scaling is a universal answer, but it has real limits.

It doesn't help with stateful bottlenecks. If all your servers talk to one database and the database is the bottleneck, adding web servers just creates more pressure on the same database. You'd need to scale the database itself (replication, partitioning), which we covered earlier.

It can introduce coordination overhead. If servers need to agree on something (who holds a lock, what the latest value is), more servers means more coordination, and at some point that coordination becomes the bottleneck. Distributed systems literature is full of this pattern.

It changes failure modes. With one server, it's either up or down. With a hundred servers, you have partial failures: 3 are slow, 1 is returning errors, 2 think it's last Tuesday. Your system has to be designed to tolerate this kind of messy, partial degradation — what people call "graceful degradation."

# Part 2: Load Balancing

## The job

A load balancer sits between clients and your server pool. Its job sounds simple — distribute requests across servers — but it's doing several things at once: choosing a target server, monitoring health, handling failures, and often terminating SSL/TLS connections, rate-limiting, and more.

Conceptually, you can think of it as a smart receptionist for your server fleet. Requests come in the front door, the receptionist looks at who's available and not overloaded, and routes the request to the right desk.

## Layer 4 vs Layer 7

Load balancers operate at one of two networking layers, and the distinction matters.

**Layer 4 (transport layer)** load balancers route based on IP address and port. They don't look at the contents of the request — they just see TCP packets and forward them. This makes them very fast and protocol-agnostic (they work for HTTP, gRPC, raw TCP, anything), but limited in what routing decisions they can make.

**Layer 7 (application layer)** load balancers understand HTTP. They can see URLs, headers, cookies, and request bodies. This lets them make intelligent decisions: "send `/api/*` requests to the API servers, send `/images/*` to the image servers, send requests with cookie `experiment=B` to the new version." They can also do things like compress responses, rewrite URLs, and inject headers.

Layer 7 is more flexible and more common for web applications today (think nginx, HAProxy in HTTP mode, AWS Application Load Balancer). Layer 4 still wins when you need raw throughput or are balancing non-HTTP traffic (AWS Network Load Balancer, for example).

## Distribution algorithms

How does the load balancer pick which server gets a request? Common strategies:

**Round-robin** rotates through servers in order: request 1 to server A, request 2 to B, request 3 to C, request 4 back to A. Simple and fair if all requests take similar time and all servers are equally powerful — which is rarely true in practice.

**Least connections** sends each new request to whichever server currently has the fewest active connections. This naturally adapts when some requests take longer than others. A server stuck on a slow query won't keep getting piled with new work.

**Weighted variants** of either algorithm let you say "server A is twice as powerful as server B, send it twice the traffic." Useful when your fleet isn't homogeneous, or during gradual rollouts of new instance types.

**IP hash / consistent hashing** routes based on a hash of the client's IP (or some request property), so the same client tends to hit the same server. This was historically used for sticky sessions, but consistent hashing has a more important modern use: routing in distributed caches and databases, where you want the same key to consistently land on the same node so the cache stays warm.

**Least response time** combines connection count and how fast a server has been responding lately. More sophisticated, more accurate, slightly more expensive to compute.

For most web applications, least connections is a sensible default. Round-robin is fine if your requests are uniform.

## Health checks and failure handling

A load balancer continuously checks whether each server is alive — typically by hitting a `/health` endpoint every few seconds. If a server fails several checks in a row, the load balancer marks it unhealthy and stops sending traffic. When it recovers, traffic resumes.

This is what gives horizontally-scaled systems their fault tolerance. A server crashes? Within seconds, the load balancer notices and routes around it. Users see no impact, or maybe a single failed request that their browser retries. Compare this to a single-server setup, where a crash means everyone is down until someone wakes up and reboots.

A subtle point: the health check should actually verify the server can do real work, not just that the process is running. A common trap is a `/health` endpoint that returns 200 OK even when the server can't reach its database — so the load balancer keeps sending traffic to a server that fails every request. Good health checks exercise the server's actual dependencies.

## The load balancer itself: not a single point of failure

You might notice a hole in this story: if all traffic flows through the load balancer, isn't *it* a single point of failure? Yes — and this is taken seriously. Production load balancers are themselves redundant: typically two or more instances in active-passive or active-active configuration, sharing a virtual IP that can fail over in seconds. Cloud-managed load balancers (AWS ELB, GCP Load Balancer) handle this transparently — they're internally distributed across many machines and availability zones, so the "load balancer" you provision is itself a horizontally-scaled system.

## Sticky sessions: the escape hatch

Sometimes you can't make your servers fully stateless — maybe you have a legacy app that stores session data locally and rewriting it would take months. The escape hatch is **sticky sessions** (a.k.a. session affinity): the load balancer remembers which server first handled a given user (via a cookie or IP hash) and keeps sending them there.

This works, but it sacrifices some of horizontal scaling's benefits. Load distribution becomes uneven (heavy users get pinned to whichever server they hit first). If a server dies, all its pinned users lose their sessions. Auto-scaling becomes awkward because you can't freely shift load.

Treat sticky sessions as a workaround, not a strategy. The right long-term move is almost always to externalize state (Redis for sessions, for example) and go fully stateless.

## How they connect

Here's the loop these two concepts form: horizontal scaling lets you handle more load, but only if requests are distributed across the fleet — which requires a load balancer. The load balancer enables the fleet to be elastic (servers come and go) and fault-tolerant (failures are routed around). Statelessness is what makes the load balancer's job easy: any server can handle any request, so distribution is trivial. Lose any one of these — stateless servers, a smart load balancer, or a homogeneous fleet — and the others get much harder.

When Malan walks through this in the Harvard lecture, he builds it up incrementally: one server, then two with a DNS round-robin (a primitive load balancer), then a real load balancer, then he hits the session-stickiness problem and discusses solutions. Watching with this framing should make every step feel motivated rather than arbitrary.

---

Want to test your understanding with a small design exercise? I can give you a scenario ("design the architecture for a site that suddenly went viral on Reddit") and we can work through what you'd add at each step. Or we can move to another pair of topics tomorrow — your call.
