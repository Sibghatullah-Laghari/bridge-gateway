# Bridge Gateway — Project Development Guide

## 1. Project Overview

**Project:** Bridge Gateway  
**Type:** Lightweight API Gateway  
**Team:** 2 members

### Main idea

Bridge Gateway will sit between clients and backend services.

```text
Client
   |
   v
+-------------------+
|   Bridge Gateway  |
|-------------------|
| Authentication    |
| Authorization     |
| Routing           |
| Rate Limiting     |
| Logging           |
| Request Tracing   |
| Error Handling    |
+-------------------+
   |
   +---------> User Service
   |
   +---------> Order Service
   |
   +---------> Product Service
```

Instead of allowing clients to communicate directly with every backend service, the gateway becomes a controlled entry point.

### Main problems we want to solve

1. **Security** — requests should be authenticated and authorized before reaching protected services.
2. **Traffic control** — clients should not be able to send unlimited requests.
3. **Centralized routing** — clients use one gateway instead of knowing every backend service address.
4. **Monitoring/debugging** — requests should have useful logs and request IDs.
5. **Reliability** — backend failures should be handled safely instead of exposing internal errors.

---

# 2. Technology Stack

The current project uses:

- Java 25
- Spring Boot 4.1.1
- Maven
- Spring MVC / embedded Tomcat
- Git + GitHub
- Postman for API testing

Later, depending on the implementation, we will add:

- Spring Security for authentication/authorization
- JWT for authentication
- Redis for shared rate-limit state
- Docker for running the gateway and supporting services
- Spring Boot Actuator for health/monitoring

We will add dependencies only when a stage actually needs them.

---

# 3. Current Project Status

The repository contains the Spring Boot project inside:

```text
bridge-gateway/
└── gateway/
```

Important files:

```text
gateway/
├── pom.xml
├── mvnw
├── mvnw.cmd
└── src/
    ├── main/
    │   ├── java/
    │   │   └── bridge/gateway/
    │   │       ├── GatewayApplication.java
    │   │       └── ServletInitializer.java
    │   └── resources/
    │       └── application.properties
    └── test/
        └── java/
            └── bridge/gateway/
                └── GatewayApplicationTests.java
```

The project is currently configured for Java 25.

The Maven Wrapper is available, so the project can use:

```powershell
.\mvnw.cmd
```

without requiring a globally installed Maven command.

---

# 4. Development Rules

We will follow these rules throughout development:

### Rule 1 — Build one stage at a time

We will not write the entire gateway at once.

Each stage should:

1. Have a clear purpose.
2. Be implemented.
3. Be tested.
4. Be reviewed for security.
5. Be committed to Git.
6. Then we move to the next stage.

### Rule 2 — Keep the implementation simple

We will avoid unnecessary:

- classes
- abstractions
- dependencies
- configuration
- design patterns that are not needed

The goal is a working and understandable gateway, not an unnecessarily large enterprise system.

### Rule 3 — Security from the beginning

We will never commit:

- passwords
- JWT secrets
- API keys
- Redis credentials
- `.env` files containing secrets

We will also avoid logging:

- JWT tokens
- passwords
- sensitive request bodies
- confidential user information

### Rule 4 — Test after changes

After important changes, use Maven tests and API testing.

Typical commands:

```powershell
.\mvnw.cmd clean test
```

and later:

```powershell
.\mvnw.cmd spring-boot:run
```

### Rule 5 — Work only on the assigned branch

The user works on:

```text
develop2
```

The teammate works on:

```text
develop1
```

The `main` branch should remain stable and should be changed through the team's agreed merge/PR process.

---

# 5. Stage 0 — Project and Git Foundation

## Goal

Make sure the project has a clean and safe development foundation.

### Completed

- Repository cloned.
- `develop2` branch checked out.
- `.gitignore` configured.
- IntelliJ files ignored.
- Maven `target/` ignored.
- Environment files ignored.
- Working tree verified clean.

Current useful `.gitignore` entries include:

```gitignore
.idea/
*.iml
target/
.env
.env.*
```

### Why this matters

Build output and IDE-specific files should not be committed.

