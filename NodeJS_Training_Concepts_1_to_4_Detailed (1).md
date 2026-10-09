# Node.js Backend Engineering — Detailed Concepts 1–4 (Enterprise Edition)

> Team training handbook • Node.js + TypeScript • Nx Monorepo • Microservices

**Audience:** Junior to senior backend developers  
**Format:** Concept explanation → implementation considerations → interview question → practical exercise  
**Assumption:** Express-style services are used for examples; adapt to NestJS or Lambda handlers as needed.

## Learning outcomes
- Explain Node.js internals and concurrency without claiming all work happens on one thread.
- Implement layered TypeScript APIs with secure validation and testable services.
- Understand Nx project boundaries, dependency graphs, and affected CI pipelines.
- Design reliable microservice communication and failure handling.

## Reference architecture
```text
Client / Next.js
      |
API Gateway / BFF
      |
  +---+---------------+
  |                   |
Order API          User API     (independently deployed Node.js services)
  |                   |
Order DB            User DB
  |
  +--> Event bus / Queue --> Notification Worker

Nx monorepo: apps/* + libs/* + shared tooling, NOT a runtime message broker
```

---

## 1. Node.js Fundamentals

**Concepts covered:** 20

### 1. Node.js runtime and V8 engine

**Explanation:** Node.js executes JavaScript outside the browser. V8 compiles and executes JavaScript; Node provides APIs such as `fs`, `http`, `crypto`, and process management.

**Interview question:** Distinguish V8, Node core APIs, and libuv; identify which part performs network I/O.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 2. Single-threaded JavaScript execution

**Explanation:** JavaScript callbacks normally execute on one main event-loop thread per Node process. This does not mean Node has only one OS thread: libuv workers and optional worker threads also exist.

**Interview question:** Explain why a CPU-heavy synchronous loop delays unrelated HTTP responses.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 3. Event loop and its phases

**Explanation:** The event loop coordinates callbacks for timers, pending callbacks, poll, check, and close phases. Exact ordering depends on context and Node/libuv version; do not memorize a universal timer-versus-immediate order.

**Interview question:** Trace a network callback scheduling `setImmediate` and `setTimeout(0)`.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 4. Call stack and callback queue

**Explanation:** The call stack tracks executing functions. Once JavaScript finishes the current stack, queued work can run according to the event-loop and microtask rules.

**Interview question:** Draw the call stack during a nested function call and queued timer.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 5. Microtasks and macrotasks

**Explanation:** Promise reactions and `queueMicrotask` run in microtask processing; `process.nextTick` uses a separate higher-priority Node queue in CommonJS contexts. Timers and immediates run in event-loop phases. Excessive microtasks can starve I/O.

**Interview question:** Predict callback order and discuss differences in ESM top-level execution.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 6. Synchronous vs asynchronous programming

**Explanation:** Synchronous code finishes before subsequent statements; asynchronous APIs allow work to complete later. `async` functions return promises but CPU work inside them still runs synchronously until yielding.

**Interview question:** Explain why adding `async` to a CPU-bound function does not make it non-blocking.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 7. Blocking vs non-blocking I/O

**Explanation:** Blocking calls hold the main JavaScript thread while waiting. Non-blocking operations let Node handle other work while I/O is pending. Async filesystem calls can use libuv’s worker pool; sockets commonly use OS readiness notifications.

**Interview question:** Compare `readFileSync` and `fs.promises.readFile` under concurrent load.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 8. Callbacks, Promises, and async/await

**Explanation:** Callbacks represent continuation functions; promises represent eventual results; `await` makes promise composition readable. Use `Promise.all` for independent work and `Promise.allSettled` when partial results matter.

**Interview question:** Implement parallel calls and explain failure behavior of `Promise.all`.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 9. Error handling and exception management

**Explanation:** Handle rejected promises, expected domain errors, validation failures, and unexpected faults differently. Preserve stack/context, log safely, and map domain errors to stable HTTP responses.

**Interview question:** Explain why `try/catch` around a call does not catch a later unhandled callback throw.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 10. CommonJS vs ES Modules

**Explanation:** CommonJS uses `require`/`module.exports`; ESM uses `import`/`export`. Module resolution and interoperability depend on package `type`, file extensions, and runtime configuration.

**Interview question:** Describe `package.json` `type: module` and why `__dirname` differs in ESM.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 11. Node.js module system

**Explanation:** Modules encapsulate code; imports are cached according to loader semantics. Core, external, and local modules resolve differently. Avoid hidden mutable singleton state across tests.

**Interview question:** Explain module caching and circular dependency hazards.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 12. npm, pnpm, and package management

**Explanation:** Package managers resolve dependencies and create lockfiles. pnpm commonly uses a content-addressed store and linked package layout; workspace configuration helps Nx share package management.

**Interview question:** Explain `dependencies` vs `devDependencies`, lockfile reproducibility, and `pnpm install --frozen-lockfile`.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 13. Environment variables and configuration

**Explanation:** Read environment variables at startup, validate required values, and fail fast on invalid configuration. Never commit secrets or log tokens. Configuration must be explicit across local, CI, and production environments.

**Interview question:** Design a typed config loader with required `DATABASE_URL`.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 14. Buffers and streams

**Explanation:** Buffers represent binary data. Readable, writable, duplex, and transform streams process data in chunks; backpressure prevents fast producers overwhelming slow consumers. Prefer `pipeline` for safe stream composition.

