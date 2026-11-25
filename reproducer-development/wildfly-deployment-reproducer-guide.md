# WildFly Deployment Reproducer Guide

## Overview

This guide describes how to create a deployment-based reproducer project for diagnosing issues with WildFly. A reproducer is a minimal, self-contained Maven project that provisions a WildFly server, deploys an application, and demonstrates a specific behaviour.

The goal is a small project the user can build, provision, start and explore in deliberate steps. It should help them understand the behaviour under investigation and provide a runnable example for a fix. Reproducer creation does not imply permission to operate the whole environment or build a regression-testing system.

**Living Document**: Update this guide as new patterns emerge from creating reproducers.

## Scope and Ownership

### Agree on the experience before implementation

Use the user's existing instructions to establish the purpose: a minimal bug demonstration, a hands-on learning environment, or explicitly requested automation. Resolve material gaps in scope before implementing; do not repeatedly ask about decisions the user has already made. A short plan in the README or existing notes is enough unless a separate planning document is useful or requested.

Default to a human-operated reproducer. The user should be able to build the applications, provision a server, start each dependency, start WildFly, deploy, send a request and inspect the result as separate actions. Small helpers may group one logical operation, such as starting one container. Do not add an all-in-one build/start/deploy/workload/capture/shutdown command by default.

Include only what exercises the requested behaviour. Before introducing another service or telemetry route, explain its role and why a simpler existing endpoint or service would not exercise the same integration. Different application APIs and exporters can expose different metrics even when they use similar endpoint names.

### Keep build ownership explicit

The reproducer builds its applications and provisions existing WildFly artifacts. **Building WildFly from source is a user activity**, performed in the user's separate WildFly checkout. The [development workflow guide](wildfly-development-workflow.md#building-wildfly-from-source-user-activity) may document those manual commands; that documentation is not authorization for an agent to execute them.

Provide a released WildFly default and a simple version-property override for a user-installed SNAPSHOT, using normal Maven settings and the local repository. Do not infer a need for source clones, reactor-build scripts, copied dependency repositories, frozen source-SHA descriptors or artifact-freezing frameworks. Automating a WildFly build is a separate task requiring explicit user instructions.

Use the latest LTS JDK locally unless the issue or a compatibility requirement calls for an older one; document that reason. An older bytecode release target does not by itself require an older local JDK. Prefer shell scripts or Java utilities launched through JBang. Introduce Python only when it is unavoidable and explain the need. Keep APIs, plugins and container images on appropriate released versions by default. Identify any server or component SNAPSHOT needed for the agreed experiment explicitly.

### Separate authoring checks from running the experiment

Permission to create or edit a reproducer is not permission to start services, deploy applications, generate traffic, execute scenarios or collect run evidence. Perform syntax and compilation checks within the authorized scope. Inspect Maven executions and helper scripts first: `-DskipTests` does not disable provisioning, embedded-server configuration, custom plugin executions or container startup.

Run the environment only when the user has asked for that execution. If execution is authorized, perform the agreed actions and report their limits; do not broaden them into a version matrix, stress test, recording framework or repeated acceptance run. Leave the user's live session and existing data alone unless changing them is authorized.

Scenario regression tests normally belong in the upstream fix's PR and test suite. Keep manual request examples or a small user-invoked workload helper in the reproducer. Add test runners, fixtures, assertions, snapshots, capture/reporting or automatic cleanup of an experiment only when requested. A successful build or a consumer-only fixture is not proof of WildFly runtime behaviour.

### Make the working environment visible

Put server homes and other resources the user will navigate in visible directories, such as `runtime/servers/`. Reserve hidden directories for tool metadata. Explain what Maven clean removes and what persists. Keep ordinary server and container logs accessible without introducing session archives by default.

Each container needs its own README, configuration and independent setup/start/logs/stop instructions. Document shared networks and their creation/removal explicitly when containers need to communicate. Starting or stopping one component must not silently operate on all the others.

## Pre-Creation Checklist

Before writing code, establish a plan:

- [ ] **Agree on scope and execution ownership** — Identify what the user wants to do manually, what helpers are useful, and what the agent may execute.
- [ ] **Understand the issue** — What behaviour are you reproducing? What's expected vs actual?
- [ ] **Identify the deployment structure** — Single WAR? Multiple WARs in an EAR? EJB module?
- [ ] **Identify required WildFly subsystems** — Which Galleon layers are needed?
- [ ] **Identify server configuration** — Does the server need CLI script customisation beyond defaults?
- [ ] **Record the plan** — Capture the agreed scope and separate manual steps in the README or existing project notes before implementation.

## Project Conventions

- **groupId**: `org.wildfly.security.experimental` (generic enough to reuse across reproducers)
- **version**: `1.0.0.Alpha1-SNAPSHOT`
- **artifactId**: Pick a meaningful, descriptive name that reflects the issue being reproduced (e.g. `security-integration-reproducer`, `multi-module-auth-test`)
- Each reproducer gets its own `README.md` with separate build/setup steps and manual request examples
- State the working directory for commands; use the project root by default and identify external checkouts explicitly

## Creation Process

### Step 1: Bootstrap from Archetype

Use the WildFly getting-started archetype to generate a baseline project with the wildfly-maven-plugin already configured:

```bash
mvn archetype:generate \
  -DarchetypeGroupId=org.wildfly.archetype \
  -DarchetypeArtifactId=wildfly-getting-started-archetype \
  -DarchetypeVersion=41.0.0.Final \
  -DgroupId=org.wildfly.security.experimental \
  -DartifactId=your-reproducer-name \
  -Dversion=1.0.0.Alpha1-SNAPSHOT \
  -DinteractiveMode=false
```

Replace the `archetypeVersion` with the WildFly version you are targeting.

**Review lifecycle bindings before executing a build**: Identify provisioning, embedded configuration, server start, deployment, integration-test and container executions in the generated POM. Choose syntax/compilation commands that stay within the authorized scope. A baseline package or runtime check is appropriate only when the user has authorized the operations it performs.

**Review the generated pom.xml**: The archetype uses `discover-provisioning-info` (Glow) for auto-provisioning. This works well for single-WAR projects. For multi-module projects (e.g. EAR with multiple WARs), you will need to switch to explicit `feature-packs` and `layers` configuration.

### Step 2: Restructure (if needed)

For multi-module projects, convert the generated project:

1. Change the root `pom.xml` packaging to `pom` and add `<modules>`
2. Create only the sub-modules the issue requires. Multiple independently deployable WARs do not require an EAR; add an EAR only when its deployment structure is part of the issue.
3. Give server provisioning an explicit owner, such as a dedicated server POM module for independently deployed WARs, and keep provisioning separate from application build and deployment by default.
4. Move shared dependency/plugin versions and the selected WildFly version property to the parent POM. Use normal Maven repositories; add special repository configuration only when the selected artifact requires it.
5. Remove the archetype's generated source files, test files, and webapp assets from the root

### Step 3: Select Galleon Layers

Configure the wildfly-maven-plugin with explicit feature packs and layers:

```xml
<plugin>
    <groupId>org.wildfly.plugins</groupId>
    <artifactId>wildfly-maven-plugin</artifactId>
    <version>${version.wildfly.maven.plugin}</version>
    <configuration>
        <feature-packs>
            <feature-pack>
                <location>org.wildfly:wildfly-galleon-pack:${wildfly.version}</location>
            </feature-pack>
        </feature-packs>
        <layers>
            <layer>ee-core-profile-server</layer>
            <!-- Add layers for the features you need -->
            <layer>core-tools</layer>
        </layers>
    </configuration>
</plugin>
```

Define `wildfly.version` in the parent POM with the released server version chosen for the project. Let users override it with `-Dwildfly.version=VERSION`. Keep plugin and application API versions independently pinned to released versions.