Environment files can contain secrets and therefore should not be pushed to GitHub.

---

# 6. Stage 1 — Basic Spring Boot Gateway

## Goal

First prove that the gateway application itself works.

At this stage we are NOT forwarding requests yet.

Architecture:

```text
Client
   |
   v
Spring Boot Gateway
   |
   v
Simple response
```

Example:

```text
GET /hello
```

Response:

```text
Hello from Bridge Gateway
```

### What we learn

- How Spring Boot starts.
- How HTTP requests reach the application.
- How controllers/endpoints work.
- How to test an endpoint.
- How to structure the gateway project.

### Security focus

Even though this is only a simple endpoint, we should avoid exposing unnecessary information or debug details.

### Completion condition

The application starts successfully and the endpoint returns the expected response.

---

# 7. Stage 2 — Basic Request Routing

## Goal

Make the gateway forward a request to another backend service.

Architecture:

```text
Client
   |
   | GET /api/hello
   v
Gateway :8080
   |
   | forward
   v
Backend :8081
```

The client communicates with the gateway rather than directly calling the backend.

### Example

Client:

```text
GET http://localhost:8080/api/hello
```

Gateway forwards to:

```text
http://localhost:8081/hello
```

### What we learn

- Reverse proxy concepts.
- Request forwarding.
- Backend communication.
- Gateway request/response flow.

### Security focus

The gateway must not allow clients to provide arbitrary destination URLs.

For example, we should NOT build something like:

```text
/api/proxy?url=<any URL>
```

because uncontrolled forwarding can create SSRF/security problems.

Destinations should be controlled by gateway configuration/routing rules.

---

# 8. Stage 3 — Multiple Services and Routing Rules

## Goal

Support several backend services.

Example:

```text
                 +--> User Service :8081
                 |
Client --> Gateway
                 |
                 +--> Order Service :8082
                 |
                 +--> Product Service :8083
```

Example routes:

```text
/api/users/**     -> User Service
/api/orders/**    -> Order Service
/api/products/**  -> Product Service
```

### Why this matters

This demonstrates why an API Gateway is useful.

The client only needs to know:

```text
Gateway address
```

instead of:

```text
User service address
Order service address
Product service address
```

### Security focus

Routing should use known/controlled services rather than arbitrary destinations.

---

# 9. Stage 4 — Authentication with JWT

## Goal

Protect gateway routes using authentication.

Expected request:

```http
Authorization: Bearer <JWT>
```

The gateway will:

1. Receive the request.
2. Extract the JWT.
3. Validate the token.
4. Check expiration and relevant claims.
5. Allow valid requests.
6. Reject invalid/missing tokens.

### Expected responses

Missing/invalid authentication:

```text
401 Unauthorized
```

### Important security rule

JWTs must never be printed in application logs.

---

# 10. Stage 5 — Authorization

## Goal

Authentication answers:

> Who are you?

Authorization answers:

> What are you allowed to do?

Example roles:

```text
USER
ADMIN
```

Example:

```text
/api/products
```

might be accessible to normal users.

An administrative route might require:

```text
ADMIN
```

### Expected response

Authenticated but not permitted:

```text
403 Forbidden
```

### Why this matters

This demonstrates that the gateway is doing more than simply checking whether a token exists.

---

# 11. Stage 6 — Rate Limiting

## Goal

Prevent a client from sending unlimited requests.

Example rule:

```text
10 requests per second
```

If the limit is exceeded:

```text
429 Too Many Requests
```

Architecture:

```text
Client
   |
   v
Gateway
   |
   v
Rate Limiter
   |
   +---- allowed ----> Backend
   |
   +---- rejected ---> 429
```

### Why rate limiting matters

It helps protect backend services from:

- accidental request floods
- abusive clients
- excessive traffic
- some denial-of-service scenarios

### Important limitation

Rate limiting is one layer of protection. It is not a complete DDoS defense.

---

# 12. Stage 7 — Redis

## Goal

Use Redis to store shared rate-limit state.

### Why not only use Java memory?

Imagine two gateway instances:

```text
Client
  |
  +--> Gateway 1
  |
  +--> Gateway 2
```