**Interview question:** Explain why streaming a multi-GB file is safer than buffering it fully.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 15. File system operations

**Explanation:** `fs/promises` supports asynchronous file operations. Handle missing files, permissions, atomic-write concerns, and path traversal when using user-supplied paths.

**Interview question:** Implement a safe file reader constrained to an uploads directory.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 16. EventEmitter

**Explanation:** `EventEmitter` provides in-process synchronous event dispatch to listeners. It is not a durable distributed message bus. Listeners should handle errors and avoid blocking work.

**Interview question:** Contrast EventEmitter with SQS or EventBridge.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 17. Worker threads and child processes

**Explanation:** Worker threads run JavaScript on additional threads and suit CPU-bound computations; child processes run separate OS processes and offer stronger isolation. Neither replaces async I/O for typical database/network calls.

**Interview question:** Choose a mechanism for image processing versus calling a CLI tool.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 18. Memory management and garbage collection

**Explanation:** V8 garbage-collects reachable JavaScript objects; memory leaks still arise from retained references, unbounded caches, timers, and listeners. Inspect heap snapshots and monitor RSS and heap usage.

**Interview question:** Diagnose memory growth in a service with an unbounded Map cache.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 19. Process lifecycle and graceful shutdown

**Explanation:** Handle SIGTERM by stopping new requests, allowing in-flight work to finish within a deadline, and closing database/queue connections. In serverless environments, execution lifecycle differs from long-running processes.

**Interview question:** Design shutdown behavior during a rolling deployment.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 20. libuv and the Node.js thread pool

**Explanation:** libuv supports event-loop I/O and a worker pool for certain operations such as filesystem, DNS lookup, and some crypto tasks. Thread-pool saturation can delay unrelated work; changing pool size is not a universal fix.

**Interview question:** Explain how many expensive `pbkdf2` calls affect filesystem latency.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

#### Lab: Event loop and async behavior
```ts
import { readFile } from "node:fs/promises";
console.log("A");
setTimeout(() => console.log("timer"), 0);
Promise.resolve().then(() => console.log("promise"));
process.nextTick(() => console.log("nextTick"));
console.log("B");
// In a typical CommonJS execution: A, B, nextTick, promise, timer
// Ordering can differ in ESM top-level context.
const data = await readFile("./package.json", "utf8");
console.log(data.length);
```
**Exercise:** Replace a synchronous filesystem read with `readFile`; benchmark concurrent requests and explain the result.

#### Review checklist
- [ ] Each developer can explain the concept without reading a definition.
- [ ] Each developer can point to a real implementation or create a small one.
- [ ] Each developer can describe one failure mode and mitigation.
- [ ] Team lead reviews naming, error handling, tests, and dependency boundaries.

---

## 2. Backend Development and TypeScript

**Concepts covered:** 16

### 1. Express.js architecture

**Explanation:** Express composes middleware and route handlers. Keep HTTP concerns in controllers and business rules in services; choose consistent dependency wiring and error handling.

**Interview question:** Trace `app.use`, router, controller, service, repository.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 2. RESTful API development

**Explanation:** Model resources with stable URLs and HTTP methods. Use meaningful status codes, idempotent semantics where applicable, and clear request/response contracts.

**Interview question:** Design CRUD routes for orders and explain POST vs PUT vs PATCH.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 3. Routing and middleware

**Explanation:** Middleware handles cross-cutting concerns such as authentication, validation, request IDs, and logging. Ordering matters; error middleware has a special signature.

**Interview question:** Show why validation should run before a controller.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 4. Controllers, services, and repositories

**Explanation:** Controller maps HTTP input/output; service enforces business logic; repository encapsulates persistence. Avoid leaking Express `Request` objects into domain services.

**Interview question:** Identify misplaced database queries in a controller.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 5. Dependency injection

**Explanation:** Pass dependencies through constructors/factories rather than importing hard-coded globals. This improves testability and makes infrastructure replacements easier.

**Interview question:** Mock an OrderRepository in a unit test.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 6. Request validation and sanitization

**Explanation:** TypeScript types disappear at runtime; validate untrusted input using a schema library and reject unexpected data. Parameterized queries prevent SQL injection; escaping depends on output context.

**Interview question:** Demonstrate why `req.body as Order` is unsafe.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 7. Centralized error handling

**Explanation:** Use a consistent error envelope with code, message, and request ID. Separate 4xx domain/input errors from 5xx internal errors; do not expose stack traces to clients.

**Interview question:** Implement error mapping for duplicate orders and database failures.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 8. Authentication and authorization

**Explanation:** Authentication verifies identity; authorization determines allowed actions. Enforce access checks server-side at resource and operation level.

**Interview question:** Prevent a user from retrieving another user’s order by ID.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 9. JWT, OAuth 2.0, and OIDC

**Explanation:** JWT is a token format; OAuth 2.0 is an authorization framework; OIDC adds authentication identity semantics. Validate issuer, audience, signature, expiry, and scopes as applicable.

**Interview question:** Explain access token versus ID token and why an ID token is not an API authorization token.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 10. Role-based access control (RBAC)

**Explanation:** Roles aggregate permissions, but resource ownership and tenant boundaries still need checks. Default deny and keep authorization policies auditable.

**Interview question:** Design admin/support/customer access to an order endpoint.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 11. TypeScript interfaces, types, and generics

