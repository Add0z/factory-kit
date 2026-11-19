# Stack overlay: Java / Spring Boot / Gradle

Apply after the kit's INSTALL.md copies `kit/`, when the project is Spring Boot
on Gradle. Each section says where its text goes. Check each version against
the project's JDK, and replace project names (`<your.base.package>`, the hot
spots, the users) with the real ones.

## 1. `factory/checks.sh`

Fix the comment's toolchain version and container list for the project.

```bash
#!/usr/bin/env bash
# The repo's checks. Exit 0 only when the branch is fit to review.
#
# The gate runs this script from the main checkout with the job's worktree as
# the working directory. Keep it fast, deterministic and non-interactive: no
# watch modes, no prompts, nothing that rewrites files.
#
# Spring Boot / Gradle project (Java <n> toolchain). `build` runs `check`:
#   spotlessCheck   formatting (google-java-format, AOSP style)
#   compileJava     javac -Xlint:all -Werror plus Error Prone
#   test            JUnit, ArchUnit rules, Flyway migrations on Testcontainers
#                   (<the project's containers>, so Docker must be running)
#   jacoco          coverage report in build/reports/jacoco
# and then assembles the jar.
set -euo pipefail

if ! docker info >/dev/null 2>&1; then
  echo "Docker is not running; the tests need it for Testcontainers." >&2
  exit 1
fi

./gradlew --console=plain build
```

No `--no-daemon`: the Gradle daemon is shared across job worktrees.

## 2. Quality gates in `build.gradle`

Plugins:

```groovy
    id 'com.diffplug.spotless' version '8.11.0'
    id 'net.ltgt.errorprone' version '5.2.0'
    id 'jacoco'
```

Dependencies (Flyway only for JPA on Postgres; on Boot 3 use
`org.flywaydb:flyway-core` instead of the Boot 4 starter):

```groovy
    implementation 'org.springframework.boot:spring-boot-starter-flyway'
    runtimeOnly 'org.flywaydb:flyway-database-postgresql'
    testImplementation 'com.tngtech.archunit:archunit-junit5:1.5.1'
    errorprone 'com.google.errorprone:error_prone_core:2.53.0'
```

In `tasks.named('test')` add `finalizedBy tasks.named('jacocoTestReport')`, then
append:

```groovy
// Quality gates. `./gradlew build` runs all of them through `check`.
spotless {
    java {
        googleJavaFormat('2.0.0').aosp().reflowLongStrings()
        removeUnusedImports()
        trimTrailingWhitespace()
        endWithNewline()
        targetExclude('build/**')
    }
}

tasks.withType(JavaCompile).configureEach {
    options.compilerArgs.addAll(['-Xlint:all,-processing,-serial', '-Werror'])
    options.errorprone {
        disableWarningsInGeneratedCode = true
        excludedPaths = '.*/build/generated/.*'
    }
}

jacocoTestReport {
    dependsOn tasks.named('test')
    reports {
        xml.required = true
        html.required = true
    }
}

```

- **palantir-java-format breaks on JDK 27** (`NoSuchFieldError ... endPositions`);
  google-java-format 2.0.0 (AOSP, 4 spaces) works.
- `-processing` in `-Xlint` avoids a Lombok false positive.
- `spotlessApply` reformats existing sources: it is part of the change the user
  approves in INSTALL.md step 2. Run it once so the repo starts formatted.
- Prove each gate with a deliberately bad file, then delete it. Bad formatting
  fails `spotlessJavaCheck`; `a == "x"` on Strings fails `compileJava`
  (Error Prone `ReferenceEquality`, fatal by `-Werror`); an `@Autowired` field
  compiles and fails only `ArchitectureTest` (so run `test`, not `-x test`).
- Versions known to work with JDK 27, Gradle 9.9 and Boot 4.1: the ones above.
  On another JDK, run `./gradlew build` after adding them; a failure inside
  the formatter or Error Prone means a version bump.

## 3. `src/main/resources/application.yaml` (JPA projects)

Merge under the existing `spring:` key; don't append a second one.

```yaml
spring:
  jpa:
    hibernate:
      # Flyway owns the schema (src/main/resources/db/migration); Hibernate only checks it.
      ddl-auto: validate
    open-in-view: false
```

