# JGit EE8 Servlet Bridge — Design Proposal

> Design proposal for adding **first-class EE8 (`javax.servlet`) modules** to
> JGit, alongside the canonical `jakarta.servlet` modules, so downstream
> consumers that cannot migrate to `jakarta.servlet` can keep tracking JGit
> master.

---

## TL;DR

JGit master ships its servlet-facing modules on `jakarta.servlet`
(Servlet 6 / Jetty 12 EE10). Major downstreams — **Gerrit Code Review,
Gitiles, the Gerrit plugin ecosystem** — still run on `javax.servlet`
(Servlet 4 / Jetty 12 EE8) and **cannot migrate yet**. Today that blocks them
from consuming JGit master at all, and a parallel servlet-4 JGit branch is
unsustainable (it already blocks JGit-submodule updates on Gerrit `stable-3.12`
and `stable-3.13`).

**Ask:** add **first-class EE8 modules** to JGit —
`org.eclipse.jgit.http.server.ee8`, `org.eclipse.jgit.lfs.server.ee8`, and their
test modules — as supported, full members of the build. They are **generated**
from the canonical sources (only `jakarta.servlet` → `javax.servlet` /
Jetty EE10 → EE8 rewritten, **package names and source line numbers
preserved**), but otherwise get the **complete first-class treatment**: Bazel
**and** Maven builds, real OSGi bundle manifests, test suites, Eclipse/IntelliJ
source attachment + debugging, and Maven-built artifacts suitable for future
Maven Central publication. This mirrors Jetty 12's own EE8+EE10 dual model.

We are deliberately asking for the **most minimally invasive** support the JGit
community can give: additive, generated modules; the canonical **production**
servlet modules and sources untouched; original package names preserved (zero
import churn for consumers); and **no change to the JGit p2 update site**. The
new maintenance surface is small and, for the Bazel transform, **shared**: a
direction-agnostic `sed` transform toolchain in bazlets (also used by Gitiles,
and by the Gerrit companion wiring), the Maven copy/rewrite wiring, a small
hand-maintained EE8 `AppServer` test overlay, and the tests that exercise and
guard them.