**Explanation:** Interfaces describe object shapes; type aliases express unions and mapped types; generics preserve relationships between input/output types. Prefer `unknown` to unsafe `any`.

**Interview question:** Create `ApiResponse<T>` and a discriminated union for errors.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 12. API versioning

**Explanation:** Use additive changes where possible; version breaking contracts deliberately, communicate deprecation, and maintain compatibility tests. URL, header, and media-type versioning each have trade-offs.

**Interview question:** Plan migration from `/v1/orders` to a changed v2 schema.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 13. Pagination, filtering, and sorting

**Explanation:** Offset pagination is simple but may be slow/unstable at scale; cursor pagination improves traversal for ordered data. Whitelist sort fields and cap page sizes.

**Interview question:** Design a cursor-based order history endpoint.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 14. Rate limiting and throttling

**Explanation:** Limit abusive or accidental traffic at gateway and application layers. Distributed deployments need shared counters or gateway controls; return useful 429 behavior.

**Interview question:** Choose limits for login versus read-only listing endpoints.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 15. API documentation with OpenAPI/Swagger

**Explanation:** OpenAPI describes routes, schemas, authentication, errors, and examples. Keep documentation generated or verified against code to avoid drift.

**Interview question:** Write an OpenAPI spec for POST `/orders`.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 16. Unit, integration, and end-to-end testing

**Explanation:** Unit tests isolate business rules; integration tests verify real boundaries such as DB/HTTP; E2E tests exercise complete workflows. Test failures, authorization, and concurrency—not just happy paths.

**Interview question:** Propose a test pyramid for an order microservice.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

#### Lab: Layered Order API (Express + TypeScript)
```ts
import express from "express";
import { z } from "zod";
const app = express();
app.use(express.json());
const CreateOrder = z.object({ productId: z.string().min(1), quantity: z.number().int().positive() });
type OrderInput = z.infer<typeof CreateOrder>;
interface OrderRepository { save(input: OrderInput): Promise<{ id: string }>; }
class OrderService {
  constructor(private repo: OrderRepository) {}
  async create(input: OrderInput) { return this.repo.save(input); }
}
const repo: OrderRepository = { async save(_input) { return { id: crypto.randomUUID() }; } };
const service = new OrderService(repo);
app.post("/orders", async (req, res, next) => {
  try {
    const parsed = CreateOrder.safeParse(req.body);
    if (!parsed.success) return res.status(400).json({ code: "VALIDATION_ERROR", issues: parsed.error.issues });
    return res.status(201).json(await service.create(parsed.data));
  } catch (error) { next(error); }
});
// Add centralized error middleware, auth, and real persistence in production.
```
**Exercise:** Add GET `/orders/:id`, repository tests, authorization checks, and a stable error contract.

#### Review checklist
- [ ] Each developer can explain the concept without reading a definition.
- [ ] Each developer can point to a real implementation or create a small one.
- [ ] Each developer can describe one failure mode and mitigation.
- [ ] Team lead reviews naming, error handling, tests, and dependency boundaries.

---

## 3. Nx Monorepo Architecture

**Concepts covered:** 12

### 1. Monorepo vs polyrepo

**Explanation:** A monorepo stores multiple projects in one repository, simplifying atomic changes and shared tooling but requiring strong boundaries and CI optimization. Polyrepos offer independent repositories at coordination cost.

**Interview question:** Explain why monorepo does not mean monolith.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 2. Nx workspace structure

**Explanation:** Nx discovers projects through configuration and plugins; layouts often use `apps/` and `libs/`, but are conventions rather than requirements. Inspect actual project configuration before teaching a folder standard.

**Interview question:** Locate project names, targets, and root paths in your workspace.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 3. Apps vs shared libraries

**Explanation:** Applications are deployable/runnable entry points; libraries package reusable code. Shared code should remain narrowly scoped to avoid coupling services to a common domain model.

**Interview question:** Decide whether authentication contracts belong in a shared lib.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 4. Nx project graph

**Explanation:** Nx builds a dependency graph from imports/configuration. The graph informs affected tasks and boundary checks; it is not a runtime network topology.

**Interview question:** Run `npx nx graph` and identify upstream dependencies.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 5. Dependency boundaries

**Explanation:** Enforce module boundaries with tags and lint rules. Domain libraries should not import app internals; avoid circular dependencies and unrestricted cross-service imports.

**Interview question:** Propose `scope:orders`, `scope:shared`, `type:api`, `type:domain` tags.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 6. Nx affected commands

**Explanation:** Affected task selection compares a base and head revision and uses project dependencies. Correct git base selection matters in CI; not every change can be safely skipped.

**Interview question:** Run `npx nx affected -t test --base=origin/main --head=HEAD`.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 7. Task caching

**Explanation:** Nx hashes task inputs and reuses matching outputs. Correct cache configuration must include all relevant environment/config inputs; never cache tasks with unsafe side effects.

**Interview question:** Explain why a changed environment variable might require cache input configuration.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 8. Parallel builds and tests

**Explanation:** Nx schedules independent tasks concurrently according to dependency constraints and worker capacity. Too much parallelism can cause memory contention in CI.

**Interview question:** Tune `--parallel` for a constrained runner.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 9. Build executors

**Explanation:** Targets map commands or executors to build, serve, lint, and test operations. Node bundling and output paths depend on selected executor and workspace version.

**Interview question:** Inspect `project.json` or inferred targets for an API service.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 10. TypeScript path aliases