Plus `src/main/resources/db/migration/V1__baseline.sql`:

```sql
-- Baseline: the schema starts empty. Every schema change is a new
-- V<n>__<description>.sql file; never edit a migration once it is merged.
SELECT 1;
```

## 4. `src/test/java/<package path>/ArchitectureTest.java`

```java
package <your.base.package>;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noFields;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.repository.Repository;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RestController;

@AnalyzeClasses(
        packagesOf = ArchitectureTest.class,
        importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    @ArchTest
    static final ArchRule noFieldInjection =
            noFields()
                    .should()
                    .beAnnotatedWith(Autowired.class)
                    .because("use constructor injection")
                    .allowEmptyShould(true);

    @ArchTest
    static final ArchRule controllersDoNotUseRepositories =
            noClasses()
                    .that()
                    .areAnnotatedWith(RestController.class)
                    .or()
                    .areAnnotatedWith(Controller.class)
                    .should()
                    .dependOnClassesThat()
                    .areAssignableTo(Repository.class)
                    .because("controllers go through a service")
                    .allowEmptyShould(true);

    @ArchTest
    static final ArchRule controllersDoNotExposeEntities =
            noClasses()
                    .that()
                    .areAnnotatedWith(RestController.class)
                    .should()
                    .dependOnClassesThat()
                    .areAnnotatedWith(jakarta.persistence.Entity.class)
                    .because("controllers return DTOs, not JPA entities")
                    .allowEmptyShould(true);
}
```

## 5. `factory/config.json`

- `protected_paths`: add `"src/main/resources/db/migration/*"` (the gate blocks
  only modified files, so new migrations pass and merged ones stay immutable).
- Backend without screens: `"surface_reviewers": {"ui": {"ux": "sonnet"}, "security": {"security": "opus"}}`.

## 6. Agent text (replaces the kit's `ADAPT` comments)

Each block below replaces the ADAPT comment. Where a block starts with the
line just above the comment (reviewer-ux, reviewer-ui), it replaces that line
too.

**`reviewer-code.md`**, in place of the ADAPT comment in "What to look for":

```markdown
- **Spring and JPA**:
  - `@Transactional` on the service method that owns the unit of work, not on
    controllers or private methods (proxies ignore them); read-only
    transactions for queries.
  - No N+1 queries or `LazyInitializationException` waiting to happen
    (`open-in-view` is off): fetch joins or entity graphs where a collection is
    read.
  - Controllers return DTOs or records, never JPA entities; request bodies are
    validated with `@Valid` and Bean Validation; errors are mapped once in a
    `@ControllerAdvice` as `ProblemDetail`.
  - Layering: controller → service → repository. Constructor injection only.
  - Entities: `equals`/`hashCode` based on id or a business key, not on
    every field; no Lombok `@Data` on entities.
  - Money is `BigDecimal` with an explicit scale and `RoundingMode`, never
    `double`.
  - Schema changes come as a new `src/main/resources/db/migration/V<n>__*.sql`
    that matches the entity changes (`ddl-auto` is `validate`).
  - Configuration via `@ConfigurationProperties`, not scattered `@Value`.
```

and replace the **Tests** bullet with:

```markdown
- **Tests**: new behaviour has tests, and no existing test was weakened,
  skipped (`@Disabled`) or special-cased to pass. Prefer slices
  (`@WebMvcTest`, `@DataJpaTest`, plain unit tests) over `@SpringBootTest`
  when the behaviour doesn't need the whole context.
```

**`reviewer-security.md`**, example from a Spring budgeting backend (keep what
applies, rename the domain):