If each gateway keeps its own counter, the client could effectively receive separate limits from each instance.

Redis provides shared state:

```text
Gateway 1 ---\
              \
               --> Redis
              /
Gateway 2 ---/
```

### What we learn

- Shared state.
- Redis basics.
- Distributed rate limiting.

---

# 13. Stage 8 — Token Bucket Rate Limiting

## Goal

Implement a controlled rate-limiting algorithm.

Example:

```text
Bucket capacity = 10 tokens
Refill = 2 tokens/second
```

Each request consumes a token.

If a token is available:

```text
Request -> allowed
```

If no token is available:

```text
Request -> 429
```

### Why Token Bucket?

It allows controlled bursts while still enforcing an average rate.

We will keep the implementation understandable rather than building an unnecessarily complicated rate limiter.

---

# 14. Stage 9 — Logging

## Goal

Create useful gateway logs.

Useful information can include:

```text
HTTP method
Request path
Response status
Request duration
Request ID
```

Example concept:

```text
GET /api/users/15 -> 200 -> 42ms -> requestId=abc123
```

### Security

Never log:

```text
JWT tokens
Passwords
API keys
Sensitive request bodies
```

Logs should help debugging without becoming a security risk.

---

# 15. Stage 10 — Request Tracing

## Goal

Give each request a request ID.

Example:

```text
X-Request-ID: 7f83a1...
```

The same ID can appear in gateway/backend logs.

Architecture:

```text
Client
  |
  | Request ID
  v
Gateway
  |
  | same Request ID
  v
Backend
```

### Why?

If a request fails, developers can search logs using one identifier.

This becomes especially useful when multiple services are involved.

---

# 16. Stage 11 — Safe Error Handling

## Goal

Return safe responses when something fails.

Example:

```text
Gateway -> Backend
             X
          unavailable
```

Gateway should return something appropriate such as:

```text
503 Service Unavailable
```

### Security rule

Do not expose:

- Java stack traces
- internal file paths
- database details
- internal service information
- sensitive configuration

to clients.

---

# 17. Stage 12 — Timeouts

## Goal

Prevent the gateway from waiting forever for a backend.

Example:

```text
Gateway
   |
   | request
   v
Backend
   |
   | hangs...
```

The gateway should have a defined timeout.

If the backend doesn't respond within that period, the gateway can stop waiting and return an appropriate error.

### Why?

Without timeouts, stuck backend requests can consume gateway resources and eventually affect other users.

---

# 18. Stage 13 — Retry

## Goal

Optionally retry certain temporary failures.

Retries must be used carefully.

For example, retrying a read operation may be reasonable in some situations.

But blindly retrying:

```text
POST /payment
```

could potentially cause duplicate operations.

Therefore retry rules must consider whether an operation is safe to repeat.

---

# 19. Stage 14 — Circuit Breaker

## Goal

Protect the gateway and backend when a service repeatedly fails.

Without a circuit breaker:

```text
Gateway
  |
  +--> failing service
  +--> failing service
  +--> failing service
  +--> failing service
```

The gateway keeps sending requests.

With a circuit breaker:

```text
Gateway
  |
  v
Circuit Breaker
  |
  X
Failing service
```

The gateway temporarily stops sending requests to the unhealthy service.

This can help prevent cascading failures.

---

# 20. Stage 15 — Security Hardening

After the main features work, review the complete system.

Check:

### Secrets

No secrets in Git.

### Authentication

JWT validation must be correct.

### Authorization

Roles/permissions must be enforced correctly.

### Headers

Do not blindly trust security-sensitive client headers.

### Routing

Do not allow arbitrary destination URLs.

### Input

Validate relevant input.

### Errors

Do not expose internal implementation details.

### Logs

Do not expose credentials or tokens.

### Rate limiting

Check whether clients can easily bypass the intended limit.

### Dependencies

Avoid unnecessary dependencies and keep important dependencies updated.

### Internal services

Backend services should not unnecessarily be exposed directly to external clients.

---

# 21. Stage 16 — API Testing

Use Postman to test the gateway.

Important cases:

### Successful request

```text
Expected: 200
```

