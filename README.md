# spring-boot-micrometer-tracing-sample

[![Java CI with Maven](https://github.com/hendisantika/spring-boot-micrometer-tracing-sample/actions/workflows/maven.yml/badge.svg)](https://github.com/hendisantika/spring-boot-micrometer-tracing-sample/actions/workflows/maven.yml)
![JDK](https://img.shields.io/badge/JDK-25-orange?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.x-6DB33F?logo=springboot)

Spring Boot Micrometer Tracing Sample: distributed tracing across two reactive (WebFlux)
services with **Micrometer Tracing** and the **Brave** bridge. A request to the `client`
service is forwarded to the `server` service, and both log the **same `traceId`**.

## Architecture

```
curl :8080/all ──► client (port 8080) ──HTTP interface + WebClient──► server (port 9090) /customers
                   logs traceId/spanId                               logs traceId/spanId
```

| Module   | Port | Description                                                                                                           |
|----------|------|-----------------------------------------------------------------------------------------------------------------------|
| `server` | 9090 | `GET /customers` returns a `Flux` of 10 random customers.                                                              |
| `client` | 8080 | `GET /all` calls the server through a declarative HTTP interface (`@GetExchange`) backed by `WebClient`, then returns the result. |

Both services call `Hooks.enableAutomaticContextPropagation()` so the tracing context (and the
MDC values `traceId`/`spanId`) follows Reactor operators across threads.

## Tech Stack

- Java **25**
- Spring Boot **4.1.x** (Spring Framework 7)
  - `spring-boot-starter-webflux`, `spring-boot-starter-actuator`
  - `spring-boot-starter-webclient` (client only)
  - `spring-boot-micrometer-tracing-brave` + `io.micrometer:micrometer-tracing-bridge-brave`
- Maven Wrapper (one per module)

> **Spring Boot 4 notes**
> - `WebClientAdapter.forClient(...)` / `HttpServiceProxyFactory.builder().clientAdapter(...)` were removed.
>   Use `HttpServiceProxyFactory.builderFor(WebClientAdapter.create(webClient))`.
> - `WebClient.Builder` auto-configuration lives in `spring-boot-starter-webclient`.
> - Tracing auto-configuration lives in `spring-boot-micrometer-tracing-brave`; without it no `traceId` is logged.

## Prerequisites

- JDK 25 (e.g. Temurin 25)
- No local Maven needed; use `./mvnw`

## Build & Test

```bash
(cd server && ./mvnw verify)
(cd client && ./mvnw verify)
```

## Run

In two terminals:

```bash
cd server && ./mvnw spring-boot:run   # http://localhost:9090
```

```bash
cd client && ./mvnw spring-boot:run   # http://localhost:8080
```

Then call the client:

```bash
curl http://localhost:8080/all
```

```json
[{"id":0,"name":"55a3a223-d484-4a91-9d23-9b9122d4ded6"},{"id":1,"name":"5d213a58-2fa2-4d4b-8cf1-aa6485a95008"}, ...]
```

### Expected logs

The log level pattern is `%5p [${spring.application.name}, %X{traceId}, %X{spanId}]`.
The `traceId` is the same in both services; each service has its own `spanId`:

```text
# client
INFO [client, 6afa4298c8df7fa57321110fd0e8649c, 7321110fd0e8649c] ... i.m.h.client.CustomerController : Fetching customers
# server
INFO [server, 6afa4298c8df7fa57321110fd0e8649c, 5c241de23a1b25ce] ... i.m.h.server.CustomerController : Generating new customers 6afa4298c8df7fa57321110fd0e8649c | 5c241de23a1b25ce
```

## CI/CD

GitHub Actions workflow [`.github/workflows/maven.yml`](.github/workflows/maven.yml):

- Runs on push and pull requests to `main`, and on manual dispatch.
- Builds and tests `server` and `client` in a matrix on JDK 25 (Temurin) with `./mvnw verify`.
- On push to `main`, submits each module's Maven dependency graph to GitHub for Dependabot alerts.

## Dependabot

[`.github/dependabot.yml`](.github/dependabot.yml) checks for updates weekly (Monday 05:00 Asia/Jakarta):

- **Maven** (`/server` and `/client`), grouped into `spring-boot`, `micrometer-tracing`, `maven-plugins`
  and `other-dependencies` (minor/patch), with the commit prefix `chore(deps)`.
- **GitHub Actions**, all bumps grouped into one PR with the commit prefix `ci(deps)`.
- PRs are labelled `dependencies` and assigned to the maintainer.

## License

See [LICENSE](LICENSE).
