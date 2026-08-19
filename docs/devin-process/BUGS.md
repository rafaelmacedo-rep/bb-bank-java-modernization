# Problems found during the balance-reader modernization

Every problem discovered along the way, where it was found, and how it ended.
"Root cause" is the underlying reason, in plain language. The environment
problem (row B5) is included even though it was not a defect in the bank's
code.

Terms used below:

- **Devin Review** — an automatic code review that reads a proposed change and
  reports likely defects before it is accepted.
- **Testing agent** — a Devin that starts the whole application and exercises
  it like a real user, then reports what differs.
- **Build** — the automated compile-and-test run.
- **Trace / tracing** — records of individual requests, used to investigate
  slowness. **Sampling** is the share of requests recorded.

| ID | Where it was found | Description | Root cause | How it was fixed (commit) | Status |
| --- | --- | --- | --- | --- | --- |
| B1 | Devin Review on [PR #1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3814887151) | Only about 1 request in 10 would have been traced instead of all of them — a silent loss of investigation data, with no error anywhere. | The tracing settings were only partly renamed. The new framework reads different setting names and defaults to 10% sampling; the old names were still in the file but ignored. | Settings rewritten to the new names with sampling kept at 100% — [`f198258a`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/f198258a06263589a2ff78c0dfba3b48d8bff478) | Fixed and resolved |
| B2 | Devin Review on [PR #1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3814887303) | The library that converts data to and from JSON (the format services exchange) was pinned to an older version than the framework expects, which can fail at runtime with hard-to-diagnose errors even though everything compiles and the tests pass. | A hand-written version number for one part of that library was carried over from the old configuration, while the framework supplied a newer version for its other parts. | The hand-written version was removed so the framework manages all parts as one consistent set — [`f198258a`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/f198258a06263589a2ff78c0dfba3b48d8bff478) | Fixed and resolved |
| B3 | Devin Review on [PR #1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3814887493) | A shared version catalogue meant for the *next* major framework generation was imported. No effect yet, but the first component later taken from it would fail to start. | The catalogue was raised to the newest release instead of the one matched to Spring Boot 3.5; it was in fact no longer needed at all, because the Google Cloud components now come from Google's own catalogue. | The unnecessary import and its version setting were deleted — [`f198258a`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/f198258a06263589a2ff78c0dfba3b48d8bff478) | Fixed and resolved |
| B4 | Devin Review on [PR #1](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/1#discussion_r3815279941), reported after PR #1 had already been merged | The logging library shipped with its five parts on two different versions (two at 2.26.0, the rest at 2.24.3). Logging can then stop working, or the service can refuse to start. Confirmed as real on the merged branch, not theoretical. | The upgrade set the version on two parts individually instead of setting it once for the whole library, leaving the remaining parts on the framework's older default. | The two individual pins were replaced by a single version setting, so all five parts move together (still 2.26.0) — [`79a8730a`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/79a8730a5e3428261f3c60073cc039f4ad0f5a78), delivered as [PR #3](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/pull/3) | Fixed and merged |
| B5 | Devin's own working environment (not in the bank's code) | The pre-built machine image Devin starts from failed to build, so the tools needed to compile and test the service were missing or out of date on the machine. | [TODO Rafael: paste the root cause from the snapshot build job `sbj-2b4ccd69a15c40438f670697dabae2d6`] | [TODO Rafael: paste what the fix changed and the link to the successful rebuild] — no repository commit; this was an environment change | Fixed (environment only) |
| B6 | Testing agent | [TODO Rafael: paste bug 1 of the 3 found by the testing agent] | [TODO Rafael: root cause] | [TODO Rafael: commit SHA or "fixed inside the session"] | [TODO Rafael] |
| B7 | Testing agent | [TODO Rafael: paste bug 2 of the 3] | [TODO Rafael: root cause] | [TODO Rafael: commit SHA] | [TODO Rafael] |
| B8 | Testing agent | [TODO Rafael: paste bug 3 of the 3] | [TODO Rafael: root cause] | [TODO Rafael: commit SHA] | [TODO Rafael] |

## Findings from testing that were recorded but deliberately not "fixed"

These three came out of the end-to-end testing and are visible in the PR #1
history. They may or may not be the same three items as B6–B8 above
([TODO Rafael: confirm which of these are the 3 bugs the testing agent fixed]).

| ID | Where it was found | Description | Root cause | Outcome | Status |
| --- | --- | --- | --- | --- | --- |
| B9 | Testing agent (side-by-side comparison, 13 of 15 responses identical) | A completely malformed login token — text that is not a token at all — now gets "not authorised" (401) where the old version returned "server error" (500). | The new token library reports malformed input as a token error, which the service already handles, instead of letting it escape as an unexpected failure. | Left as is and documented: both versions refuse the request, so nobody gains access; the new answer is the more accurate one. Anything that reads error codes should know. | Accepted, documented in [`MIGRATION.md`](../../src/balancereader/MIGRATION.md) via [`d6db61df`](https://github.com/rafaelmacedo-rep/bb-bank-java-modernization/commit/d6db61dfea69510b06562df21a2f895d41085fe2) |
| B10 | Testing agent (same comparison) | When the login token is missing entirely, the error response no longer contains an empty `"message"` field. | A cosmetic change in how the new framework builds error responses. | Left as is; appearance only, the request is refused exactly as before. | Accepted, documented |
| B11 | Testing agent (while setting the application up) | The service crashes at startup on any machine whose hostname does not contain a `-`, and it does so even when metrics collection is switched off. | Code that builds monitoring labels assumes a Kubernetes-style hostname and cuts the text at the first `-`; without one it cuts at an invalid position. | **Not fixed here** — it reproduces on the unmodified Java 8 version, so it is pre-existing and outside a behaviour-preserving upgrade. The workaround (give each service a Kubernetes-style hostname) is written down in the testing skill. | Open, pre-existing, workaround documented in [`SKILL.md`](../../.agents/skills/testing-bank-of-anthos-docker/SKILL.md) |

## Still to be checked at rollout (not defects)

Listed in full in [`src/balancereader/MIGRATION.md`](../../src/balancereader/MIGRATION.md):
the export of traces and metrics to Google Cloud could not be verified without
real cloud credentials; the Kubernetes deployment files still pass two
obsolete memory options; the automated build pipeline must run on Java 21 or
newer; and a load test on production-like data volumes is recommended after
the database layer upgrade that came with the framework.