**Explanation:** Aliases improve imports but do not automatically enforce architecture. Runtime bundling/module resolution must agree with TypeScript configuration.

**Interview question:** Explain why a TS alias may compile but fail at runtime.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 11. Shared contracts and utilities

**Explanation:** Share DTOs, schemas, and cross-cutting utilities carefully; avoid importing another service’s repository or business implementation. Version contracts and preserve backward compatibility.

**Interview question:** Design a small `@org/contracts` library.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 12. CI/CD integration with Nx

**Explanation:** Use affected lint/test/build, cache, artifact publishing, and deployment per service. Add integration tests for cross-service contract changes and keep deployments independently reversible.

**Interview question:** Sketch a GitHub Actions pipeline for changed services.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

#### Lab: Nx inspection commands
```bash
npx nx show projects
npx nx graph
npx nx show project order-service
npx nx affected -t lint test build --base=origin/main --head=HEAD
npx nx run order-service:build
```
**Exercise:** Change one shared library, inspect affected projects, and document whether the dependency direction is appropriate.

#### Review checklist
- [ ] Each developer can explain the concept without reading a definition.
- [ ] Each developer can point to a real implementation or create a small one.
- [ ] Each developer can describe one failure mode and mitigation.
- [ ] Team lead reviews naming, error handling, tests, and dependency boundaries.

---

## 4. Microservices and Distributed Systems

**Concepts covered:** 20

### 1. Monolithic vs microservices architecture

**Explanation:** A monolith deploys as one unit; microservices split deployable capabilities. Microservices add network latency, operational complexity, and data consistency challenges; use them when autonomy justifies the cost.

**Interview question:** Explain how a modular monolith differs from a microservice system.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 2. Domain-driven design (DDD)

**Explanation:** Bounded contexts define business-language and ownership boundaries. Entities, value objects, aggregates, and domain events help structure complex domains; not every CRUD service needs full DDD ceremony.

**Interview question:** Identify bounded contexts in an ordering platform.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 3. Backend-for-Frontend (BFF)

**Explanation:** A BFF tailors APIs to a specific client experience, aggregating backend calls and adapting contracts without taking over core domain ownership.

**Interview question:** Compare web and mobile BFF requirements.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 4. API Gateway pattern

**Explanation:** A gateway routes and protects requests, often handling auth, rate limits, and observability. Avoid embedding complex domain logic into the gateway.

**Interview question:** Trace a request through gateway to order service.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 5. Synchronous vs asynchronous communication

**Explanation:** Synchronous HTTP/gRPC gives immediate response but couples availability; async messaging decouples processing and needs eventual-result tracking and duplicate handling.

**Interview question:** Choose between REST and queue for payment confirmation.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 6. REST vs event-driven communication

**Explanation:** REST models direct commands/queries; events announce facts that occurred. Events should be versioned, immutable in meaning, and consumed without assuming one consumer.

**Interview question:** Distinguish `CreateOrder` command from `OrderCreated` event.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 7. Event-driven architecture

**Explanation:** Producers publish events; consumers react independently. Design for at-least-once delivery, ordering limits, schema evolution, and replay.

**Interview question:** Map `OrderPlaced` to inventory and notification consumers.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 8. Publish-subscribe pattern

**Explanation:** Pub/sub fans out messages to multiple subscribers. Broker guarantees vary; a subscription is not equivalent to durable processing unless configured accordingly.

**Interview question:** Compare SNS/EventBridge fan-out to one SQS queue.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 9. Message queues

**Explanation:** Queues buffer work and smooth load. Consumers acknowledge successful processing; visibility timeout, retries, DLQ, and idempotency are core design choices.

**Interview question:** Explain what happens if a worker crashes after DB commit but before ack.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 10. Service discovery

**Explanation:** Services need reliable endpoint resolution via DNS, registry, or platform routing. Kubernetes and managed cloud services provide different mechanisms.

**Interview question:** Describe discovery when service instances scale dynamically.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 11. Database per service

**Explanation:** Each service owns its data schema and access rules; cross-service joins become API/events/read models. Separate physical databases are not always required, but ownership boundaries must be real.

**Interview question:** Explain why direct cross-service table writes are dangerous.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 12. Distributed transactions and Saga pattern

**Explanation:** A saga coordinates local transactions using events or orchestration and compensating actions. Compensation is business reversal, not a guaranteed database rollback.

**Interview question:** Design order→payment→inventory with a payment failure path.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 13. Idempotency

**Explanation:** Repeated requests/events should not repeat side effects. Use idempotency keys, unique constraints, deduplication records, and transactional boundaries.

**Interview question:** Prevent double charges from duplicate payment webhooks.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 14. Eventual consistency

**Explanation:** Replicated or event-updated views can lag the source of truth. Expose pending states and design reconciliation rather than promising immediate consistency across services.

**Interview question:** Explain why a newly placed order is absent from a search index briefly.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 15. Circuit breaker pattern

**Explanation:** A circuit breaker stops repeatedly calling a failing dependency and probes recovery later. Combine with timeouts, bulkheads, and sensible fallback behavior.

**Interview question:** Explain open, closed, and half-open states.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 16. Retry strategies and exponential backoff

**Explanation:** Retry transient failures with bounded attempts, jitter, and clear timeout budgets. Do not retry non-idempotent operations blindly or create retry storms.

