# Functional event driven architecture

## 1. Event-driven architecture
 - is a siiftware architecture paradigm promoting production, detection, consumption and reaction to events
 - events describe something that has happened in the past, event describe fact so it is immutable
 - what problem it solves?
     - having user authentication flow in monolith app, what should we do when last step of authentication that is trivial notification for metrics collector fails? shall we return HTTP 200 or 500?
     - event-driven microservices threat events as source of true, they have clear responisblities and plans for error hanling, they allow to convert described synchronous process into asynchronous, decoupled parts
- if our monolithic application does the job and aligns and aligns with the SLA, introducing an event-driven architecture will be more likely an over-kill, the time and resource needed to build such a system could be better assigned to business features improvement
- EDA should be considered for apps with requirements of hight availability, uptime and fault tolerance - decoupling functionality increases chances of achieving that 
- we can easily reach auditability and observability with EDA
- microservice: 
    - unit of funcitonality that can be deployed and scaled independently, has its own lifecycle
    - enable fault-tolerance = our system might be capable of continuing to server requests in the presence of failures
    - increase maintenance difficulty and roll-out coordination
- not each functionality should be implemented in a new service, we can have multiple paths in a single service that exchange events - it is known as "listen-to-yourself" pattern
- by decoupling functionalities we can extend and scale them independently 
- CQRS:
    - promotes the idea of separating an application between the writing and reading parts, that allows to apply required optimizations on either side
    - required projections can be created between "read" and "write" side of an application
    - good for asynchronous apps, not really for request-response model, or any transactional operations

## 2. Distributed systems
 - have important property: no single point of failure (SPOF)
 - in such sytems we need to identify essential services that guarantee the correct functionality of the system
 - it's much easier to scale stateless services, so we should avoid state as much possible and delegate it into cache or database
 - BASE (basically-available, soft-state, eventual consistency) semantics as contrast to ACID
 - eventual consistency is used to achieve (is tradeoff for) high availability
 - HTTP 409: returns when operation for requested input was already processed (conflict)
 - deduplication:
     - can be handled by message broker itself (e.g Kafka can do that)
     - producer (keep track of sent messages) vs consumer (keep track of consumed messages, requires a state) side  
 - atomicity:
     - having series of operations either all occur or nothing occurs
     - a lot of options to achieve that locally when working with in-memory transactions, it is also available in relational databases
     - distributed transactions:
         - supporte by many SQL databases
         - supported by Kafka 
         - usually implemented via:
           - two-phase commit
           - allowing to revert some operations (so we can keep valid state at the end)
           - change data capture (CDC): 
             - solves the problem: "atomically write to multiple stores"
             - example: we need to store data in PostgreSQL and Redis, we can write it into PSQL first, and than emit PSQL change log to Redis
           - outbox pattern:
             - solves the same problem as CDC, instead of reading log, we write change to PSQL table, and we leave command in another table (outbox) that should be forwarded to second system, both writes happens in single transaction on writer side. Connector then forwards that command to second service
           - distributed lock
             - used to synchronize access to shared resource 
             - efficient and lightweight version can be implemented with Redis (create key with you "lock" key and your "client_id" as a value, set expiration time => you can safely access resource as long as "lock" keeps your "client_id"" in value)

## 3. Stateless vs. stateful
 - a system is described as stateful if it is designed to remember preceding events or user interactions. It also needs to read an initial state from an external storage to start operating. 
 - Pulsar:
     - Kafka alternative
     - has reliable deduplication mechanism
     - allows for topic compaction (keeps only latest value for given key)
     - support for transactions (consume, process and produce messages in one atomic operation)
     - simpler to use and maintain than Kafka (quite important if you have no resources for Kafka maintenance)
 - schema evolution:
     - schema compatibility: new schema can be processed by old instances of services
     - schema versioning: like with API endpoints versioning, topics with different schema versions can be maintained

## 4. Functional programming in Scala 3
## 5. Effectful streams
 - having the stream of data that needs to be processed both in real time and batches we should split these processing modes as batch can block/slow-down the real-time = they should be splited into two independent subflows that are processed separately
 - batching should be used when we for example need to write stream elements into database so that we can reduce amount of connection to database
 
