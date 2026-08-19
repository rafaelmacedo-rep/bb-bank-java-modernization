# Transaction History: Java 8 / Spring Boot 2.3 → Java 21 / Spring Boot 3.5 migration

## In one paragraph

The transaction-history service used to run on Java 8 and Spring Boot 2.3.1,
both of which reached end of free public support years ago (no security
patches, no vendor support). This change moves the service to Java 21 and
Spring Boot 3.5, which are the current long-term-supported versions. **No
business logic changed**: the service still exposes the same four HTTP
endpoints, reads the same `TRANSACTIONS` table, caches the same per-account
transaction lists the same way, and returns the same values. Every change below
is either "same code, new package name" or "same library, newer version". The
balance-reader service was migrated first; this change deliberately reuses its
exact target versions and conventions, including the lessons learned from the
problems found on that pull request.

## Why now

- Java 8 and Spring Boot 2.3 no longer receive security fixes, so any newly
  discovered vulnerability in the framework stays unpatched.
- The rest of the industry (and Google's upstream Bank of Anthos sample) has
  moved on; staying behind makes every future dependency bump harder.
- Java 21 brings better memory/CPU efficiency in containers, which typically
  lowers the pod's footprint at the same traffic level.
- Keeping the two ledger read services (balance-reader, transaction-history) on
  the same framework generation means one set of versions to maintain instead of
  two.

## What changed

Only files inside `src/transactionhistory` were touched. Other services, the
Kubernetes manifests, and the CI workflows were left alone.

### 1. `pom.xml` — framework and dependency versions

| Item | Before | After | Why |
| --- | --- | --- | --- |
| `spring-boot-starter-parent` | 2.3.1.RELEASE | 3.5.16 | Latest supported Spring Boot 3.x; same version as balance-reader |
| `java.version` | 1.8 | 21 | Target the current Java LTS |
| Spring Cloud GCP starters | `org.springframework.cloud:spring-cloud-gcp-*` (Hoxton.SR5) | `com.google.cloud:spring-cloud-gcp-*` (BOM 5.13.11) | Google moved these artifacts to their own coordinates/release train; the Hoxton line does not support Spring Boot 3 |
| Spring Cloud release train | Hoxton.SR5 | removed | Nothing in this service uses a plain Spring Cloud artifact once the GCP starters come from Google's own BOM, so the import is dead weight |
| `micrometer-registry-stackdriver` | pinned to `${micrometer.version}` | version managed by Spring Boot | Avoids a hand-maintained version that can drift out of sync with the framework |
| log4j2 | two artifacts pinned individually to 2.17.1 | one `log4j2.version` property set to 2.26.0 | Security/bugfix updates, and one setting keeps **all** log4j2 parts on the same version (see below) |
| Guava | 30.1.1-jre | 32.1.3-jre | Java 21 compatibility |
| Jackson databind | 2.11.0 | version managed by Spring Boot | Jackson's modules must all be on one version; letting the framework pick it prevents a mismatch |
| Lettuce | 5.2.1 | 6.8.2 | Java 21 / Boot 3 compatibility |
| `java-jwt` (Auth0) | 3.9.0 | 4.5.2 | Supported line; same API surface used here |
| Mockito | 2.7.2 | 5.20.0 | Mockito 2 cannot instrument Java 21 classes |
| jib-maven-plugin | 2.1.0 | 3.5.1 | Needed for a Java 21 base image and current registry APIs |
| surefire | 2.22.2 | 3.5.4 | Java 21 support |
| checkstyle plugin | 3.1.0 | 3.6.0 | Java 21 syntax support |
| jacoco | 0.8.5 | 0.8.13 | Java 21 bytecode support (0.8.5 cannot read it) |

Two mistakes made on the balance-reader upgrade were avoided here from the
start:

- **The logging library is versioned once, not per part.** The balance-reader
  upgrade pinned two of the five log4j2 artifacts individually and left the rest
  on the framework's older default, which can stop logging or prevent startup
  (recorded as B4 in `docs/devin-process/BUGS.md`, fixed by PR #3). Here the
  single `log4j2.version` property governs the whole library. Verified: every
  log4j2 artifact on the classpath resolves to 2.26.0.
- **No hand-written version for parts of a library the framework manages.** The
  Jackson and Micrometer pins were dropped rather than carried over (B2/B3), and
  no BOM for a future framework generation is imported.

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
  silently dropped tracing to one request in ten (B1). The old rule that skipped
  tracing for `cleanup*` and `favicon` URLs was dropped rather than rebuilt:
  this service serves no such URLs, so it never had an effect here.
- `checkstyle.xml`: the `LineLength` rule moved from inside `TreeWalker` to the
  top level. Newer Checkstyle versions reject the old placement. The rule itself
  is unchanged (still 80 characters), so linting is exactly as strict as before.

No test was deleted or weakened; the existing test file needed no changes at all.

### 4. Two things specific to this service that were checked deliberately

Unlike the balance-reader, this service hands a **mutable, ordered** collection
straight from the database layer into its cache and then modifies it in place:

- `TransactionRepository.findForAccount` declares its result as a
  `LinkedList<Transaction>`, and the cache stores it as a `Deque<Transaction>`.
  Spring Data JPA 3 still produces a real, mutable `LinkedList` for that
  declared return type, so no type change was needed.
- The background ledger reader then calls `addFirst` / `removeLast` on that
  cached list to keep it at `HISTORY_LIMIT` entries. Nothing in the new stack
  substitutes an immutable list there, so the newest-first ordering and the
  trimming behaviour are unchanged. Both were confirmed against a real database
  in the end-to-end test below.

## How this was verified

- `mvn clean verify` inside `src/transactionhistory`: **BUILD SUCCESS**, 8/8
  unit tests pass, JaCoCo coverage report generated.
- `mvn checkstyle:check`: **0 violations**.
- `mvn jib:buildTar`: container image builds successfully on the Java 21 base
  image.
- Dependency check: all log4j2 artifacts on the classpath resolve to 2.26.0.
- End-to-end test of the whole application with plain Docker: the upgraded
  service was run alongside the other, unchanged services (`ledger-db`,
  `frontend` and the rest) and driven through the web UI, and the endpoints were
  called directly with valid and invalid login tokens.
- The old (Java 8) and the new (Java 21) version of the service were run side by
  side against the **same** database and their answers compared request by
  request. The results of that comparison are recorded in the pull request for
  this change.

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
   scope). They must run on a JDK 21+ toolchain for this module to compile; the
   remaining services still target Java 8 and continue to build under JDK 21.
4. **Database behaviour under Hibernate 6.** Spring Boot 3 upgrades Hibernate 5
   → 6. The entity mapping and the JPQL queries — including the paged,
   newest-first history query — were exercised against a real PostgreSQL
   instance and returned identical results, but a load test against
   production-like data volumes is recommended to confirm query plans and
   connection-pool behaviour.
5. **Error-code consumers.** Where the new version answers a rejected request
   with a different status code than the old one (see the comparison in the pull
   request), confirm no dashboard, alert, or client treats those cases
   differently.
6. **Rollback.** The change is contained in one service and one image tag, so
   rollback is simply redeploying the previous `transactionhistory` image; no
   data migration is involved.