**Interview question:** Design retry behavior for HTTP 429 versus 400.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 17. Dead-letter queues (DLQ)

**Explanation:** DLQs hold messages that repeatedly fail processing. Configure redrive policies, alert on depth, investigate root cause, and replay safely after fixes.

**Interview question:** Explain why a DLQ is not a substitute for idempotency.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 18. Distributed tracing

**Explanation:** Propagate trace context across HTTP and messaging; spans show latency and failure across services. Protect sensitive data in span attributes.

**Interview question:** Follow one checkout request across gateway, orders, and payments.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 19. Correlation IDs

**Explanation:** A correlation ID connects related logs and messages even without full tracing. Generate or validate at trust boundaries and propagate consistently.

**Interview question:** Find all logs associated with one order ID and request ID.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

### 20. Fault tolerance and resilience

**Explanation:** Set timeouts, concurrency limits, backpressure, health probes, and fallback strategies. Test partial outages and overload, not only normal traffic.

**Interview question:** Design behavior when the payment provider is down for 10 minutes.

**Practice:** Find or implement one concrete example of this concept in your current Nx workspace; explain trade-offs and failure modes.

#### Lab: Reliable order-processing design
```text
POST /orders -> Order Service -> Order DB
                    | (transactional outbox)
                    v
              OrderCreated event
                    v
              Queue / Broker
                 |       |
          Inventory     Notification
          Consumer       Consumer
```
**Failure scenarios to explain:** duplicate event; consumer crash after DB commit; unavailable inventory; message schema change; poison message; tracing across asynchronous boundaries.

#### Review checklist
- [ ] Each developer can explain the concept without reading a definition.
- [ ] Each developer can point to a real implementation or create a small one.
- [ ] Each developer can describe one failure mode and mitigation.
- [ ] Team lead reviews naming, error handling, tests, and dependency boundaries.

---
## Final assessment: 20 interview questions with answer guidance

1. **Why is Node.js good for I/O-heavy workloads?**  
   - Expected: Non-blocking I/O plus event-loop scheduling; not because CPU work is automatically parallel.

2. **Can Node.js use multiple threads?**  
   - Expected: Yes: libuv pool, worker threads, and multiple processes; main JS callback execution is usually single-threaded per isolate.

3. **What blocks the event loop?**  
   - Expected: CPU-heavy JS and synchronous operations.

4. **Promise.all vs Promise.allSettled?**  
   - Expected: Fail-fast aggregate rejection vs waiting for all outcomes.

5. **What is backpressure?**  
   - Expected: Consumer signals producer to slow down; prevents memory overload.

6. **Why validate TypeScript API input at runtime?**  
   - Expected: Types are erased; external payloads are untrusted.

7. **Controller vs service?**  
   - Expected: Transport handling vs domain rules.

8. **What is dependency injection?**  
   - Expected: Supply dependencies from outside for decoupling and testing.

9. **JWT vs OAuth vs OIDC?**  
   - Expected: Token format vs authorization framework vs authentication identity layer.

10. **How do you secure object-level authorization?**  
   - Expected: Check authenticated subject permissions/ownership on each resource.

11. **What is Nx project graph?**  
   - Expected: Static workspace dependency graph for tooling, not service communication.

12. **What does nx affected do?**  
   - Expected: Select tasks for changed projects and impacted dependents using a git diff.

13. **How do you prevent cross-service coupling in Nx?**  
   - Expected: Tagged library boundaries, contract sharing, independent persistence ownership.

14. **Does monorepo mean monolith?**  
   - Expected: No: repository layout differs from runtime/deployment architecture.

15. **When choose async messaging over REST?**  
   - Expected: For decoupled background work, buffering, fan-out, and eventual processing.

16. **What does at-least-once delivery imply?**  
   - Expected: Duplicates are possible; consumers must be idempotent.

17. **What is a saga?**  
   - Expected: Sequence of local transactions with compensation/orchestration.

18. **What is eventual consistency?**  
   - Expected: Replicas/read models converge after a delay; design UX/reconciliation accordingly.

19. **How do retries become dangerous?**  
   - Expected: Amplified traffic, duplicate side effects, retry storms; use jitter and limits.

20. **What belongs in a DLQ runbook?**  
   - Expected: Alerting, diagnosis, redrive rules, idempotent replay, and ownership.

## Capstone assignment
Build an Nx monorepo with `api-gateway`, `order-service`, and `notification-worker`, plus `contracts` and `shared-logging` libraries. Use a database of your choice, implement validated order creation, an event for order creation, idempotent event handling, and unit/integration tests. Enforce dependency boundaries and use affected CI tasks.

**Acceptance criteria:** correct 2xx/4xx/5xx behavior, documented OpenAPI contract, no secrets committed, request correlation, graceful shutdown for long-running services, duplicate-event safety, passing tests, and a diagram of data ownership.

## Suggested training cadence
| Session | Focus | Time | Deliverable |
|---|---|---|---|
| 1 | Runtime, event loop, async I/O | 90 min | Event-loop lab |
| 2 | Streams, workers, errors, memory | 90 min | Profiling notes |
| 3 | Express/TypeScript layers | 90 min | Order API |
| 4 | Auth, validation, testing | 90 min | Tested endpoints |
| 5 | Nx projects, libraries, affected CI | 90 min | Project graph review |
| 6 | Service boundaries and messaging | 90 min | Event workflow diagram |
| 7 | Reliability, sagas, observability | 90 min | Failure simulation |
| 8 | Capstone review + mock interviews | 120 min | Demo + scorecard |