```markdown
Hot spots in this project (Spring Boot, Java):

- **Budget data is per user**: every JPA repository query, REST controller and
  gRPC service method must scope reads and writes to the authenticated user.
  Flag lookups by id alone (`findById`) that skip an ownership check.
- **Queries**: `@Query` with string concatenation, `EntityManager.createQuery`
  or native SQL built from input. Require bind parameters.
- **Sessions**: Spring Session on Redis. Session fixation, data put into the
  session that should not be, missing logout/invalidation.
- **Actuator**: new exposed endpoints (`management.endpoints.web.exposure`),
  especially `env`, `heapdump`, `configprops`, `loggers` without auth.
- **gRPC**: services exposed without authentication interceptors; reflection
  enabled in production config.
- **Spring AI / Anthropic**: user input placed into prompts (prompt injection
  that drives tools), tool callbacks that read or write other users' data,
  model output rendered or executed unchecked, API keys in `application.yaml`.
- **JobRunr**: dashboard exposed without auth, jobs that take user-controlled
  class or method names.
- **Spring Shell**: commands that run shell or file operations from input.
- **Config**: secrets in `application*.yaml` or `compose.yaml` rather than
  environment variables; `ddl-auto` set to `update`/`create` outside dev.
```

**`approver.md`**, example:

```markdown
In this project, also ESCALATE when the change:

- changes how money amounts are stored, rounded or calculated (types,
  `BigDecimal` scale, currency handling) or alters existing budget data;
- changes JPA entities in a way that alters the database schema, or touches
  `build.gradle`, `compose.yaml` or `application*.yaml`;
- adds or changes Spring AI prompts, tools or model choice, or anything that
  sends user data to Anthropic or another external service;
- exposes new actuator, gRPC or JobRunr endpoints.
```

**`reviewer-ux.md`**, example users line:

```markdown
  gets a sensible path, following the patterns the app already uses. This
  project's users are people managing a personal or household budget, through
  the Spring Shell CLI, and client developers calling its REST and gRPC APIs:
  for them, clear error responses (status codes, messages), consistent
  resource names and money amounts shown with their currency matter most.
```

**`reviewer-ui.md`**, backend without UI:

```markdown
   global stylesheets, and the neighbouring components. This project is a
   backend with no web UI yet; templates and static assets would live in
   `src/main/resources/templates/` and `src/main/resources/static/`, and Spring
   Shell CLI output is the main user-visible text. If the branch adds the first
   UI, check it is self-consistent and uses one styling approach.
```

**`intake.md`**, example project line:

```markdown
This project is a Spring Boot (Java) backend for a budgeting application:
REST (Spring MVC) and gRPC APIs, a Spring Shell CLI, JPA on Postgres, Redis
sessions, JobRunr background jobs and Spring AI (Anthropic) features.
```

## 7. `AGENTS.md`

- **Dev server**: `./gradlew bootTestRun --args='--server.port=<port> --spring.grpc.server.port=0'`
  (Testcontainers gives each job its own database; the fixed gRPC port 9090
  would collide between parallel jobs).
- **Metrics registries**: a Spring Initializr project with
  `micrometer-registry-datadog` or `-dynatrace` and no API key won't start
  (`apiKey was 'null' but it is required`). Remove the unused registries, or
  add `--management.<datadog|dynatrace>.metrics.export.enabled=false` to the
  dev-server `--args` **[USER decides]**.
- **Single test**: `./gradlew test --tests '<Class>'`. **Formatter**:
  `./gradlew spotlessApply`.
- **Conventions**: controller → service → repository; DTOs/records out of
  controllers; constructor injection; Flyway owns the schema; `BigDecimal`
  with scale and currency for money; `ProblemDetail` from one
  `@ControllerAdvice`; test slices first.

## 8. Known Initializr problems

- **Spring AI Redis chat memory**: `spring-ai-starter-model-chat-memory-repository-redis`
  (Jedis) connects to `localhost:6379` and ignores Testcontainers'
  `@ServiceConnection`, so `contextLoads()` fails. Initializr also generates
  two `@ServiceConnection(name = "redis")` beans (`redis` and `redis-stack`).
  Unless the project uses chat memory now, remove the starter and the
  `redisStackContainer` bean **[USER decides]**.

## 9. `.mcp.json`

Add `"--exclude", "build", "--exclude", ".gradle"` to CodeGraph's `args`.

## 10. jdtls (Java LSP)

```bash
brew install jdtls
claude plugin install jdtls-lsp@claude-plugins-official --scope project
```

The kit's agents already list the `LSP` tool. The first start can exceed 120 s
while it indexes, and diagnostics arrive late, so the gate stays the real check.
