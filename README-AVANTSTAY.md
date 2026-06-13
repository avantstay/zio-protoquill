# AvantStay fork notes

This is AvantStay's fork of [li-nkSN/zio-protoquill](https://github.com/li-nkSN/zio-protoquill)
(the community-maintained continuation of [zio/zio-protoquill](https://github.com/zio/zio-protoquill);
`io.getquill` has no ProtoQuill release past 4.8.6, Oct 2024). The fork exists to:

1. run CI we control (Blacksmith runners),
2. carry patches that are not yet released upstream or on the li-nk fork,
3. build the vendored ProtoQuill artifacts consumed by AvantStay's backend.

## Branch map

| Branch | What it is |
|---|---|
| `main` | Default. li-nk fork lineage (tracks `li-nkSN/zio-protoquill` `main`) plus our CI commits. |
| `master` | Mirror of upstream `zio/zio-protoquill` `master` lineage, inherited from the fork network. |
| `avantstay/fix-quat-cache-scope` | Source of the published vendored artifact, tagged `v4.8.7-avantstay.1`. Upstream `master` @ `dc8505c` (the #720 macro-caching perf work) + the QuatMaking cache-scoping fix + regression sentinels (upstreamed as [li-nkSN/zio-protoquill#13](https://github.com/li-nkSN/zio-protoquill/pull/13)). |
| `update/*`, `feature/*` | Inherited Scala Steward / feature branches from li-nk. Steward and release-drafter are disabled on this fork. |

## CI

- **`Scala CI`** (`ci.yml`, inherited): `sqltest` / `db` / `bigdata` matrix on
  Blacksmith runners. Triggers on PRs, pushes to `main`, and GitHub Releases.
  The `db` job's Oracle wait polls an actual login, not just the listener port —
  the port opens 1–2 minutes before the XE database registers its SID, and fast
  runners otherwise lose that race to `ORA-12505` (#3).
- **`avantstay publish`** (`avantstay-publish.yml`, ours): on PRs it runs the
  `sqltest` gate only; on manual `workflow_dispatch` it can publish to GitHub
  Packages.

Do not publish GitHub Releases on this repo casually: the inherited `release`
job will trigger `sbt ci-release` and fail for lack of secrets. Harmless, but
red.

## Publishing

- The inherited `release` job (`sbt ci-release` → Maven Central under
  `org.li-nk`) is dead on this fork: no signing/Sonatype secrets, and that
  namespace belongs to li-nkSN. Left in place to keep the diff against the
  li-nk fork small.
- The `avantstay publish` workflow can publish `quill-sql_3` + `quill-jdbc_3`
  to GitHub Packages via `workflow_dispatch` (credential-free fallback;
  currently unused).
- **Vendored releases are published manually from a local checkout to
  AvantStay's internal package repository (AWS CodeArtifact).** Versions are
  `<upstream-base>-avantstay.<N>` under the `org.li-nk.protoquill` groupId
  (Scala packages remain `io.getquill`, so artifacts are binary drop-ins), and
  each published version's source commit is tagged here (`v4.8.7-avantstay.1`).

Only `quill-sql` and `quill-jdbc` are published; `quill-engine` / `quill-util`
resolve as stock `io.getquill:*:4.8.5` from Maven Central. **Do not bump the
engine to 4.8.6+ and do not pin the li-nk engine**: both carry the raw-query
decoder corruption from upstream commit `c334986f` ("Do not wrap top-level
infixes") — entity-typed `sql"...".as[Query[T]]` loses its wrapping projection
and decodes positionally against table DDL order. Issue:
[zio/zio-quill#3403](https://github.com/zio/zio-quill/issues/3403), fix PRs:
[zio/zio-quill#3404](https://github.com/zio/zio-quill/pull/3404),
[li-nkSN/zio-quill#5](https://github.com/li-nkSN/zio-quill/pull/5).

## When to retire the vendored pin

When the li-nk fork releases ProtoQuill with
[li-nkSN/zio-protoquill#13](https://github.com/li-nkSN/zio-protoquill/pull/13)
merged **and** an engine containing the
[zio/zio-quill#3404](https://github.com/zio/zio-quill/pull/3404) fix, drop the
vendored version and consume `org.li-nk` straight from Maven Central.