> **Note:** Examples are educational starting points. Confirm framework/runtime versions and deployment requirements before applying them to production.

---

# Enterprise-level additions (2026 curriculum)

These additions extend—not replace—the original 68 concepts. They focus on current production engineering practices for TypeScript, Node.js, Nx and distributed systems. **Architecture choices depend on scale and constraints; examples are teaching patterns, not universal rules.**


## 1. Node.js Runtime — Advanced Production Concepts


### E01. AsyncLocalStorage and request context

**Enterprise explanation:** Use node:async_hooks AsyncLocalStorage to propagate correlation IDs and tenant context through asynchronous call chains. Avoid storing request context in module globals.

**Interview question:** What are the limitations of AsyncLocalStorage across worker threads or external messages?

**Hands-on exercise:** Add correlationId middleware and verify concurrent requests never mix context.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E02. AbortController and cancellation

**Enterprise explanation:** Propagate AbortSignal through outbound fetch, timers, and application work; enforce deadlines and release resources on client disconnect where safe.

**Interview question:** How do you prevent orphaned requests and wasted work when a caller times out?

**Hands-on exercise:** Cancel a slow downstream HTTP call and record a cancellation metric.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E03. HTTP connection pooling and keep-alive

**Enterprise explanation:** Understand undici/fetch connection reuse, DNS, TLS handshake cost, idle sockets, max connections, and request timeout budgets.

**Interview question:** Why can a service have high latency even when its downstream CPU is low?

**Hands-on exercise:** Benchmark repeated requests with and without a reusable client.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E04. Event-loop lag and utilization

**Enterprise explanation:** Monitor event-loop delay, utilization, CPU, and heap independently; high lag can arise from synchronous JSON parsing, regex, or crypto.

**Interview question:** How would you distinguish event-loop blocking from database latency?

**Hands-on exercise:** Introduce a blocking task, measure event-loop delay, then offload it.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E05. Heap snapshots and memory leak investigation

**Enterprise explanation:** Use process.memoryUsage, heap snapshots, allocation profiling, and retained-object analysis. Track RSS versus heapUsed and external Buffer memory.

**Interview question:** Why can RSS grow while V8 heap usage stays relatively stable?

**Hands-on exercise:** Create a controlled leak in a lab and document its retained references.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E06. Stream backpressure and pipeline

**Enterprise explanation:** Prefer stream.pipeline or promises.pipeline for large files; respect writable backpressure and handle cancellation/errors.

**Interview question:** Why is reading a 5 GB file with readFile unsafe for a memory-constrained service?

**Hands-on exercise:** Stream a large CSV through a transform with bounded memory.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E07. Worker threads vs child processes vs Lambda

**Enterprise explanation:** Worker threads help CPU-bound JavaScript; child processes isolate execution; Lambda isolates deployed workloads. None makes a blocking main thread automatically non-blocking.

**Interview question:** When would you choose a worker pool over adding another microservice?

**Hands-on exercise:** Compare CPU-heavy hashing on main thread and a bounded worker pool.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E08. Runtime upgrades and compatibility

**Enterprise explanation:** Adopt supported Node.js LTS releases based on your organization runtime policy; test ESM/CJS interop, native addons, OpenSSL, and dependency compatibility.

**Interview question:** What checks belong in a major Node.js upgrade plan?

**Hands-on exercise:** Create a compatibility matrix and CI test against current and next approved runtimes.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


## 2. Backend Engineering — Enterprise API Concepts


### E09. Contract-first APIs and schema evolution

**Enterprise explanation:** Design OpenAPI contracts before implementation, generate typed clients when useful, and check backward compatibility in CI.

**Interview question:** How can you add a required response field without breaking consumers?

**Hands-on exercise:** Add an OpenAPI contract test for an Order API.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E10. Runtime validation versus TypeScript types

**Enterprise explanation:** TypeScript types disappear at runtime; validate untrusted HTTP bodies, event payloads, headers, and configuration with a schema library.

**Interview question:** Why is `req.body as Order` not validation?

**Hands-on exercise:** Reject unknown and invalid payloads with consistent problem details.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E11. Idempotent HTTP APIs

**Enterprise explanation:** Use an idempotency key and durable deduplication for retries of non-idempotent operations such as payment creation. Define retention and concurrent request behavior.

**Interview question:** What happens if two identical POST requests arrive simultaneously?

**Hands-on exercise:** Implement a unique-key guarded order creation endpoint.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E12. Timeouts, retries, and retry budgets

**Enterprise explanation:** Set explicit connect, request, and overall deadlines; retry only safe/transient failures with jitter, backoff, and a bounded budget.

**Interview question:** Why can retries cause a cascading outage?

**Hands-on exercise:** Simulate a flaky dependency and measure amplified traffic.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E13. OAuth2/OIDC security boundaries

**Enterprise explanation:** Differentiate identity (OIDC) from delegated authorization (OAuth2); validate issuer, audience, expiry, signature, and authorized scopes on each resource server.

**Interview question:** Why is decoding a JWT without verifying its signature insecure?

**Hands-on exercise:** Add JWKS-based JWT verification with invalid issuer/audience tests.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E14. Multi-tenancy and data isolation

