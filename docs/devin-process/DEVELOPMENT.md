# How the balance-reader modernization was done, step by step

This is a plain-language log of everything that happened while the
**balance-reader** service was moved from Java 8 to Java 21. It is written for
readers who do not work with code every day, so each technical term is
explained the first time it appears.

A few words used throughout:

- **Repository (repo)** — the folder of files that make up the application,
  with a full history of every change.
- **Branch** — a private copy of the repo where work happens before it is
  accepted. `legacy-java8` is the branch that represents "the bank as it runs
  today".
- **Commit** — one saved change, identified by a short code (e.g. `7218cd01`).
- **Pull request (PR)** — a proposal to add a set of commits to a branch,
  together with a description and a review discussion.
- **Devin session** — one working session of the AI engineer; it has its own
  link where the full transcript of what it did can be read.
- **Service** — one independently deployable part of the application. This
  application has eight; only one of them was changed.

All times are UTC, taken from the repository history.

## Chronological log

| Step | Date/time (from git) | What happened | Devin capability used | Evidence |
| --- | --- | --- | --- | --- |
| 1 | 2022-01-10 20:16 | Starting point: this repo is a copy ("fork") of Google's open-source *Bank of Anthos* sample, frozen at release v0.5.3, which runs on Java 8 and Spring Boot 2.3.1 (both no longer receive security fixes). | None — human setup | Base commit [`dcdeb0cb`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/dcdeb0cb170f81255927142738395ba163d2e393) ("release/v0.5.3") |
| 2 | [TODO Rafael: date the fork and the `legacy-java8` branch were created — not recorded in git] | A human created the fork and the `legacy-java8` branch, which is the baseline the modernization work is measured against. | None — human setup | Branch `legacy-java8`; [TODO Rafael: paste fork/branch creation evidence if you want it linked] |
| 3 | [TODO Rafael: paste session link and start time of Session 1] | Session 1 started: Devin read the service, decided what had to change, and agreed the scope — one service only, no change to the other services, the Kubernetes deployment files or the automated build pipeline. | Planning | Session: https://app.devin.ai/sessions/fdb76861cef5456ab7459df728238cdf (the session linked from PRs #1–#3) |
| 4 | 2026-08-19 16:23 | The upgrade itself: Java 8 → Java 21, Spring Boot 2.3.1 → 3.5.16, all supporting libraries and build tools raised to versions that work on Java 21, renamed `javax.*` packages changed to `jakarta.*`, and the container image switched to a Java 21 base image. Also added `MIGRATION.md`, a plain-language explanation of the change. | Code migration | Commit [`7218cd01`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/7218cd018744603905c7f15ddf1eb997e70fb1aa); [`src/balancereader/MIGRATION.md`](../../src/balancereader/MIGRATION.md) |
| 5 | 2026-08-19 16:24 | Pull request [#1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1) opened against `legacy-java8`, with the reasoning for every version change and the results of the build and test runs. | Code migration (PR authoring) | [PR #1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1) |
| 6 | 2026-08-19 16:34 | **Devin Review** (an automatic code review that reads the proposed change and looks for defects) reported 3 problems on PR #1: trace sampling silently reduced to 10%, a JSON library pinned to the wrong version, and a version catalogue imported for the wrong framework generation. | Devin Review | PR #1 review comments [3814887151](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3814887151), [3814887303](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3814887303), [3814887493](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3814887493); see also [BUGS.md](./BUGS.md) rows B1–B3 |
| 7 | 2026-08-19 16:37 | All 3 review findings fixed in one commit and each review comment answered and marked resolved. | Auto-fix (acting on review feedback) | Commit [`f198258a`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/f198258a06263589a2ff78c0dfba3b48d8bff478) |
| 8 | [TODO Rafael: date/time of the environment snapshot failure and fix] | The pre-built machine image Devin starts from (its "environment snapshot") failed to build, so the tools needed to compile and test the service were missing or stale. The setup was corrected and rebuilt. | Environment setup | Snapshot build job `sbj-2b4ccd69a15c40438f670697dabae2d6`; [TODO Rafael: paste the build-job link and what the fix changed] |
| 9 | 2026-08-19 16:40–17:01 (approx.) | A **testing agent** (a Devin that sets up the application and exercises it like a user would) brought up the complete 8-service application with plain Docker — no Kubernetes cluster — with only balance-reader replaced by the upgraded version. It logged into the demo account in a browser, made a $250.00 deposit and a $100.00 payment through the *old, unchanged* services, and read the balances back through the *upgraded* service; every amount matched to the cent. It also ran the old and the new version of the service side by side against the same database and compared the answers request by request: 13 of 15 were byte-for-byte identical. | Testing agent | Test-results comment on PR #1 ([5345400519](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#issuecomment-5345400519)), including screenshots and the comparison table; [TODO Rafael: paste testing-agent session link] |
| 10 | [TODO Rafael: paste session link] | The testing agent found 3 problems and fixed them itself during the session. | Testing agent + auto-fix | [TODO Rafael: paste the exact list of the 3 bugs and the fix evidence] — see [BUGS.md](./BUGS.md) rows B6–B8 |
| 11 | 2026-08-19 17:00 | The two remaining differences from the comparison were written up in `MIGRATION.md`: a completely malformed login token now returns "not authorised" instead of "server error", and an error message field that used to be empty is no longer shown. Both refuse the request as before. | Code migration (documentation of findings) | Commit [`d6db61df`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/d6db61dfea69510b06562df21a2f895d41085fe2) |
| 12 | 2026-08-19 17:26 | The testing recipe was saved as a reusable **skill** — a written procedure Devin can follow in future sessions (how to start all eight services with Docker, how to create login tokens, the traps that produce false "everything passed" results). | Skills / knowledge | Commit [`f705e958`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/f705e958f55db2184c1e520759f0d28cb17ebf41), [PR #2](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/2), file [`.agents/skills/testing-bank-of-anthos-docker/SKILL.md`](../../.agents/skills/testing-bank-of-anthos-docker/SKILL.md) |
| 13 | 2026-08-19 17:27 | PR #2 (the skill) merged into `legacy-java8`. | Human merge | Merge commit [`d7ed6d0b`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/d7ed6d0b) |
| 14 | 2026-08-19 17:27 | PR #1 (the upgrade) merged into `legacy-java8`. | Human merge | Merge commit [`bdfc1cb0`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/bdfc1cb0) |
| 15 | 2026-08-19 17:31 | Devin Review ran again on PR #1 and reported a 4th problem: the logging library was left with its five parts on two different versions, which can stop the service from logging or from starting. PR #1 had already been merged, so the fix had to go into a new PR. | Devin Review | PR #1 review comment [3815279941](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3815279941) |
| 16 | 2026-08-19 17:33 | Fix delivered as [PR #3](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/3): the version is now set once, in one place, so all five logging parts move together (still 2.26.0). Verified by listing the resolved versions, rebuilding, re-running the tests and restarting the container against the real database. | Auto-fix + code migration | Commit [`79a8730a`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/79a8730a5e3428261f3c60073cc039f4ad0f5a78) |
| 17 | 2026-08-19 17:39 | Devin Review on PR #3: no issues found. | Devin Review | PR #3 review (2026-08-19 17:39) |
| 18 | 2026-08-19 17:41 | PR #3 merged into `legacy-java8`. This is the current state of the branch. | Human merge | Merge commit [`e6d441f7`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/e6d441f710d73242453a26e6035bed6de8249f81) |

## What this shows about the way of working

- The upgrade was proposed, reviewed, corrected and merged inside a single
  afternoon, and every step left written evidence behind.
- Nothing was accepted on trust: the code review found 4 real defects the
  build and the unit tests had not caught, and the end-to-end test compared
  the upgraded service against the old one on the same data.
- The work was deliberately kept to one service, so the change can be rolled
  back by redeploying the previous image, with no data migration involved.
- The reusable skill means the next service (transaction-history or
  ledger-writer) can be tested the same way without rediscovering how.
