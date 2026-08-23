# Transaction History: Java 8 / Spring Boot 2.3 → Java 21 / Spring Boot 3.5 migration

## In one paragraph

The transaction-history service used to run on Java 8 and Spring Boot 2.3.1,
both long past the end of free public support (no security patches, no vendor
support). This change moves the service to Java 21 and Spring Boot 3.5, the
current long-term-supported versions, following the same recipe already applied
to `balancereader` (see `src/balancereader/MIGRATION.md`). **No business logic
changed**: the service still exposes the same HTTP endpoints, reads the same
`TRANSACTIONS` table, caches per-account transaction lists the same way, and
returns the same values. Every change below is either "same code, new package
name" or "same library, newer version".

## What changed

Only files inside `src/transactionhistory` were touched. Other services
(`ledgerwriter`, `ledgermonolith`), the Kubernetes manifests, and the CI
workflows were left alone.

### 1. `pom.xml` — framework and dependency versions

| Item | Before | After | Why |
| --- | --- | --- | --- |
| `spring-boot-starter-parent` | 2.3.1.RELEASE | 3.5.16 | Latest supported Spring Boot 3.x; matches `balancereader` |
| `java.version` | 1.8 | 21 | Target the current Java LTS |
| Spring Cloud GCP starters | `org.springframework.cloud:spring-cloud-gcp-*` (Hoxton.SR5) | `com.google.cloud:spring-cloud-gcp-*` (BOM 5.13.11) | Google moved these artifacts to their own coordinates/release train; the Hoxton line does not support Spring Boot 3 |
| Spring Cloud release train | Hoxton.SR5 | removed | Nothing here uses a plain Spring Cloud artifact once the GCP starters move to Google's own BOM |
| `micrometer-registry-stackdriver` | pinned to `${micrometer.version}` | version managed by Spring Boot | Avoids a hand-maintained version that can drift out of sync with the framework |
| log4j2 | 2.17.1 (pinned per artifact) | 2.26.0 via `${log4j2.version}` | Security/bugfix updates, and one property keeps every log4j2 artifact aligned |
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
- `TransactionHistoryApplication.java`: `javax.annotation.PreDestroy` →
  `jakarta.annotation.PreDestroy`.
  Java EE was donated to the Eclipse Foundation and renamed Jakarta EE; Spring
  Boot 3 only ships the `jakarta.*` packages. The annotations behave identically.
- `TransactionHistoryApplication.java`: the application now excludes
  `ZipkinAutoConfiguration`. Spring Boot 3 replaced Spring Cloud Sleuth with
  Micrometer Tracing, whose default exporter is Zipkin; without this exclusion
  the service would try to ship traces to a non-existent local Zipkin server.
  Tracing still goes to Google Cloud Trace via the GCP trace starter.
- `application.properties`: the tracing settings were rewritten from the old
  `spring.sleuth.*` names to the `management.tracing.*` names the new framework
  reads, so the existing `ENABLE_TRACING` toggle and the "trace every request"
  sampling rate both keep working. Leaving the old names in place would have
  silently dropped tracing to one request in ten. Two old settings were dropped
  rather than rebuilt: `spring.sleuth.web.skipPattern` (this service serves no
  `cleanup*` or `favicon` URLs, so it never had an effect) and
  `spring.sleuth.log.slf4j.enabled=false` (Sleuth-specific; the replacement
  stack does no MDC logging of its own here).
- `checkstyle.xml`: the `LineLength` rule moved from inside `TreeWalker` to the
  top level. Newer Checkstyle versions reject the old placement. The rule itself
  is unchanged (still 80 characters), so linting is exactly as strict as before.

No production Java class other than the two above needed an edit, and no test
was deleted, weakened, or changed: `TransactionHistoryControllerTest` compiles
and passes as-is against Mockito 5 and Micrometer 1.15.

## How this was verified

All under JDK 21 (Temurin/Oracle 21.0.5), inside `src/transactionhistory`:

- `mvn clean verify`: **BUILD SUCCESS**, 8/8 unit tests pass (0 failures,
  0 errors, 0 skipped), JaCoCo report generated for all 10 classes.
- `mvn checkstyle:check`: **0 violations**.
- `mvn jib:buildTar`: container image builds successfully on the Java 21 base
  image.

Runtime behaviour was not re-verified end-to-end for this service; the
equivalent Boot 3 configuration was validated end-to-end for `balancereader`
(same framework, same tracing/metrics setup, same JWT library, same ledger
schema) — see `src/balancereader/MIGRATION.md`.

## Risks to validate before/at rollout

1. **Cloud Trace / Cloud Monitoring export.** The tracing stack was replaced
   wholesale by the framework upgrade, and it can only be checked locally as
   "does not break startup" (no GCP credentials). On a real cluster, confirm
   traces still appear in Cloud Trace and that the Stackdriver metrics
   (including the Guava cache metrics registered by
   `TransactionHistoryController`) still show up in Cloud Monitoring.
2. **Malformed-token error codes.** `java-jwt` 4.x maps a token that is not a
   token at all to a verification exception rather than a generic failure — on
   `balancereader` this turned a `500` into a `401`. Both refuse the request, so
   no access is gained, but confirm no dashboard, alert, or client distinguishes
   those two cases.
3. **Memory settings.** `kubernetes-manifests/transaction-history.yaml` sets the
   obsolete flags `-XX:+UnlockExperimentalVMOptions
   -XX:+UseCGroupMemoryLimitForHeap` via `JVM_OPTS`. Java 21 honours container
   limits natively and will refuse or ignore the old flag. The manifests were
   intentionally not modified (out of scope), so verify the pod starts
   in-cluster and adjust `JVM_OPTS` in a follow-up.
4. **Database behaviour under Hibernate 6.** Spring Boot 3 upgrades Hibernate 5
   → 6. The `Transaction` entity mapping and the paged JPQL query in
   `TransactionRepository` are unchanged, but a run against production-like data
   volumes is recommended to confirm query plans and connection-pool behaviour.
5. **CI toolchain.** The repository CI workflows were not modified (out of
   scope). `.github/workflows/ci-pr.yaml` runs `mvn checkstyle:check` and
   `mvn test` from the aggregator POM, using whatever `default-jdk`
   `install-dependencies.sh` installs on the runner image; that must be JDK 21+
   for this module to compile. The remaining Java 8 modules continue to build
   under JDK 21.
6. **Rollback.** The change is contained in one service and one image tag, so
   rollback is simply redeploying the previous `transactionhistory` image; no
   data migration is involved.