**Enterprise explanation:** Resolve tenant identity from trusted claims or routing; enforce tenant scope at service and data access layers; test cross-tenant denial.

**Interview question:** How could a missing WHERE tenant_id filter expose another customer’s data?

**Hands-on exercise:** Write an integration test proving tenant A cannot read tenant B orders.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E15. Transactional outbox pattern

**Enterprise explanation:** Persist domain state and an outbox event in one database transaction; asynchronously publish with at-least-once semantics and deduplicate consumers.

**Interview question:** How do you avoid updating an order without publishing its event?

**Hands-on exercise:** Build a simple outbox table and relay worker.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E16. API pagination at scale

**Enterprise explanation:** Use cursor/keyset pagination for large mutable datasets; stable ordering, indexed sort keys, opaque cursors, and maximum page sizes.

**Interview question:** Why does OFFSET pagination degrade and skip rows under concurrent writes?

**Hands-on exercise:** Implement cursor pagination on (created_at, id).

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E17. Secure file and webhook handling

**Enterprise explanation:** Verify webhook signatures against the raw request body, check replay windows, and use object storage for uploads with content/type/size checks.

**Interview question:** Why can JSON parsing before HMAC verification break webhook signatures?

**Hands-on exercise:** Add a replay-protected webhook endpoint with tamper tests.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


## 3. Nx Monorepo — Enterprise Workspace Governance


### E18. Project graph and enforceable boundaries

**Enterprise explanation:** Use explicit tags and Nx module-boundary lint rules to prevent circular imports and accidental cross-domain dependencies.

**Interview question:** How do you stop payment code from importing order-service internals?

**Hands-on exercise:** Define domain and layer tags and demonstrate a failing forbidden import.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E19. Buildable versus publishable libraries

**Enterprise explanation:** Distinguish source-only shared code, buildable packages, and independently published libraries; avoid sharing domain internals merely for convenience.

**Interview question:** What are the trade-offs of publishing shared DTOs versus keeping them source-only?

**Hands-on exercise:** Extract a small contracts library and document its consumers.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E20. Nx affected CI and remote caching

**Enterprise explanation:** Run affected lint/test/build on pull requests; configure deterministic task inputs/outputs and carefully protect cache access and secrets.

**Interview question:** What causes false cache hits or non-reproducible builds?

**Hands-on exercise:** Modify one library and verify only its dependents are rebuilt.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E21. Release independence in a monorepo

**Enterprise explanation:** A shared repository does not require a shared deployment; use per-service Docker images, versioned contracts, and isolated rollout/rollback.

**Interview question:** How do you roll back only the payment service after a bad release?

**Hands-on exercise:** Write a deployment manifest with independent service versions.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E22. Dependency and supply-chain security

**Enterprise explanation:** Use lockfile integrity, dependency scanning, provenance, SBOMs, secret scanning, and least-privilege CI credentials.

**Interview question:** Why is a green unit test insufficient for supply-chain assurance?

**Hands-on exercise:** Add a dependency audit and SBOM artifact to CI.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E23. Monorepo developer experience at scale

**Enterprise explanation:** Use generators, consistent tsconfig/eslint, local compose dependencies, seeded fixtures, and ownership rules to reduce drift.

**Interview question:** How would you onboard a new engineer to 20 Nx services?

**Hands-on exercise:** Create a documented service generator checklist.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


## 4. Microservices — Enterprise Distributed Systems


### E24. Service ownership and bounded contexts

**Enterprise explanation:** Split services around business capabilities and data ownership, not simply by CRUD tables; define contracts and team ownership.

**Interview question:** What signs indicate a distributed monolith?

**Hands-on exercise:** Draw service boundaries for checkout, inventory, and payments.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E25. Delivery semantics and consumer deduplication

**Enterprise explanation:** Assume duplicate delivery with many brokers; implement durable idempotent consumers. FIFO ordering is scoped and does not remove the need for deduplication.

**Interview question:** Why is exactly-once business processing difficult?

**Hands-on exercise:** Process the same event twice and verify one business effect.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E26. Ordering, partitioning, and hot keys

**Enterprise explanation:** Choose partition/message-group keys based on ordering requirements and throughput; understand head-of-line blocking and skew.

**Interview question:** Why can one tenant’s events slow everyone else down?

**Hands-on exercise:** Model per-order ordering and evaluate throughput impact.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E27. Saga orchestration versus choreography

**Enterprise explanation:** Coordinate cross-service transactions with compensating actions; explicitly handle timeouts, partial failure, and non-reversible external effects.

**Interview question:** How do you recover when payment succeeds but inventory reservation fails?

**Hands-on exercise:** Diagram and test an order cancellation compensation path.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E28. Circuit breakers and bulkheads

**Enterprise explanation:** Isolate dependency failures using concurrency limits, queues, circuit breakers, and fallback policies; avoid masking critical failures.

**Interview question:** What is the difference between a circuit breaker and a timeout?

**Hands-on exercise:** Introduce a failing payment dependency and show isolation.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E29. Distributed tracing and OpenTelemetry

**Enterprise explanation:** Propagate W3C trace context across HTTP and messages; correlate spans with structured logs and metrics while redacting secrets and PII.

**Interview question:** How do you debug one checkout request across five services?

**Hands-on exercise:** Trace a request through gateway, order, and payment services.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E30. Event schema governance

**Enterprise explanation:** Version events for backward compatibility, document ownership, and run producer-consumer contract tests before rollout.