A complete, tested implementation is already up for review under topic
[`ee8-servlet-bridge`](https://review.gerrithub.io/q/topic:%22ee8-servlet-bridge%22).

---

## The shared transform toolchain

The Bazel EE8 generation uses a single, shared toolchain that lives in
[bazlets](https://gerrit.googlesource.com/bazlets)
(`@com_googlesource_gerrit_bazlets//tools`):

- `servlet_transform.bzl` — the `transform_srcjar` rule that produces the
  flavour srcjar.
- `junit.bzl` — the `junit_tests` macro, with an optional `suite_srcs`
  parameter so a transformed test `.srcjar` is compiled while the
  `@Suite.SuiteClasses` names are scanned from the canonical `.java` sources
  (valid because the transform preserves package and class names).
- `generated_srcs_test.sh` — the generation checker (derivation, line-count,
  and direction-specific residue checks).

The rewrite is a plain **`sed` package-prefix substitution**, so it pulls in
**no external dependency at all**. The universal servlet/Jetty package mapping
is built into the rule; a consumer selects only a **direction** (`to_javax`
for JGit's `.ee8`, `to_jakarta` for Gitiles' `.ee10`) and carries **no renames
data** of its own. Being dependency-free, the same toolchain works on both
bzlmod (JGit master) and WORKSPACE (the `stable-7.4` backport) branches.

**Trade-off for the JGit community to weigh:** the Bazel EE8 generation depends
on the bazlets repo (which JGit master already references via `git_repository`).
If JGit prefers the transform to live in-tree, the same `sed` rule can instead
be vendored under `tools/jgit-ee8` with no functional change — it is
deliberately dependency-free.

---

## Why we need it

- JGit master uses `jakarta.servlet`, so **Gerrit cannot consume JGit master
  as-is**.
- Gerrit / Gitiles / plugins on Jetty 12 **EE8** cannot move to
  `jakarta.servlet` now or in the near future — a non-trivial migration across
  the servlet and dependency-injection (Guice) surface; see the options
  analysis under [References](#references).
- A separate **servlet-4 JGit branch is not sustainable** — it was never a
  stable release line, and it forces continuous merge work from master.
- It is a **hard blocker for updating the JGit submodule on Gerrit's supported
  stable branches** (`stable-3.12`, `stable-3.13`). In-depth options analysis:
  <https://davido.github.io/gerrit-jgit-maintenance/>

---

## First-class, not a shim

This is the key point, and it is what distinguishes the proposal from a
downstream hack: the EE8 modules are **full members of JGit**, not a temporary
bridge living in a consumer's build.

- **Both toolchains, at parity with canonical** — Bazel (a shared, `sed`-based
  servlet-flavour transform from bazlets) **and** Maven (copy-and-rewrite
  reactor); the Maven build + tests have been verified with the series.
- **Real OSGi bundles** — each EE8 module carries its own
  `META-INF/MANIFEST.MF` (`Bundle-SymbolicName: …ee8`,
  `Import-Package: javax.servlet;version="[4.0.0,5.0.0)"`, Jetty EE8 imports),
  `build.properties`, and Eclipse PDE project files — i.e. valid OSGi bundles
  just like every other JGit jar.
- **Tested** — in Bazel, `generated_srcs_test` (the shared bazlets checker)
  guards the generation (sources derived from the canonical filegroups, line
  counts preserved, no `jakarta` residue); the Maven copy/rewrite path is
  exercised by the EE8 Maven reactor tests. Both run the servlet-facing HTTP/LFS
  suites against the generated EE8 jars.
- **IDE-supported** — generated source jars attach to the EE8 binary jars, so
  breakpoints in the running `javax.servlet` classes bind and step correctly;
  verified live in **Eclipse and IntelliJ** (developer story documented in
  `Documentation/dev-eclipse.txt` and `dev-intellij.txt` on the Gerrit side).
- **Publishable (future option)** — the Maven-built EE8 artifacts are suitable
  for **Maven Central** publication if the project chooses to release them, for
  non-Bazel consumers.
- **Canonical production untouched** — the EE8 modules and the EE8 Maven
  reactor are *additive*; the canonical **production** servlet modules and
  sources are not modified. (The root Maven reactor gains the EE8 reactor, and
  the only canonical-source change is the shared `junit.http` test-helper
  refactor that dedupes the EE8 `AppServer` overlay.)

The only thing that distinguishes the EE8 modules from the canonical ones is
that their sources are **generated** rather than hand-edited — deliberately, so
they stay in lock-step and we avoid hand-maintaining ~6,500 LoC of duplicated
servlet code (4,409 in `http.server`, 2,058 in `lfs.server`) plus the
corresponding tests. First-class status, without duplication cost.

---

## Scope — the one deliberate restriction: no Tycho/p2 publishing

There is exactly one place the EE8 modules are intentionally **not** first-class,
and it is by necessity: they are **not** added to the JGit feature set, the
`org.eclipse.jgit.repository` **p2 update site**, or the Tycho **target
platform**.

The reason is OSGi identity. The EE8 modules keep the **same exported packages —
the same class FQDNs** — as the canonical modules, differing only in the servlet
API. An EE8 bundle and its canonical counterpart therefore **must never sit on
the same classpath**. Adding both to the JGit features or target platform would
make two bundles exporting the same packages available in the same install /
runtime resolution space, unless explicit conflicts were modeled — so we keep
that ambiguity out of the update site entirely. The EE8 jars are instead
consumed as plain artifacts (optionally, in future, from **Maven Central**) by
consumers that put exactly **one** of the two on their classpath. Everything else —
Bazel + Maven build, real OSGi manifests, tests, Eclipse/IntelliJ debugging —
remains first-class.

This is also precisely **why both toolchains are first-class**. With the p2
update site off the table, Bazel and Maven *are* the two delivery channels:
**Bazel** serves source consumers (Gerrit builds JGit from source and picks up
the EE8 module directly from its targets), and **Maven** provides the
build/test path and the artifact shape needed for binary consumers if the
project later chooses to publish them (e.g. to **Maven Central**).
Together they give EE8 a complete, supported delivery story that does not rely
on — and does not pollute — the JGit p2 update site.

---

## Previous related work — the earlier attempt and why it was reverted

A prior attempt lived **on the Gerrit side** and rewrote the servlet API on the
fly when repacking the canonical `jgit-servlet` jar:

- **Original** — Ivan Frade, *"lib/BUILD: Replace jakarta with javax in
  jgit-servlet sources"* (2024-12-18, `Ib47963a`):
  <https://gerrit-review.googlesource.com/c/gerrit/+/446221>
- **Reverted** — Matthias Sohn, *"Revert 'lib/BUILD: Replace jakarta with javax
  in jgit-servlet sources'"* (2025-01-17, `I1cd5f80`):
  <https://gerrit-review.googlesource.com/c/gerrit/+/447822>
  > Reason for revert: this … only works in bazel build but breaks the build in
  > Eclipse

It was Bazel-only, Gerrit-local, and not covered by Eclipse/PDE integration or
JGit-owned generation tests. **This proposal is the opposite approach** —
first-class EE8 modules *inside JGit*, covered by tests and IDE debugging, while
leaving the canonical production servlet modules and sources untouched. It
addresses that revert reason by moving the rewrite into JGit-owned modules with
Maven- and Eclipse-visible metadata, source attachment, and tests.

---

## Implementation (already up for review — topic `ee8-servlet-bridge`)

- JGit: <https://review.gerrithub.io/q/topic:%22ee8-servlet-bridge%22>
- Gerrit companion:
  <https://gerrit-review.googlesource.com/q/topic:%22ee8-servlet-bridge%22>

**JGit master series — 5 commits** (the GerritHub topic also contains separate
stable-7.4 EE8 experiments/backports that are not part of this request):

1. **Bazel: Rename `//lib:servlet-api` → `//lib:jakarta-servlet-api`** —
   disambiguate the label before introducing the `javax` one.
2. **Bazel: Add EE8 servlet bridge targets** — the generated `.ee8` libraries
   (`org.eclipse.jgit.http.server.ee8`, `…lfs.server.ee8`). They consume the
   shared bazlets `transform_srcjar` rule with `direction = "to_javax"`
   (rewrites only servlet imports — `jakarta` → `javax`, Jetty EE10 → EE8 —
   keeping the original Java packages and source line numbers). JGit carries no
   transform code or renames data of its own.
3. **Bazel: Add EE8 servlet test targets** — EE8 test modules and the shared
   `generated_srcs_test`; the servlet-facing HTTP/LFS suites run against the
   generated EE8 jars, built with the bazlets `junit_tests` macro (canonical
   suite class names supplied via its `suite_srcs` parameter).
4. **Maven: Add EE8 build, tests, and resources** — a Maven reactor and OSGi
   manifests so the EE8 jars also build and test under Maven.
5. **junit.http: Extract AppServerBase to dedupe the EE8 AppServer overlay** —
   refactor of the one test helper that cannot be auto-rewritten (a structural
   Jetty EE8 security-API difference) so it shares a base class instead of being
   a copy.

> The shared toolchain itself is added by a **bazlets** change,
> *"tools: Add shared servlet-flavour transform toolchain"*, which JGit (and
> Gitiles/Gerrit) then consume.

**Gerrit companion** — *"Bazel: Move Gerrit off JGit servlet-4 via EE8 bridge"*:
rewires `//lib:jgit-servlet` to the EE8 module, attaches the generated EE8
source jar in `tools/eclipse/project.py`, documents the Eclipse + IntelliJ
developer story, and adds a Bazel `somepath()` no-duplicate-classpath test
(canonical and EE8 jars share class FQDNs and must not co-exist on a classpath).

---

## Status and verification

The JGit implementation is complete and locally green on **both toolchains**
(verified on the series head, *"junit.http: Extract AppServerBase to dedupe the
EE8 AppServer overlay"*):

- **Bazel** — `bazelisk test //tools/jgit-ee8:generated_srcs_test
  //org.eclipse.jgit.http.test.ee8:http
  //org.eclipse.jgit.lfs.server.test.ee8:lfs_server` → all pass.
- **Maven** (Java 17) — `mvn -DskipTests install` then
  `mvn -f org.eclipse.jgit.ee8/pom.xml test` → `BUILD SUCCESS`, 293 tests,
  0 failures, 0 errors.

The public review systems should still be treated as the source of truth for
current CI state, especially for companion changes that may need to be
re-verified after the prerequisite bazlets/JGit changes are published.

Adjacent ecosystem validation is also in place, outside this JGit request:
Gitiles has migrated to the same shared bazlets transform in the opposite
direction (`to_jakarta`) for its EE10 servlet flavour, and the Gerrit LFS plugin
has migrated to consume the JGit EE8 servlet bridge.

The `generated_srcs_test` enforces the generation invariants: the EE8 sources
are derived from the canonical `srcs` filegroups, source line counts are
preserved, no `jakarta` residue remains, and `javax.servlet` references are
present. Structural checks confirm the canonical servlet sources are unmodified,
the EE8 modules carry OSGi manifests importing `javax.servlet`, and no EE8
module appears under `org.eclipse.jgit.packaging` — so nothing is added to the
features, the p2 repository, or the target platform.

---

## The bigger picture (future goal — explicitly NOT part of this request)

The longer-term direction is a **"flavoured" Gerrit** that, mirroring Jetty 12,
produces **two artifacts from one tree**:

- **`release.war`** — EE8 / `javax.servlet` (the default, what current
  deployments need),
- **`release-ee10.war`** — EE10 / `jakarta.servlet` (the forward path).

Reaching that requires **every major Gerrit dependency — JGit, Gitiles, and the
plugin ecosystem — to provide both EE8 and EE10 variants.** Adding first-class
EE8 modules to JGit is the **first building block** — and the shared bazlets
transform is what lets the *same* toolchain produce JGit's `.ee8` flavour and
Gitiles' `.ee10` flavour from one rule. To be clear: the dual-WAR work is a
**future goal and is not requested here** — it is the *motivation* for why JGit
(and the others) should expose EE8 as a first-class citizen alongside EE10. As
Nasser Grainawi framed it on the Gerrit contributor call and in the
[#jgit channel of the Gerrit Discord](https://discord.com/channels/775374026587373568/1512148475893121114)
(viewable by members of the Gerrit Discord):

> Hmm, that's an interesting idea. Another idea geared towards the future support
> problem … was what if we followed the jetty approach in jgit/gitiles/gerrit
> and have in-tree support for both servlet-4 and servlet-6+. It's absolutely
> more overhead to maintain both and maybe it falls apart when you start looking
> at other dependencies, but it could be a path to providing a newer servlet
> version while still providing compatibility with what Google needs.

---

## Alternatives considered

- **Gerrit-local on-the-fly rewrite** (the reverted approach) — broke the
  Eclipse build, no Eclipse/PDE or JGit-owned generation-test coverage,
  downstream-only.
- **A long-lived servlet-4 JGit branch** — unsustainable; ongoing merge burden;
  never a stable release line.
- **Copying servlet code into a parallel EE8 source tree** — ~6,500 LoC plus
  duplicated tests to hand-maintain; rejected in favour of generation.
- **Relocate the EE8 classes to distinct `org.eclipse.jgit.*.ee8` packages**
  (which *would* make them p2-co-installable). Rejected on two counts: (1) it
  breaks the simple, low-risk generation — instead of rewriting only servlet
  imports, the transform would have to relocate every package declaration and
  every internal reference; and (2) it forces **every consumer to change its
  imports** (Gerrit code would no longer keep its existing
  `org.eclipse.jgit.http.server.*` imports), defeating the drop-in purpose.
  Keeping the original package names is exactly what makes both the generation
  trivial *and* the consumer change zero. The same-FQDN choice is thus
  deliberate, with the p2 exclusion above as its only cost.

---

## References

- Maintenance options analysis:
  <https://davido.github.io/gerrit-jgit-maintenance/>
- JGit reviews (topic `ee8-servlet-bridge`):
  <https://review.gerrithub.io/q/topic:%22ee8-servlet-bridge%22>
- Gerrit companion (topic `ee8-servlet-bridge`):
  <https://gerrit-review.googlesource.com/q/topic:%22ee8-servlet-bridge%22>
- Prior attempt (Ivan Frade):
  <https://gerrit-review.googlesource.com/c/gerrit/+/446221>
- Revert (Matthias Sohn):
  <https://gerrit-review.googlesource.com/c/gerrit/+/447822>
