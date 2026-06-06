# ADR-0001: Selenium Manager Migration (replace WebDriverManager)

**Status:** Proposed
**Date:** 2026-06-06
**Decider:** Qytera Quality GmbH
**Scope:** QTAF Maintenance-Mode reactivation

## Context

QTAF currently uses **Boni Garcia's WebDriverManager 5.8.0** (March 2024) for browser driver provisioning. After 19 months without an active release, the CI on `windows-latest` (and likely `ubuntu-latest`) is broken in two ways:

1. **ChromeDriver version mismatch.** `browser-actions/setup-chrome@v1` installs Chrome 149 (current stable). WebDriverManager 5.8.0 only knows ChromeDriver mappings up to ~Chrome 123. Result: `SessionNotCreatedException: This version of ChromeDriver only supports Chrome version 149`.
2. **Microsoft Edge driver host shut down.** `msedgedriver.azureedge.net` was decommissioned by Microsoft in 2024. WebDriverManager 5.8.0 still resolves the dead host and crashes the test JVM (`UnknownHostException`). WebDriverManager 6.1.0 (latest, April 2025) has the same hardcoded URL — bumping does **not** fix it, and it introduces a new regression: `No proper candidate URL to download IEDriverServer` (Internet Explorer EOL since 2022).

Additionally, **Selenium 4.6+ (April 2023) ships with Selenium Manager built-in** — Selenium's own driver provisioning. QTAF uses **Selenium 4.20.0** and therefore already has Selenium Manager available, but `DriverFactory.java` explicitly calls `WebDriverManager.chromedriver().setup()` instead of letting Selenium Manager auto-resolve.

This means QTAF carries a **redundant driver-management stack** (Selenium Manager + WebDriverManager), and the WebDriverManager part is the broken one.

## Decision

**Migrate QTAF from WebDriverManager to Selenium Manager:**

1. Remove `<dependency><groupId>io.github.bonigarcia</groupId><artifactId>webdrivermanager</artifactId></dependency>` and the `<webDriverManagerVersion>` property from `qtaf-core/pom.xml`.
2. Refactor `de.qytera.qtaf.core.selenium.DriverFactory` and the per-browser driver classes (`ChromeDriver.java`, `FirefoxDriver.java`, `EdgeDriver.java`): remove all `WebDriverManager.*().setup()` calls. Selenium 4.20 will auto-resolve drivers via Selenium Manager.
3. Add Surefire `<argLine>--add-opens=java.base/java.lang=ALL-UNNAMED</argLine>` to work around Gson's reflection on `Throwable.detailMessage` on Java 17+ (Selenium-internal error serialization needs this).
4. Verify the 575 existing unit tests stay green on both `ubuntu-latest` and `windows-latest`, with the temporary `-DexcludedGroups=ie,edge,chrome,driver` workaround **removed** (or at minimum trimmed back to `ie,edge`).

## Alternatives

| Option | Pro | Con |
|--------|-----|-----|
| **A. Selenium Manager Migration (proposed)** | Single-stack driver mgmt, no recurring drift, ~2-4h once | Code refactor in `DriverFactory`, regression risk |
| B. WebDriverManager 5.9.x minor bump | 5 min effort | Does **not** fix Chrome 149 issue; Edge CDN still dead |
| C. WebDriverManager 6.1.0 major bump | Larger change | Verified 2026-06-06 to introduce IEDriverServer EOL regression — strictly worse |
| D. Pin Chrome version in CI | 30 min | Symptom treatment, drift recurs every 3 months when stable Chrome ticks up |
| E. Exclude chrome+driver test groups permanently (current state via this PR) | 5 min | Loses Selenium test coverage — acceptable as **temporary** workaround, not as final state |

## Productisability assessment

| Criterion | Rating | Notes |
|-----------|--------|-------|
| Multi-tenant | ✅ | Affects only QTAF library, every downstream consumer benefits |
| Self-service | ✅ | Users of QTAF don't need to think about driver management |
| Packaging | ✅ | Lighter dependency tree (one less Maven dependency) |
| Vendor lock-in | ✅ | Reduces lock-in (Selenium Manager is part of Selenium itself) |
| Cost at scale | ✅ | Zero cost, fewer transitive deps to audit |

## P2P assessment (per QYQ-System ADR-0005)

| Challenge | Question | Answer |
|-----------|----------|--------|
| Feature focus | Product problem or customer wish? | **Product problem** — broken CI blocks every contributor |
| Roadmap ownership | Who decided? | Architect (Maintenance-Mode reactivation sprint 2026-06-06) |
| Engineering capacity | More or less maintenance? | **-maintenance** (eliminates recurring ChromeDriver drift) |
| Technical debt | New variant or consolidation? | **Consolidation** (removes redundant stack) |
| Discovery | Based on data or assumptions? | **Data** — failing CI runs 27060252048, 27060487256, 27061207599 documented |
| Standardization | All customers or just one? | **Standard** — affects every QTAF user |
| Change readiness | Reversible? | **Yes** — Selenium Manager is opt-in, can re-add WebDriverManager if needed |

## Consequences

**Positive:**
- Single source of truth for driver provisioning (Selenium Manager).
- No more ChromeDriver/Chrome version drift breaking CI.
- Lighter `qtaf-core` dependency tree (one less transitive cluster).
- Removes a 2-year-stale third-party dependency.

**Negative / Risk:**
- Requires code change in `DriverFactory` and per-browser driver classes — non-trivial test surface.
- Selenium Manager downloads drivers at runtime via Selenium-controlled HTTP endpoint — different network failure modes than WebDriverManager (failure manifests in Selenium's own code, not in a separate library).
- Gson `Throwable.detailMessage` reflection issue persists across both stacks; needs the `--add-opens` workaround independently.
- Tests of WebDriverManager-specific behaviour (if any) need to be removed or rewritten.

## Implementation plan (Estimated 2-4h)

1. **Branch**: `chore/migrate-to-selenium-manager` from `develop`.
2. **Remove dependency**: edit `qtaf-core/pom.xml` (remove `<dependency>` block and `<webDriverManagerVersion>` property).
3. **Refactor**: in `de.qytera.qtaf.core.selenium.*Driver.java`, remove `WebDriverManager.*().setup()` calls. Let `new ChromeDriver()` / `new FirefoxDriver()` / `new EdgeDriver()` use Selenium Manager directly.
4. **Surefire `--add-opens`**: add `<argLine>--add-opens=java.base/java.lang=ALL-UNNAMED</argLine>` to the surefire-plugin configuration in `qtaf-core/pom.xml`.
5. **CI**: in `.github/workflows/Test.yml`, trim `-DexcludedGroups=ie,edge,chrome,driver` back to `-DexcludedGroups=ie,edge` once Selenium Manager works on both OS.
6. **Verify**: 575 tests green locally + green on Ubuntu CI + green on Windows CI (Chrome+Firefox tests should now pass).
7. **Documentation**: update `README.md` to reflect that QTAF no longer requires WebDriverManager.

## Related

- **PR #336** (this PR): applies the temporary workaround `-DexcludedGroups=ie,edge,chrome,driver`. ADR-0001 is the planned permanent fix.
- **QYQ-System ADR-0024**: QTAF Maintenance + qtaf-playwright-core Dual-Track (platform-level decision, scheduled 2026-06-07/08).
- **Upstream**: Selenium Manager documentation — https://www.selenium.dev/documentation/selenium_manager/