**Interview question:** How can a new producer break an old consumer?

**Hands-on exercise:** Add a compatibility test for OrderCreated v1 and v2.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E31. Eventual consistency and user experience

**Enterprise explanation:** Define expected staleness, read models, reconciliation, and user-visible pending states instead of pretending all services update atomically.

**Interview question:** What should the UI show when an order is accepted but payment confirmation is delayed?

**Hands-on exercise:** Implement a pending/confirmed status flow.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E32. Resilience testing and failure injection

**Enterprise explanation:** Practice network partitions, latency, duplicate messages, DLQ replays, and dependency outages in non-production environments.

**Interview question:** What should happen if the broker is down for 20 minutes?

**Hands-on exercise:** Run a controlled outage drill with recovery criteria.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E33. Service-level objectives and error budgets

**Enterprise explanation:** Define availability/latency SLOs, measurable SLIs, burn-rate alerts, and runbooks tied to business-critical journeys.

**Interview question:** Why is 99.9% uptime alone an incomplete reliability target?

**Hands-on exercise:** Draft SLIs and SLOs for order placement.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


### E34. Zero-downtime migrations and deployments

**Enterprise explanation:** Use expand-migrate-contract for schemas, backward-compatible contracts, canary/blue-green rollouts, health probes, and automated rollback.

**Interview question:** How can old and new service versions safely coexist during rollout?

**Hands-on exercise:** Write a two-release plan for renaming a database column.

**Review evidence:** Demonstrate behavior with an automated test, measurement, architecture decision record, or reproducible failure scenario.


## Today's 90-minute team workshop — Nx microservice production readiness

**0–15 min:** Have a developer draw `client → gateway → order service → database` and identify the request ID, authentication boundary, timeout budget, and ownership of each component.

**15–35 min:** Open the real Nx project graph. Identify one service, its shared libraries, forbidden imports, and its independent build/deployment artifact.

**35–60 min:** Pair-program an Order API with runtime validation, structured logging, error mapping, and a correlation ID propagated to a downstream call.

**60–80 min:** Simulate a timeout and duplicate event. Discuss retry safety, idempotency, and how to observe the failure.

**80–90 min:** Each developer explains the request flow and answers two interview questions. Record knowledge gaps without ranking individuals publicly.

### Exit criteria

- [ ] Can explain Node.js event-loop blocking versus asynchronous I/O.
- [ ] Can trace an HTTP request through middleware, controller, service, repository and database.
- [ ] Can explain Nx apps, libraries, dependency boundaries and affected builds.
- [ ] Can identify service ownership and why database-per-service matters.
- [ ] Can explain timeout, retry, idempotency and eventual consistency.
- [ ] Can demonstrate at least one integration test and one failure scenario.

## Enterprise capstone — order processing platform

**Reference stack:** Node.js + TypeScript, Nx, Express (or your actual HTTP framework), PostgreSQL, message broker, OpenTelemetry, Docker, GitHub Actions. AWS Lambda/EventBridge/SQS can be used in the later serverless module.

**Services:** API gateway/BFF, order service, inventory service, payment adapter and notification consumer. Keep each service independently deployable with its own owned data; share contracts and cross-cutting utilities, not mutable database models.

**Required behaviors:**

1. `POST /orders` validates input, authorizes tenant access and accepts an idempotency key.
2. Order state and an outbox event are committed atomically.
3. A publisher relays the event; consumers tolerate duplicates and preserve required per-order ordering.
4. Payment failure causes an explicit compensating workflow or recoverable state.
5. Logs and traces include correlation IDs; dashboards show latency, error rate and queue age.
6. Nx affected CI runs tests/builds for changed projects, with service-specific deploy artifacts.
7. Deployments support rollback and backward-compatible database evolution.

**Acceptance tests:** duplicate POST, invalid tenant access, downstream timeout, repeated event delivery, database failure, old/new contract coexistence, graceful shutdown, and a large-response streaming test.

## Senior-level interview scenario bank

1. An endpoint is slow only under concurrency. How do you isolate event-loop lag, database waits and downstream connection saturation?
2. A client retries `POST /payments` after a timeout, but the first call succeeded. How do you prevent double charging?
3. A message arrives twice after a consumer restart. How do you make the side effect idempotent?
4. A shared Nx library change triggers nearly every service build. How do you redesign dependencies and caching?
5. An OrderCreated event adds a mandatory field. How do you roll it out safely across independent consumers?
6. A worker thread crashes while processing a CPU-heavy task. What is the recovery and backpressure strategy?
7. A multi-tenant API accidentally returns another tenant’s records. Which controls and tests prevent recurrence?
8. Your payment dependency is degraded. How do timeout budgets, circuit breakers and queues protect order placement?
9. Two service versions run during a database schema migration. How do you maintain compatibility?
10. An incident shows no useful cross-service logs. What trace propagation, structured logs and SLOs would you introduce?

**Answer rubric:** Expect clear assumptions, a concrete design, failure modes, observability, security implications, testing strategy, and trade-offs. Do not accept only definitions.

## Trainer notes

- Use examples from the team's real Nx repository where permissible; adapt folder names and tooling to the actual workspace.
- Verify exact Nx command syntax and Node.js runtime support against the versions pinned by the repository before running labs.
- Keep Node.js/Nx core training distinct from AWS service-specific modules 5–8.
- Never put credentials, customer data or production secrets into training examples.
