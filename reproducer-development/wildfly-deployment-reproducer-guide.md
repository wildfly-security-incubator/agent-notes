# WildFly Deployment Reproducer Guide

## Overview

This guide describes how to create a deployment-based reproducer project for diagnosing issues with WildFly. A reproducer is a minimal, self-contained Maven project that provisions a WildFly server, deploys an application, and demonstrates a specific behaviour.

The goal is a project that anyone can clone, build with `mvn clean package -DskipTests`, start, and verify with curl commands — no prior WildFly installation required.

**Living Document**: Update this guide as new patterns emerge from creating reproducers.

## Pre-Creation Checklist

Before writing code, establish a plan:

- [ ] **Understand the issue** — What behaviour are you reproducing? What's expected vs actual?
- [ ] **Identify the deployment structure** — Single WAR? Multiple WARs in an EAR? EJB module?
- [ ] **Identify required WildFly subsystems** — Which Galleon layers are needed?
- [ ] **Identify server configuration** — Does the server need CLI script customisation beyond defaults?
- [ ] **Create a plan document** — Capture the above in a markdown file before writing code. Agree on the approach before implementing.

## Project Conventions

- **groupId**: `org.wildfly.security.experimental` (generic enough to reuse across reproducers)
- **version**: `1.0.0.Alpha1-SNAPSHOT`
- **artifactId**: Pick a meaningful, descriptive name (e.g. `ee-security-multi-war-ear-experiment`)
- Each reproducer gets its own `README.md` with build instructions and curl-based verification examples

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

**Verify it builds**: Run `mvn package -DskipTests` in the generated project before making any changes. This confirms the baseline works and caches Maven dependencies.

**Review the generated pom.xml**: The archetype uses `discover-provisioning-info` (Glow) for auto-provisioning. This works well for single-WAR projects. For multi-module projects (e.g. EAR with multiple WARs), you will need to switch to explicit `feature-packs` and `layers` configuration.

### Step 2: Restructure (if needed)

For multi-module projects, convert the generated project:

1. Change the root `pom.xml` packaging to `pom` and add `<modules>`
2. Create sub-modules (WAR modules, EAR module)
3. Move the wildfly-maven-plugin configuration to the module that produces the deployable artifact (typically the EAR module)
4. Move dependency management and repository configuration to the parent pom
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
                <location>org.wildfly:wildfly-galleon-pack:${version.wildfly.bom}</location>
            </feature-pack>
        </feature-packs>
        <layers>
            <layer>ee-core-profile-server</layer>
            <!-- Add layers for the features you need -->
            <layer>core-tools</layer>
        </layers>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>package</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

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

⚠️ **`core-tools` is essential** — Without it, the provisioned server will not include `jboss-cli.sh` and the only way to stop the server is to kill the process. Always include this layer.

### Step 4: Server Configuration via CLI Scripts

If the server needs configuration beyond defaults (security domains, data sources, etc.), create a CLI script and reference it in the plugin config:

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

The CLI script runs via `embed-server` during provisioning. Each command should produce `{"outcome" => "success"}` — the build will fail if any command fails.

### Step 5: Server Lifecycle Scripts

Create `start-server.sh` and `stop-server.sh` scripts in the project root:

**`start-server.sh`** — Starts the server in the background and waits for startup:

```bash
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SERVER_HOME="$SCRIPT_DIR/ear/target/server"
LOG_FILE="$SERVER_HOME/standalone/log/server.log"

if [ ! -d "$SERVER_HOME" ]; then
    echo "Server not provisioned. Run 'mvn clean package -DskipTests' first."
    exit 1
fi

if ss -tlnp 2>/dev/null | grep -q ":8080 "; then
    echo "Port 8080 already in use — is the server already running?"
    exit 1
fi

rm -f "$LOG_FILE"

"$SERVER_HOME/bin/standalone.sh" > /dev/null 2>&1 &
SERVER_PID=$!
echo "Starting server (PID: $SERVER_PID)..."

for i in $(seq 1 60); do
    if grep -q "WFLYSRV0025" "$LOG_FILE" 2>/dev/null; then
        echo "Server started."
        exit 0
    fi
    if ! kill -0 $SERVER_PID 2>/dev/null; then
        echo "Server process died. Check $LOG_FILE"
        exit 1
    fi
    sleep 1
done

echo "Timed out after 60s. Check $LOG_FILE"
exit 1
```

**`stop-server.sh`** — Clean shutdown via JBoss CLI:

```bash
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SERVER_HOME="$SCRIPT_DIR/ear/target/server"

"$SERVER_HOME/bin/jboss-cli.sh" --connect --command=shutdown
```

Adjust `$SCRIPT_DIR/ear/target/server` to match your project's module layout (e.g. for a single WAR project, it would be `$SCRIPT_DIR/target/server`).

### Step 5.5: Test Client Script (Optional)

For reproducers with **multiple test scenarios** (3 or more curl commands), create a `test-client.sh` script that provides both interactive and direct execution modes.

#### When to Create a Test Client

✅ **Create** when:
- Testing multiple authentication mechanisms (BASIC, FORM, DIGEST)
- Testing multiple endpoints with different security configurations
- Testing both authenticated and unauthenticated requests
- Any scenario requiring 3+ curl commands to verify

❌ **Skip** when:
- Only 1-2 simple curl commands needed
- All tests are variations of the same endpoint/credentials

#### Implementation Pattern

The test client should support two modes:

**Interactive Mode** — User runs `./test-client.sh` with no arguments:
1. Display a numbered menu of all test scenarios with descriptions
2. Prompt for user input
3. Execute the selected test
4. Return to the menu (loop until 'q' or 'Q' is entered)

**Direct Mode** — User runs `./test-client.sh <N>`:
1. Validate that N is a valid test number
2. Execute test N immediately
3. Exit with the test's exit code (useful for scripting)

#### Implementation Details

Use Bash associative arrays to define tests:

```bash
#!/bin/bash

declare -A TESTS
declare -A DESCRIPTIONS

# Define test cases
TESTS[1]="curl -v http://localhost:8080/secured/secured"
DESCRIPTIONS[1]="war-secured without credentials (should return 401)"

TESTS[2]="curl -v -u testuser:password1! http://localhost:8080/secured/secured"
DESCRIPTIONS[2]="war-secured with BASIC credentials (should return 200)"

# ... more tests ...

# Function to display menu
show_menu() {
    echo ""
    echo "=== Your Reproducer Test Client ==="
    echo ""
    for i in $(seq 1 ${#TESTS[@]} | sort -n); do
        echo "  $i) ${DESCRIPTIONS[$i]}"
    done
    echo ""
    echo "  q) Quit"
    echo ""
}

# Function to execute a test
run_test() {
    local num=$1
    if [[ ! -v TESTS[$num] ]]; then
        echo "Error: Invalid test number: $num"
        return 1
    fi
    echo ""
    echo "=== Running Test $num ==="
    echo "Description: ${DESCRIPTIONS[$num]}"
    echo "Command: ${TESTS[$num]}"
    echo ""
    eval "${TESTS[$num]}"
    local exit_code=$?
    echo ""
    echo "=== Test $num completed (exit code: $exit_code) ==="
    return $exit_code
}

# Main logic: check if argument provided
if [[ $# -eq 1 ]]; then
    # Direct execution mode
    if [[ "$1" =~ ^[0-9]+$ ]]; then
        run_test "$1"
        exit $?
    else
        echo "Error: Argument must be a test number (1-${#TESTS[@]})"
        show_menu
        exit 1
    fi
else
    # Interactive mode
    while true; do
        show_menu
        read -p "Select test number (or q to quit): " choice
        case "$choice" in
            q|Q)
                echo "Exiting..."
                exit 0
                ;;
            [0-9]*)
                run_test "$choice"
                ;;
            *)
                echo "Invalid input. Please enter a number (1-${#TESTS[@]}) or 'q' to quit."
                ;;
        esac
    done
fi
```

**Key implementation points**:
- Use `eval` to execute the curl command stored in the TESTS array
- Capture and display the exit code so users can see test failures
- Show both the description and the actual command before execution (transparency)
- Validate input in both modes
- Make the script executable: `chmod +x test-client.sh`

#### README Integration

The README should retain the raw curl commands for documentation purposes, but add convenience shortcuts:

```markdown
Without credentials (should return 401):
```bash
curl -v http://localhost:8080/secured/secured
# Or: ./test-client.sh 1
```
```

This pattern:
- Documents the actual command for copy-paste or manual execution
- Shows the test client shortcut for convenience
- Helps users understand what each test does without running the script

### Step 6: Write the README

Every reproducer must include a `README.md` with:

1. Brief description of what the reproducer demonstrates
2. Prerequisites (JDK version, Maven version)
3. Build command: `mvn clean package -DskipTests`
4. Start command: `./start-server.sh`
5. Curl examples for verification
6. Expected vs actual results (table format)
7. How to check server logs for evidence
8. Stop command: `./stop-server.sh`

For simple reproducers with one or two test scenarios, curl examples can be run directly. For reproducers with multiple test scenarios (e.g., testing different authentication mechanisms), see **Step 5.5: Test Client Script** for detailed implementation guidance.

## Verification

After building and starting the server, verify by running curl commands individually against the running server. For reproducers with multiple test scenarios, use a `test-client.sh` script for convenience. Check:

- HTTP status codes match expectations
- Response bodies are correct
- Server logs show expected (or unexpected) behaviour

```bash
# Interactive testing with test client (if available)
./test-client.sh

# Or execute specific test directly
./test-client.sh 1

# Check server logs for specific evidence
grep "SomeRelevantMessage" ear/target/server/standalone/log/server.log
```

## Common Pitfalls

### ❌ Running `mvn package` without `clean` after config changes

The wildfly-maven-plugin detects an existing provisioned server in the target directory and skips reprovisioning. If you change CLI scripts, Galleon layers, or feature-pack configuration, **always** use `mvn clean package`.

### ❌ Missing `core-tools` layer

Without this layer, `jboss-cli.sh` is not provisioned. The server can be started but not cleanly stopped.

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

- [ ] Plan document created and agreed before implementation
- [ ] Project bootstrapped from archetype and baseline verified
- [ ] Restructured to required module layout (if multi-module)
- [ ] Galleon layers selected from WildFly Galleon Guide documentation
- [ ] `core-tools` layer included
- [ ] CLI scripts created for any server configuration
- [ ] `start-server.sh` and `stop-server.sh` scripts created
- [ ] `test-client.sh` created (if multiple test scenarios)
- [ ] `README.md` written with build/run/verify instructions
- [ ] Bug or behaviour verified with curl commands (or test-client.sh) and server log inspection
- [ ] `mvn clean package -DskipTests` builds from clean state

---

**Last Updated**: 2026-08-21
**Version**: 1.0
