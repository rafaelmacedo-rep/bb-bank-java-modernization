# Balance Reader: Java 8 / Spring Boot 2.3 → Java 21 / Spring Boot 3.5 migration

## In one paragraph

The balance-reader service used to run on Java 8 and Spring Boot 2.3.1, both of
which reached end of free public support years ago (no security patches, no
vendor support). This change moves the service to Java 21 and Spring Boot 3.5,
which are the current long-term-supported versions. **No business logic
changed**: the service still exposes the same four HTTP endpoints, reads the
same `TRANSACTIONS` table, caches balances the same way, and returns the same
values. Every change below is either "same code, new package name" or "same
library, newer version".

## Why now

- Java 8 and Spring Boot 2.3 no longer receive security fixes, so any newly
  discovered vulnerability in the framework stays unpatched.
- The rest of the industry (and Google's upstream Bank of Anthos sample) has
  moved on; staying behind makes every future dependency bump harder.
- Java 21 brings better memory/CPU efficiency in containers, which typically
  lowers the pod's footprint at the same traffic level.

## What changed

Only files inside `src/balancereader` were touched. Other services, the
Kubernetes manifests, and the CI workflows were left alone.

### 1. `pom.xml` — framework and dependency versions

| Item | Before | After | Why |
| --- | --- | --- | --- |
| `spring-boot-starter-parent` | 2.3.1.RELEASE | 3.5.16 | Latest supported Spring Boot 3.x |
| `java.version` | 1.8 | 21 | Target the current Java LTS |
| Spring Cloud GCP starters | `org.springframework.cloud:spring-cloud-gcp-*` (Hoxton.SR5) | `com.google.cloud:spring-cloud-gcp-*` (BOM 5.13.11) | Google moved these artifacts to their own coordinates/release train; the Hoxton line does not support Spring Boot 3 |
| Spring Cloud release train | Hoxton.SR5 | removed | Nothing in this service uses a plain Spring Cloud artifact once the GCP starters move to Google's own BOM, so the import is dead weight |
| `micrometer-registry-stackdriver` | pinned to `${micrometer.version}` | version managed by Spring Boot | Avoids a hand-maintained version that can drift out of sync with the framework |
| log4j2 | 2.17.1 | 2.26.0 | Security/bugfix updates |
| Guava | 30.1.1-jre | 32.1.3-jre | Java 21 compatibility |
| Jackson databind | 2.11.0 | version managed by Spring Boot | Jackson's modules must all be on one version; letting the framework pick it prevents a mismatch |
| Lettuce | 5.2.1 | 6.8.2 | Java 21 / Boot 3 compatibility |
| `java-jwt` (Auth0) | 3.9.0 | 4.5.2 | Supported line; same API surface used here |
| Mockito | 2.7.2 | 5.20.0 | Mockito 2 cannot instrument Java 21 classes |
| jib-maven-plugin | 2.1.0 | 3.5.1 | Needed for a Java 21 base image and current registry APIs |
| surefire | 2.22.2 | 3.5.4 | Java 21 support |
| checkstyle plugin | 3.1.0 | 3.6.0 | Java 21 syntax support |
| jacoco | 0.8.5 | 0.8.13 | Java 21 bytecode support (0.8.5 cannot read it) |

### 2. Container base image

This service has **no Dockerfile** — the image is built by the
`jib-maven-plugin` declared in `pom.xml`, which is what `skaffold.yaml` invokes.
The Java 21 base image is therefore configured there:
`eclipse-temurin:21-jre-alpine` (previously jib's implicit Java 8/11 default).

### 3. Source code

- `Transaction.java`: `javax.persistence.*` → `jakarta.persistence.*`.
- `BalanceReaderApplication.java`: `javax.annotation.PreDestroy` →
  `jakarta.annotation.PreDestroy`.
  Java EE was donated to the Eclipse Foundation and renamed Jakarta EE; Spring
  Boot 3 only ships the `jakarta.*` packages. The annotations behave identically.
- `BalanceReaderApplication.java`: the application now excludes
  `ZipkinAutoConfiguration`. Spring Boot 3 replaced Spring Cloud Sleuth with
  Micrometer Tracing, whose default exporter is Zipkin; without this exclusion
  the service would try to ship traces to a non-existent local Zipkin server.
  Tracing still goes to Google Cloud Trace via the GCP trace starter.
- `application.properties`: the tracing settings were rewritten from the old
  `spring.sleuth.*` names to the `management.tracing.*` names the new framework
  reads, so the existing `ENABLE_TRACING` toggle and the "trace every request"
  sampling rate both keep working. Leaving the old names in place would have
  silently dropped tracing to one request in ten. The old rule that skipped
  tracing for `cleanup*` and `favicon` URLs was dropped rather than rebuilt:
  this service serves no such URLs, so it never had an effect here.
- `checkstyle.xml`: the `LineLength` rule moved from inside `TreeWalker` to the
  top level. Newer Checkstyle versions reject the old placement. The rule itself
  is unchanged (still 80 characters), so linting is exactly as strict as before.

No test was deleted or weakened; the existing test file needed no changes at all.

## How this was verified

- `mvn clean verify` inside `src/balancereader`: **BUILD SUCCESS**, 9/9 unit
  tests pass, JaCoCo coverage report generated.
- `mvn checkstyle:check`: **0 violations**.
- `mvn jib:buildTar`: container image builds successfully on the Java 21 base
  image.
- End-to-end smoke test against a real PostgreSQL `ledger-db` (same schema as
  `src/ledger-db/initdb`) with a JWT signed by the repo's demo key:
  - `GET /version` → `v0.2.0`; `GET /ready` → `ok`; `GET /healthy` → `ok`;
    `GET /actuator/health` → `UP`.
  - `GET /balances/{acct}` with a matching token → correct balance (`12345`).
  - Token for a different account → `401`; malformed token → `401`.
  - Inserting a new transaction into the ledger and re-reading the balance →
    `12400`, i.e. the background `LedgerReader` thread and the cache update
    still work.
- Full end-to-end test of the whole application: the upgraded service was run
  alongside the seven other, unchanged services and driven through the web UI.
  A deposit and a payment made through the old services were read back through
  the upgraded one and displayed to the cent. The old and the new version of the
  service were also run side by side against the same database and their answers
  compared request by request: 13 of 15 were byte-for-byte identical.

### The one behaviour that is not identical

When the service is sent a completely malformed login token (not just a wrong
one, but text that is not a token at all), the old version returned a generic
"server error" and the new version returns "not authorised". Both versions
refuse the request, so nobody gains access who did not have it before; the new
answer is simply the more accurate one. This comes from the updated token
library and is considered an improvement rather than a defect. Anything that
reads the service's error codes should be aware of it.

## Risks to validate before/at rollout

1. **Cloud Trace / Cloud Monitoring export.** Tracing and metrics could only be
   verified as "does not break startup" locally (no GCP credentials). On a real
   cluster, confirm that traces still appear in Cloud Trace and that the
   Stackdriver metrics (including the Guava cache metrics) still show up in
   Cloud Monitoring. This is the single most likely place for a regression,
   because the tracing library was replaced wholesale by the framework upgrade.
2. **Memory settings.** The deployment sets the obsolete flags
   `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap` via
   `JVM_OPTS`. Java 21 honours container limits natively and will refuse or
   ignore the old flag. The manifests were intentionally not modified (out of
   scope), so verify the pod starts in-cluster and adjust `JVM_OPTS` in a
   follow-up.
3. **CI toolchain.** The repository CI workflows were not modified (out of
   scope). They must run on a JDK 21+ toolchain for this module to compile;
   the other services still target Java 8 and continue to build under JDK 21.
4. **Database behaviour under Hibernate 6.** Spring Boot 3 upgrades Hibernate 5
   → 6. The entity mapping and the native/JPQL queries were exercised against a
   real PostgreSQL instance and returned identical results, but a load test
   against production-like data volumes is recommended to confirm query plans
   and connection-pool behaviour.
5. **Error-code consumers.** As noted above, a malformed token now yields "not
   authorised" instead of "server error". Confirm no dashboard, alert, or
   client treats those two cases differently.
6. **Rollback.** The change is contained in one service and one image tag, so
   rollback is simply redeploying the previous `balancereader` image; no data
   migration is involved.