### Missing JWT

```text
Expected: 401
```

### Invalid JWT

```text
Expected: 401
```

### Insufficient permissions

```text
Expected: 403
```

### Rate limit exceeded

```text
Expected: 429
```

### Backend unavailable

```text
Expected: 503
```

### Successful routing

```text
Gateway -> correct backend
```

---

# 22. Stage 17 — Automated Tests

Important gateway behavior should eventually have automated tests.

Focus on behavior rather than trying to test every line.

Examples:

```text
JWT accepted
JWT rejected
Correct route selected
Unauthorized request rejected
Rate limit enforced
Backend failure handled
```

---

# 23. Stage 18 — Docker

Once the local version is stable, we can containerize the project.

Possible final local environment:

```text
Docker Compose
│
├── Gateway
├── Redis
├── User Service
├── Order Service
└── Product Service
```

### Why Docker?

It makes it easier to run the complete project consistently.

We will do this later rather than introducing Docker complexity at the beginning.

---

# 24. Stage 19 — Health and Monitoring

We can add Spring Boot Actuator later.

For example:

```text
/actuator/health
```

This can allow us to check whether the gateway is running.

Additional metrics can be added if they are useful to the project.

---

# 25. Stage 20 — Documentation

The final README should explain:

1. What Bridge Gateway is.
2. Why we built it.
3. Problems it solves.
4. Architecture.
5. Technology stack.
6. Security features.
7. Routing.
8. Rate limiting.
9. Redis.
10. How to run it.
11. How to test it.
12. Team responsibilities.

This project-development guide is separate from the final README so that we can keep detailed development notes without making the main README unnecessarily large.

---

# 26. Final Expected Architecture

The target architecture is approximately:

```text
                         +----------------+
                         |     Client     |
                         +-------+--------+
                                 |
                                 v
                     +-----------------------+
                     |    Bridge Gateway     |
                     |-----------------------|
                     | Authentication        |
                     | Authorization         |
                     | Routing               |
                     | Rate Limiting         |
                     | Request ID            |
                     | Logging               |
                     | Error Handling        |
                     | Timeouts              |
                     +----------+------------+
                                |
              +-----------------+------------------+
              |                 |                  |
              v                 v                  v
       +------------+    +------------+    +------------+
       |   User     |    |   Order    |    |  Product   |
       |  Service   |    |  Service   |    |  Service   |
       +------------+    +------------+    +------------+
              \                 |                  /
               \                |                 /
                +---------------+----------------+
                                |
                              Redis
                    (shared rate-limit state)
```

---

# 27. How We Will Work Through the Project

For every stage we will follow:

```text
Understand
    ↓
Plan
    ↓
Implement
    ↓
Run
    ↓
Test
    ↓
Security check
    ↓
Git commit
    ↓
Push to develop2
    ↓
Next stage
```

We should never make a large number of unrelated changes and then push everything at once.

Small commits make it easier to understand the project and recover from mistakes.

---

# 28. Current Next Step

Before implementing Stage 1, verify the current project.

Use the Maven Wrapper from the `gateway` directory:

```powershell
.\mvnw.cmd -version
```

Then:

```powershell
.\mvnw.cmd clean test
```

The current project should compile and its initial test should pass.

After that we will inspect the application class and configuration, then build the first simple gateway endpoint.

---

# 29. Git Workflow

Our normal workflow will be:

```powershell
git status
```

Make one logical change.

Test:

```powershell
.\mvnw.cmd clean test
```

Then:

```powershell
git status
git add .
git commit -m "Short meaningful message"
git push origin develop2
```

Before pushing, check that no secrets or unwanted files are included.

Examples of useful commit messages:

```text
Add basic gateway endpoint
Add backend request routing
Add JWT authentication
Add Redis rate limiter
Add request tracing
```

---

# 30. Project Goal

The final project should demonstrate that we understand how an API Gateway works rather than simply copying a framework example.

The main concepts we want to demonstrate are:

```text
Security
   +
Routing
   +
Traffic Control
   +
Observability
   +
Reliability
```

The implementation should remain small enough for a 2-member university project while still solving meaningful real-world problems.