Invoke [provisioning](https://docs.wildfly.org/wildfly-maven-plugin/provision-mojo.html) explicitly, for example `mvn -pl server wildfly:provision` for a server module. Configure its provisioning directory to match the visible server home documented by the project. Explain any provisioning or embedded-configuration work bound to another lifecycle phase instead of hiding it inside an application build.

#### Finding the Right Layers

**Always consult the WildFly Galleon Guide** for the version you are targeting:

- https://docs.wildfly.org/41/Galleon_Guide.html

Replace `41` with your target WildFly version. This page lists all supported layers, their descriptions, and dependencies.

❌ Do not guess layer names or search through feature pack ZIP contents.

✅ Use the official documentation as the authoritative reference.

#### Commonly Needed Layers

| Layer | Purpose |
|-------|---------|
| `ee-core-profile-server` | Core Jakarta EE: Undertow (web), CDI, Naming, etc. |
| `ee-security` | Jakarta EE Security (Soteria) and JASPIC support |
| `core-tools` | `jboss-cli.sh`, `add-user.sh`, and `elytron-tool.sh` scripts |
| `ejb` | EJB support |
| `jpa` | JPA / Hibernate support |

Include `core-tools` when using the CLI examples below: it supplies `jboss-cli.sh` for management operations, including shutdown. A foreground server can also be stopped with Ctrl+C.

### Step 4: Server Configuration via CLI Scripts

If the server needs configuration beyond defaults (security domains, data sources, etc.), keep that configuration in a CLI script. For a separate `wildfly:provision` step, document a separate CLI invocation to apply it, including the target server home and whether it connects to a running server or uses `embed-server`.

For projects deliberately using the plugin's [wildfly:package](https://docs.wildfly.org/wildfly-maven-plugin/package-mojo.html) goal instead, the following configuration runs scripts during packaging. It does not apply to `wildfly:provision`:

```xml
<packaging-scripts>
    <packaging-script>
        <scripts>
            <script>configure-server.cli</script>
        </scripts>
    </packaging-script>
</packaging-scripts>
```

Place the CLI script file in the same module as the wildfly-maven-plugin configuration (e.g. `ear/configure-server.cli`).

With `packaging-scripts`, the plugin runs the CLI script using an embedded server. Explain that this executes server code during packaging, and keep it separate from application compilation. Check management operation outcomes and let failed configuration stop the setup step.

### Step 5: Server Lifecycle Scripts

Prefer foreground WildFly startup so the user can see the console and decide when to move on. The following examples assume provisioning has explicitly created `runtime/server/` at the project root; align this path with the plugin configuration. A visible `target/server/` is also suitable when its Maven-clean behaviour is understood.

**`start-server.sh`** — Starts only the provisioned server:

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
SERVER_HOME="$SCRIPT_DIR/runtime/server"
[[ -x $SERVER_HOME/bin/standalone.sh ]] || { echo 'Provision the server first' >&2; exit 1; }
export JBOSS_HOME="$SERVER_HOME"
exec "$SERVER_HOME/bin/standalone.sh" -b=127.0.0.1 -bmanagement=127.0.0.1
```

The user watches the startup messages in this terminal. In another terminal they can deploy an application or send a request when ready. Do not delete logs, suppress console output, run a workload or shut the server down as part of this command.

**`stop-server.sh`** — A separate clean shutdown action:

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
export JBOSS_HOME="$SCRIPT_DIR/runtime/server"
"$JBOSS_HOME/bin/jboss-cli.sh" --connect --command=shutdown
```

Document the selected management address and any authentication requirements. If the user may have several servers/checkouts, verify the management target's server home before sending a mutating operation. Background startup and readiness polling are options for explicitly requested automation, not the default manual workflow.

### Step 5.5: Manual Request or Workload Helpers (Optional)

Start with visible curl or CLI commands that the user can run individually. The number of commands is not a reason to create a test framework.

A small shell or JBang helper can make a repeated action convenient when the user wants it. Keep its action explicit: send the selected request or bounded workload, print the response, then return control. Keep the underlying curl/API example in the README so the user can understand what the helper does.

Do not automatically turn manual examples into a menu of scenarios, a pass/fail harness, expected-value assertions, a regression matrix, evidence capture or automatic teardown. Scenario regression tests belong with the upstream fix unless the user has explicitly requested tests in this project. Ordinary client error handling and checks that prevent operating on another server remain appropriate.

### Step 6: Write the README

The README should introduce the reproducer before its implementation details:

1. Explain its purpose, the behaviour being explored and the applications involved. Introduce unfamiliar concepts and vocabulary before using them.
2. Describe the components and why each is needed. For a multi-stage flow, explain where data is generated, collected, routed and consumed, using a small diagram when useful.
3. State prerequisites, version selection and who supplies any SNAPSHOT artifacts.
4. Present application build, server provisioning/configuration, shared-network setup, each container's startup, WildFly startup, deployment, workload and shutdown as separate user-invoked steps. Use one logical operation per command block, with room to pause and inspect it.
5. Explain what each step creates or changes, where files live and which terminal or UI to use. Link each container's own setup guide, including its image, configuration, host/container ports, network names, data lifetime and independent lifecycle commands.
6. Provide manual request examples, live log views and graph/query instructions. Say exactly where a query goes, explain its syntax and how it relates to the application's measurements. Avoid unexplained instructions such as "enter this in Prometheus".
7. Explain the reported behaviour and relevant observation points without presenting predictions as measured results. Include existing observations when useful; do not require the agent to generate a results report.
8. Document separate stop/restart actions and what happens to application state, processor state and stored history. Preserve user data unless its reset was requested.
9. Document debugging and version/component overrides only to the extent the task needs them. A simple release/SNAPSHOT property switch is the normal starting point; advanced channel/manifests are opt-in.

Explain how users can observe readiness in live output. Do not hide the whole workflow behind a single script or combine all commands into one copy-and-run block.

#### README Remote Debugging Section

When debugging is relevant and the launcher implements these flags, document `--debug` and `--suspend` in the README:

````markdown
### Remote Debugging

The `start-server.sh` script supports remote debugging with optional flags:

**Normal debug mode** (server starts immediately, debugger can attach anytime):
```bash
./start-server.sh --debug
```

**Suspend mode** (server pauses at boot until debugger attaches):
```bash
./start-server.sh --debug --suspend
```

Both modes listen on port `5005`. Attach your IDE debugger to `localhost:5005`.

**When to use `--suspend`**: [Describe which subsystems or initialization logic need boot-time debugging for this reproducer]. Use `--suspend` to catch:
- [List relevant initialization points - e.g., CDI portable extensions, ServletContainerInitializers, subsystem boot]

**Debugging [topic] initialization**: Set breakpoints in:
- `[package.ClassName]` — [what this class does in the initialization flow]
- `[package.ClassName]` — [what this class does in the initialization flow]
````

**Guidance for customization**:
- Replace `[Describe which subsystems...]` with the specific subsystems relevant to your reproducer (e.g., "Security subsystems (Elytron, Jakarta EE Security)" or "Messaging subsystems" or "Persistence initialization")
- Replace the bullet list with initialization points relevant to what you're debugging
- Replace the debugging targets section with actual class names and packages from the components being debugged
- If boot-time debugging isn't relevant to your reproducer, you can omit the "When to use `--suspend`" paragraph and just document the two modes

## Verification

For agent authoring work, check syntax, local documentation links and compilation within the user's authorized scope. Inspect lifecycle bindings before invoking Maven; a package goal can do more than compile, and `-DskipTests` is not a blanket runtime-execution guard.

The README should give the user individual requests and live views for exploring behaviour. Starting dependencies or WildFly, deploying, generating requests, running scenarios, taking snapshots and capturing evidence require authorization for those actions. Do not run the user's experiment merely to mark the reproducer complete.

If runtime verification is requested, agree on the necessary checks, execute only that scope and describe what was actually observed. Keep upstream regression test work separate from the manual reproducer.

## External Dependencies (Keycloak, Databases, etc.)

Some reproducers require external services like Keycloak (for OIDC), databases, or message brokers. First establish that each service is needed for the selected code path. Give each container its own directory, README, configuration and setup/lifecycle commands under `containers/`. Starting one container must not start its neighbours or WildFly. Document shared-network creation/removal separately when containers need to communicate, including host-published ports versus container-network addresses.

### Container Naming Convention

**❌ Wrong** — Generic container names conflict across projects:
```bash
CONTAINER_NAME="keycloak"
VOLUME_NAME="keycloak-data"
```

If you work on multiple reproducers that use Keycloak, starting one will conflict with or stop containers from another project.

**✅ Correct** — Use project-specific suffixes:
```bash
CONTAINER_NAME="keycloak-my-reproducer"
VOLUME_NAME="keycloak-data-my-reproducer"
```

Replace `my-reproducer` with your project's artifactId or a short identifier. This allows multiple Keycloak instances to coexist.

### Container Directory Structure

```
containers/
└── keycloak/
    ├── README.md                    # Container-specific documentation
    ├── start-keycloak.sh            # Idempotent start script
    ├── stop-keycloak.sh             # Clean shutdown script
    └── setup-realm.sh               # Configure Keycloak (create realms, clients, users)
```

### Container Start Script Pattern

Make start scripts idempotent (safe to run when already running):

```bash
#!/bin/bash
set -euo pipefail

CONTAINER_NAME="keycloak-my-reproducer"
IMAGE="quay.io/keycloak/keycloak:VERSION"  # Replace VERSION with the chosen released tag
VOLUME_NAME="keycloak-data-my-reproducer"
HOST_PORT="8180"              # Avoid conflict with WildFly on 8080
CONTAINER_PORT="8080"

# Create volume if it doesn't exist
if ! podman volume inspect "$VOLUME_NAME" &>/dev/null; then
    echo "Creating Podman volume '$VOLUME_NAME'..."
    podman volume create "$VOLUME_NAME"
fi

# Handle existing container
if podman container exists "$CONTAINER_NAME" 2>/dev/null; then
    STATUS=$(podman inspect --format '{{.State.Status}}' "$CONTAINER_NAME")
    if [ "$STATUS" = "running" ]; then
        echo "Keycloak is already running: http://localhost:$HOST_PORT"
        exit 0
    else
        echo "Starting existing container..."
        podman start "$CONTAINER_NAME"
        echo "Keycloak started: http://localhost:$HOST_PORT"
        exit 0
    fi
fi

echo "Starting Keycloak ${IMAGE}..."
podman run -d \
    --name "$CONTAINER_NAME" \
    -p "127.0.0.1:${HOST_PORT}:${CONTAINER_PORT}" \
    -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
    -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
    -v "${VOLUME_NAME}:/opt/keycloak/data:Z" \
    "$IMAGE" \
    start-dev

echo "Keycloak starting on http://localhost:$HOST_PORT"
echo "Startup takes 15-30 seconds. Watch progress with:"
echo "  podman logs -f $CONTAINER_NAME"
echo ""
echo "Wait until you see Keycloak startup complete messages, then press Ctrl+C"
```

### Container Stop Script Pattern

```bash
#!/bin/bash
set -euo pipefail

CONTAINER_NAME="keycloak-my-reproducer"

if podman container exists "$CONTAINER_NAME" 2>/dev/null; then
    echo "Stopping $CONTAINER_NAME..."
    podman stop "$CONTAINER_NAME"
    echo "Container stopped (not removed — data persists in volume)"
else
    echo "Container $CONTAINER_NAME not found"
fi
```

### Port Mapping Strategy

Map external services to non-conflicting ports:

| Service | Default Port | Mapped Host Port | Reason |
|---------|-------------|------------------|--------|
| WildFly | 8080 | 8080 | Primary application server |
| Keycloak | 8080 | 8180 | Avoids conflict with WildFly |
| PostgreSQL | 5432 | 5432 | Standard (usually no conflict) |
| Management console | 9990 | 9990 | WildFly management |

### README Documentation for External Dependencies

The main README should explain the topology and link each container's setup guide. Each guide should cover:

- The component's role, selected released image and why this integration is used.
- Configuration files and how they are mounted or loaded.
- Prerequisites and explicit shared-network setup, if needed.
- Host loopback ports, container ports and container-name addressing.
- Separate start, live-logs, status and stop commands for that component only.
- Where the user goes next: a browser URL/query input, another terminal or the next component's guide.
- Whether configuration and data persist across stop/restart/removal, and how the user can deliberately reset them.

Whenever a step requires waiting for startup, show how the user can watch it, for example `podman logs -f keycloak-my-reproducer`. Explain that Ctrl+C ends a log-following command while the container continues running. A foreground WildFly terminal already supplies its live console.

Do not make a combined "build and start everything" section, perform an implicit data reset, or automatically collect logs into a run report. Ordinary live server/container logs are useful observation tools and should remain accessible.

## Common Pitfalls

### ❌ Telling users to "wait" without providing a progress check

Never tell users to "wait 15-30 seconds" without giving them a way to monitor progress.

**❌ Wrong**:
```markdown
Start Keycloak and wait 30 seconds for it to be ready.
```

**✅ Correct**:
````markdown
Start Keycloak:
```bash
./containers/keycloak/start-keycloak.sh
```

Watch startup progress:
```bash
podman logs -f keycloak-my-reproducer
```

Wait until you see startup complete messages, then press `Ctrl+C` to stop following logs.
````

This pattern applies to:
- Container startup (use `podman logs -f` or `docker logs -f`)
- WildFly server startup (use `tail -f standalone/log/server.log` or grep for `WFLYSRV0025`)
- Database initialization (use container logs or connection test commands)
- Any service that takes >5 seconds to become ready

### ❌ Using generic container names for external dependencies

Container names like `keycloak` or `postgres` conflict when working on multiple reproducers. Always use project-specific names (e.g., `keycloak-my-reproducer`). See **External Dependencies** section above for the complete pattern.

### ❌ Reusing a stale provisioned server after config changes

A previously provisioned home may not reflect changed CLI scripts, layers or feature-pack selection. Document how the user stops and explicitly reprovisions that home. Use `clean` when it removes the generated server for that project layout; a home under `runtime/` survives Maven clean and needs its own deliberate replacement. Do not erase a running or user-modified home automatically.

### ❌ Missing `core-tools` layer

Without this layer, `jboss-cli.sh` is not provisioned and the documented CLI operations are unavailable. Foreground shutdown with Ctrl+C remains available.

### ❌ Port conflicts from orphaned servers

If a previous server was not shut down, the next start will fail with `Address already in use`. Check with `ss -tlnp | grep 8080` and shut down the old server with `./stop-server.sh` or `jboss-cli.sh --connect --command=shutdown`.

### ❌ Guessing Maven plugin/dependency versions

Always verify versions exist in Maven Central before using them. Use `mcs search` (if available) or check Maven Central directly. A failed resolution gets cached in the local Maven repo and blocks subsequent attempts until the cache entry expires.

## Plugin/Dependency Version Lookup

To verify a version exists before adding it to a pom:

```bash
mcs search "org.apache.maven.plugins:maven-ear-plugin"
```

This shows all available versions with release dates, preventing build failures from non-existent versions.

## Summary Checklist

- [ ] Purpose, execution ownership and implementation scope established using the user's existing instructions
- [ ] Minimal topology chosen; each service and API/export path justified
- [ ] Application structure matches the issue; an EAR is used only when needed
- [ ] Galleon layers selected from the target version's documentation, including `core-tools`
- [ ] Released server default and simple version override documented; WildFly source builds remain an explicit user activity elsewhere
- [ ] Latest LTS JDK used locally, or a concrete reason for an older JDK recorded
- [ ] Helpers use shell or Java/JBang; any unavoidable Python dependency is explained
- [ ] User-facing generated resources live in visible directories and Maven-clean behaviour is documented
- [ ] Build, provision, start, deploy, workload and stop steps are independent and explained
- [ ] WildFly console and normal container logs are accessible; foreground startup is the default
- [ ] Each container has its own guide, configuration and lifecycle commands, with explicit networking and data-lifetime information
- [ ] README introduces the reproducer and vocabulary before naming applications or presenting queries
- [ ] Manual request/workload examples remain user-controlled; test suites and capture/reporting are added only when requested
- [ ] Syntax, links and compilation checked within scope; any runtime checks reported only if authorized and performed
- [ ] Debugging and advanced component overrides documented only when relevant


---

**Last Updated**: 2026-11-25
**Version**: 1.3
