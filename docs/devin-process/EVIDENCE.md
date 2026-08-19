# How to prove the upgrade worked

Everything below can be checked by anyone, either by opening a link or by
running a command. No access to a Devin session is required.

## 1. Read the change itself

| What to look at | Link |
| --- | --- |
| The full proposal, its reasoning and the review discussion | [PR #1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1) |
| The change to `pom.xml` — the file that declares the Java version and every library the service uses | [PR #1 diff of `src/balancereader/pom.xml`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1/files#diff-c858009df7f8553601ce4c497e0492f782e48be3b3d0fdf1e3a894828a42fe39) |
| **Before** — Java 8 (`1.8`) and Spring Boot `2.3.1.RELEASE` | [`pom.xml` lines 31 and 35 at the v0.5.3 baseline](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/blob/dcdeb0cb170f81255927142738395ba163d2e393/src/balancereader/pom.xml#L31-L35) |
| **After** — Java `21` and Spring Boot `3.5.16` | [`pom.xml` lines 31 and 35 on `legacy-java8` today](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/blob/legacy-java8/src/balancereader/pom.xml#L31-L35) |
| The plain-language explanation of the whole migration, including the section **"How this was verified"** | [`MIGRATION.md` — How this was verified](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/blob/legacy-java8/src/balancereader/MIGRATION.md#how-this-was-verified) |
| The four defects the automatic review caught, and the answers to them | [PR #1 conversation](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1) and [PR #3](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/3) |
| The end-to-end test results, with browser screenshots of real balances and the old-vs-new comparison table | [Test results comment on PR #1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#issuecomment-5345400519) |
| The reusable testing procedure | [`.agents/skills/testing-bank-of-anthos-docker/SKILL.md`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/blob/legacy-java8/.agents/skills/testing-bank-of-anthos-docker/SKILL.md) |
| The follow-up that put all logging library parts on one version | [PR #3](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/3) |

## 2. Reproduce the build and the tests yourself

You need Java 21 (a JDK, the developer kit) and Maven (the build tool that
compiles the service and runs its tests).

```bash
git clone https://github.com/rafaelmacedo-rep/bb-bank-java-modernization.git
cd bb-bank-java-modernization
git checkout legacy-java8
cd src/balancereader

java -version          # must report 21
mvn clean verify       # compiles the service and runs its 9 automated tests
```

Expected result: `BUILD SUCCESS` and
`Tests run: 9, Failures: 0, Errors: 0, Skipped: 0`.

Two more optional checks, run from the same folder:

```bash
mvn checkstyle:check   # coding-style rules -> "You have 0 Checkstyle violations."
mvn jib:buildTar       # builds the container image on eclipse-temurin:21-jre-alpine
mvn dependency:list | grep log4j    # all five log4j2 parts must read 2.26.0
```

## 3. Reproduce the side-by-side comparison of the old and new service

This is the strongest evidence, because it compares the upgraded service
against the *original, untouched* one on the *same* database: any difference in
an answer is either a real regression or a framework change that must be
explained. The full procedure, including the traps that produce false
"everything passed" results, is the skill file linked above; the outline is:

1. Pull the eight unmodified release images (they need no login):
   `gcr.io/bank-of-anthos-ci/{frontend,userservice,contacts,transactionhistory,ledgerwriter,balancereader,accounts-db,ledger-db}:v0.5.3`.
2. Build the upgraded service locally as a container image
   (`mvn jib:dockerBuild` inside `src/balancereader`).
3. Start the two databases first with `USE_DEMO_DATA=True` and wait about 25
   seconds, then the other services, pointing them at each other by container
   name (see skill §3). Every Java service must be started with
   `--hostname <service>-0` and `NAMESPACE=default`, otherwise it crashes for a
   reason unrelated to the upgrade (see BUGS.md row B11).
4. Run the **upgraded** image and the **unmodified v0.5.3** image at the same
   time on two different host ports, both connected to the same `ledger-db`.
5. For every endpoint and every login-token case, call both services and
   compare the answer, the status code and the content type — for example:

   ```bash
   for port in 8081 8082; do
     curl -s -o body.$port -w '%{http_code} %{content_type}\n' \
       -H "Authorization: Bearer $TOKEN" \
       http://localhost:$port/balances/1011226111
   done
   diff body.8081 body.8082
   ```

   Cases worth covering (skill §4): a valid token for the matching account, a
   valid token for a different account, unreadable text instead of a token, an
   expired token, a token signed with the wrong key, an unsigned token, and no
   token at all.
6. Confirm the live-data path as well: insert a transaction directly into the
   ledger and check that the balance both services report changes within the
   polling interval, then restart the upgraded service so the balance is
   recomputed from scratch and check it still matches.

Result obtained in this project: **13 of 15 comparisons byte-for-byte
identical**, with the two differences explained in
[BUGS.md](./BUGS.md) rows B9 and B10 and in `MIGRATION.md`.

## 4. What this evidence does *not* cover

- Export of traces and metrics to Google Cloud — needs real cloud
  credentials, which were not available; verified only as "does not break
  startup".
- A deployment on a real Kubernetes cluster — none was available; the Docker
  bring-up above is the equivalent.
- Performance under production-like data volumes.
