# WildFly Development Workflow

## Overview

This guide describes development workflows for working with WildFly and its components (e.g. Soteria, Elytron) in a reproducer project. It covers remote debugging, SNAPSHOT builds, and version overrides via channels/manifests.

Follow [Scope and Ownership](wildfly-deployment-reproducer-guide.md#scope-and-ownership) first. These are user-invoked workflows, not an instruction for an agent to run every procedure or add every feature to a reproducer. Start with a released server and a simple version override. Configure debugging or advanced component channels only when the task needs them. Use the latest LTS JDK locally unless a documented issue or compatibility requirement calls for an older one.

**Living Document**: Update this guide as new patterns emerge from WildFly development work.

## Remote Debugging

### Starting WildFly with Remote Debugging

To debug WildFly, start the server with JDWP (Java Debug Wire Protocol) enabled. The following startup examples assume a provisioned `runtime/server/` at the project root; adapt the path to the project layout.

#### Debug Normally Running Server

Add the debug arguments to `standalone.sh` via `JAVA_OPTS`:

```bash
export JAVA_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=127.0.0.1:5005"
./runtime/server/bin/standalone.sh
```

- `transport=dt_socket` — Use socket transport for the debugger connection
- `server=y` — WildFly listens for debugger connections (doesn't initiate)
- `suspend=n` — Server starts immediately (doesn't wait for debugger)
- `address=127.0.0.1:5005` — Listen on loopback, port 5005

#### Debug Boot Process (Suspend Until Debugger Attaches)

When debugging security initialization or other boot-time behaviour, use `suspend=y`:

```bash
export JAVA_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=127.0.0.1:5005"
./runtime/server/bin/standalone.sh
```

The server will **pause immediately** and wait for a debugger to attach before continuing the boot process.

**Why `suspend=y` matters**: Security subsystems (Elytron, Soteria/Jakarta EE Security) initialize during server boot. By the time the server is fully started, initialization has already completed. To debug this initialization, the server must pause before it begins.

#### Attaching the Debugger

In your IDE (IntelliJ, Eclipse, VS Code):

1. Create a "Remote JVM Debug" configuration
2. Set host to `localhost` (or the server's hostname if remote)
3. Set port to `5005`
4. Set breakpoints in the code you want to debug
5. Click "Debug" to attach

The server will resume (if `suspend=y` was used) once the debugger connects.

#### Scripting Remote Debug Startup

A small foreground launcher can expose optional debugger flags without starting other components. This example assumes provisioning creates `runtime/server/` at the project root; keep the path consistent with the project's provisioning configuration.

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
SERVER_HOME="$SCRIPT_DIR/runtime/server"
DEBUG_MODE=no
SUSPEND_MODE=n
while [[ $# -gt 0 ]]; do
    case $1 in
        --debug) DEBUG_MODE=yes ;;
        --suspend) SUSPEND_MODE=y ;;
        *) echo "Usage: $0 [--debug] [--suspend]" >&2; exit 2 ;;
    esac
    shift
done
[[ -x $SERVER_HOME/bin/standalone.sh ]] || { echo 'Provision the server first' >&2; exit 1; }
[[ $SUSPEND_MODE == n || $DEBUG_MODE == yes ]] || { echo '--suspend requires --debug' >&2; exit 2; }
if [[ $DEBUG_MODE == yes ]]; then
    export JAVA_OPTS="${JAVA_OPTS:-} -agentlib:jdwp=transport=dt_socket,server=y,suspend=$SUSPEND_MODE,address=127.0.0.1:5005"
fi
export JBOSS_HOME="$SERVER_HOME"
exec "$SERVER_HOME/bin/standalone.sh" -b=127.0.0.1 -bmanagement=127.0.0.1
```

The user follows the console and attaches the debugger when ready. Keep live logs and the server process available for exploration; do not delete logs, launch requests, collect a report or tear down the session from this launcher.

**Usage**:

```bash
# Normal debug mode (server starts immediately)
./start-server.sh --debug

# Debug boot process (server pauses until debugger attaches)
./start-server.sh --debug --suspend
```

## Working with WildFly SNAPSHOT Builds

### Why Use SNAPSHOT Builds

When testing fixes or features that have been merged to WildFly main but not yet released, you need to provision a server from a SNAPSHOT build.

### Switching to a SNAPSHOT Build

Define one server-version property with a released default, and reference it from provisioning:

```xml
<feature-packs>
    <feature-pack>
        <location>org.wildfly:wildfly-galleon-pack:${wildfly.version}</location>
    </feature-pack>
</feature-packs>
```

The user can select a different version without editing the POM:

```bash
mvn -pl server wildfly:provision -Dwildfly.version=VERSION
```

Here `server` is an example provisioning module and `VERSION` must be replaced with the desired release or SNAPSHOT coordinate. Reuse an existing suitable property name when the project already has one. Keep application API and plugin versions independent of the chosen server version.

Use the normal Maven settings and local repository containing the user's installed artifacts. SNAPSHOT availability depends on the selected version and configured repositories; do not assume all snapshots are published or that none are. Configure additional repositories only when they are actually required.

Explain how to stop and reprovision a server or select another visible home when changing versions. `mvn clean` only removes paths managed by that lifecycle; an independently provisioned `runtime/` home survives it. Do not silently patch a running server, switch its version, create a source-SHA matrix or copy whole Maven repositories as part of a simple version override.

### Building WildFly from Source: User Activity

**This is a user activity in a separate WildFly checkout, not an automated reproducer step.** Keep these instructions available when the user needs an unpublished fix, but do not have an agent, provisioning script, Maven execution or test harness clone and build WildFly as part of creating or checking the reproducer. Any separately requested build automation is a different task with explicit authorization.

The user selects the branch or revision in their own checkout, then performs the build there. For example, these are commands for the user to adapt and run:

```bash
cd /absolute/path/to/your/wildfly-checkout
mvn clean install -DskipTests -Dquickly
```

Follow that checkout's build requirements. This preparation installs artifacts into the user's Maven repository; the reproducer subsequently consumes the selected feature-pack coordinates. The reproducer should not own the source tree, orchestrate the reactor or copy its repository into the reproducer directory.

## Overriding Component Versions with Channels

This is an advanced, opt-in workflow for a component-only experiment. A whole-server release/SNAPSHOT version switch does not by itself require channel profiles or custom manifests. Read this section when the task actually needs component replacement; do not add its example files or modes to every reproducer.

### Why Use Channels

WildFly uses the Galleon Channels feature to manage component versions. Channels allow you to override the version of a specific component (e.g. Soteria, Elytron) without rebuilding all of WildFly.

**Use case**: Testing a fix in Soteria 4.0.3-SNAPSHOT without building a full WildFly SNAPSHOT.

**Reference**: https://docs.wildfly.org/wildfly-maven-plugin/provision-mojo.html

### Channel Configuration in wildfly-maven-plugin

Add the `<channels>` element to your wildfly-maven-plugin configuration:

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
        <channels>
            <channel>
                <manifest>
                    <groupId>org.wildfly.channels</groupId>
                    <artifactId>wildfly-ee</artifactId>
                    <version>${version.wildfly.bom}</version>
                </manifest>
            </channel>
            <channel>
                <manifest>
                    <url>file://${project.basedir}/soteria-override.yaml</url>
                </manifest>
            </channel>
        </channels>
        <layers>
            <layer>ee-core-profile-server</layer>
            <layer>ee-security</layer>
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

The first channel points to the standard WildFly manifest. The second channel points to a custom manifest file that overrides specific components.

### Manifest Reference Options

Manifests can be referenced in three ways:

#### 1. Maven Coordinates

```xml
<channel>
    <manifest>
        <groupId>org.wildfly.channels</groupId>
        <artifactId>wildfly-ee</artifactId>
        <version>${version.wildfly.bom}</version>  <!-- version is optional -->
    </manifest>
</channel>
```

**When to use**: The manifest is published to a Maven repository (e.g. Maven Central, JBoss Nexus).

#### 2. URL Reference

```xml
<channel>
    <manifest>
        <url>https://example.org/manifests/custom-manifest.yaml</url>
    </manifest>
</channel>
```

**Supported URL schemes**: `https://`, `http://`, `file://`

**When to use**: 
- Remote manifests not in Maven repos
- Local file-based manifests (development/testing)

#### 3. Local File Reference

```xml
<channel>
    <manifest>
        <url>file://${project.basedir}/soteria-override.yaml</url>
    </manifest>
</channel>
```

**When to use**: Custom manifests stored in the project repository. This is the recommended approach for reproducers.

#### 4. Command-Line Override

```bash
mvn clean package \
  -Dwildfly.channels="file://$(pwd)/soteria-override.yaml"
```

Or with multiple channels (comma-delimited):

```bash
mvn clean package \
  -Dwildfly.channels="org.wildfly.channels:wildfly-ee:41.0.0.Final,file://$(pwd)/soteria-override.yaml"
```

**When to use**: Temporary overrides without modifying the pom.xml.

### Creating a Custom Manifest

Create a YAML file (e.g. `ear/soteria-override.yaml`) that specifies the component override:

```yaml
schemaVersion: "1.1.0"
id: "soteria-override"
name: "Soteria Override"
description: "Override Soteria to 4.0.3-SNAPSHOT for testing"

streams:
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria"
    version: "4.0.3-SNAPSHOT"
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria.spi.bean.decorator.weld"
    version: "4.0.3-SNAPSHOT"
```

**Manifest Format**:
- `schemaVersion`: Use `"1.1.0"` (current schema version used by WildFly and RESTEasy manifests)
- `id`: Unique identifier for this manifest (optional but recommended)
- `name`: Human-readable name for the manifest (used in logs)
- `description`: What this manifest provides (documentation)
- `streams`: Array of Maven artifact streams to override
  - Each stream defines `groupId`, `artifactId`, and `version`
  - **Only include artifacts you want to override** — not the full component set

### Understanding Component Artifact Groups

**Important**: A single component version in WildFly typically spans **multiple Maven artifacts**. When overriding a component, you need to analyze which artifacts belong together and decide which ones to override.

**Analysis Process**:

1. **Find the artifacts** — Check the provisioned server's module structure:
   ```bash
   find ear/target/server/modules -name "*soteria*" -o -name "*component-name*"
   ls -lh ear/target/server/modules/system/layers/base/org/glassfish/soteria/main/
   ```

2. **Check the module.xml** — This reveals all artifacts in the component:
   ```bash
   cat ear/target/server/modules/system/layers/base/org/glassfish/soteria/main/module.xml
   ```
   
   Example output:
   ```xml
   <resources>
       <resource-root path="soteria-4.0.2.jar"/>
       <resource-root path="soteria.spi.bean.decorator.weld-4.0.2.jar"/>
   </resources>
   ```

3. **Derive the artifact coordinates** — The JAR filenames map to `artifactId-version.jar`:
   - `soteria-4.0.2.jar` → `org.glassfish.soteria:soteria:4.0.2`
   - `soteria.spi.bean.decorator.weld-4.0.2.jar` → `org.glassfish.soteria:soteria.spi.bean.decorator.weld:4.0.2`

4. **Decide the override set**:
   - **Full component override**: List all related artifacts (most common for testing a component fix)
   - **Partial override**: Override only specific artifacts (less common, use when testing a fix to one submodule)

**Soteria Example**: Soteria has 2 artifacts. To test a Soteria SNAPSHOT, override both:
```yaml
streams:
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria"
    version: "4.0.3-SNAPSHOT"
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria.spi.bean.decorator.weld"
    version: "4.0.3-SNAPSHOT"
```

**Elytron Example**: WildFly Elytron has many artifacts (`wildfly-elytron-http`, `wildfly-elytron-realm`, etc.). You can override all of them to a SNAPSHOT version, or just the ones relevant to your test case. Analyze first, then decide.

**Why this matters**: If you only override some artifacts in a component group, you may end up with version mismatches that cause runtime errors or hide bugs. Always be intentional about which artifacts you include

**How Channel Layering Works** (verified with WildFly + RESTEasy example):

- WildFly EE manifest: 668 streams (all WildFly dependencies)
- RESTEasy manifest: ~20 streams (just RESTEasy components)
- When both channels are configured, streams with matching `groupId:artifactId` from later channels override earlier ones
- Your custom manifest only needs the specific streams you want to change

**Real-World Example**: The [RESTEasy 6.2 manifest](https://repo1.maven.org/maven2/dev/resteasy/channels/resteasy-6.2/6.2.14.Final/resteasy-6.2-6.2.14.Final-manifest.yaml) demonstrates this pattern — it defines only RESTEasy artifacts, layered on top of the WildFly manifest.

**Multiple component overrides** — Add additional entries to the `streams` array:

```yaml
schemaVersion: "1.1.0"
id: "multi-component-override"
name: "Multi-Component Override"
description: "Override Soteria and Elytron for testing"

streams:
  # Soteria artifacts
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria"
    version: "4.0.3-SNAPSHOT"
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria.spi.bean.decorator.weld"
    version: "4.0.3-SNAPSHOT"
  # Elytron artifacts (example - analyze to determine the full set)
  - groupId: "org.wildfly.security"
    artifactId: "wildfly-elytron-http"
    version: "3.0.0.Beta1-SNAPSHOT"
  - groupId: "org.wildfly.security"
    artifactId: "wildfly-elytron-realm"
    version: "3.0.0.Beta1-SNAPSHOT"
```

**How it works**:

- The `streams` section defines Maven coordinates and the version to use
- When Galleon provisions the server, it checks all configured channels
- Channels are processed in order — later channels can override earlier ones
- Your custom manifest only needs to define the specific streams you want to override, not the entire component set
- The WildFly channel provides the defaults; your channel overrides just Soteria

**Repository requirement**: The SNAPSHOT artifact must be available in one of your configured `<repositories>`. If Soteria 4.0.3-SNAPSHOT is only in your local `~/.m2`, that's sufficient. If it's in a remote snapshot repository, ensure that repository is configured in your pom.

**Published Manifest Artifact Convention**: When manifests are published to Maven repositories (like the WildFly and RESTEasy manifests), they use a `-manifest.yaml` classifier:

```
groupId:artifactId:version:yaml:manifest

Example:
  org.wildfly.channels:wildfly-ee:41.0.0.Final:yaml:manifest
  dev.resteasy.channels:resteasy-6.2:6.2.14.Final:yaml:manifest

File naming pattern:
  artifactId-version-manifest.yaml
```

This is automatic when using Maven coordinates in the `<manifest>` element — Galleon resolves the `-manifest.yaml` artifact from the repository.

### Verifying the Override

After building with the channel override, check which version was provisioned:

```bash
# Check Soteria JARs
ls -lh ear/target/server/modules/system/layers/base/org/glassfish/soteria/main/*.jar

# Or search for any component
find ear/target/server/modules -name "*soteria*.jar"
```

You should see the SNAPSHOT version you specified in the JAR filenames.

### Verified Pattern

**Channel layering with partial manifests is the correct approach**. This is proven by the RESTEasy channel pattern used in WildFly itself:

- WildFly EE 41.0.0.Final manifest contains 668 streams (all components)
- WildFly EE already includes RESTEasy streams with specific versions
- RESTEasy 6.2 manifest contains only ~20 streams (just RESTEasy components)
- When both channels are configured, RESTEasy streams override the WildFly ones
- No need to duplicate the full WildFly manifest — only define what you want to override

This pattern is documented in the [WildFly Maven Plugin channel examples](https://docs.wildfly.org/wildfly-maven-plugin/channel-example.html).

## Generic Component Override Pattern (Three-Tier Usage)

When a component-override workflow has been requested, the following pattern can offer three usage modes without POM edits. Add only the modes and manifest files the user needs; this is not a baseline requirement for reproducers or for a whole-server version override.

### Recommended Pattern: Property-Activated Profile with Default Manifest

Use a Maven profile activated by one property (`component.override`) and parameterized by another (`override.manifest`) to enable flexible override modes.

**Step 1: Add properties** (in the module with wildfly-maven-plugin):

```xml
<properties>
    <!-- Default manifest file for component override mode -->
    <override.manifest>component-override.yaml</override.manifest>
</properties>
```

**Step 2: Default plugin configuration**:

```xml
<plugin>
    <groupId>org.wildfly.plugins</groupId>
    <artifactId>wildfly-maven-plugin</artifactId>
    <configuration>
        <feature-packs>
            <feature-pack>
                <location>org.wildfly:wildfly-galleon-pack:${version.wildfly.bom}</location>
            </feature-pack>
        </feature-packs>
        <channels>
            <channel>
                <manifest>
                    <groupId>org.wildfly.channels</groupId>
                    <artifactId>wildfly-ee</artifactId>
                    <version>${version.wildfly.bom}</version>
                </manifest>
            </channel>
        </channels>
        <layers>
            <layer>ee-core-profile-server</layer>
            <layer>ee-security</layer>
            <layer>core-tools</layer>
        </layers>
        <!-- packaging-scripts, etc. -->
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

**Step 3: Add profile for override channel**:

```xml
<profiles>
    <profile>
        <id>component-override</id>
        <activation>
            <property>
                <name>component.override</name>
            </property>
        </activation>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.wildfly.plugins</groupId>
                    <artifactId>wildfly-maven-plugin</artifactId>
                    <configuration>
                        <channels>
                            <channel>
                                <manifest>
                                    <groupId>org.wildfly.channels</groupId>
                                    <artifactId>wildfly-ee</artifactId>
                                    <version>${version.wildfly.bom}</version>
                                </manifest>
                            </channel>
                            <channel>
                                <manifest>
                                    <url>file://${project.basedir}/${override.manifest}</url>
                                </manifest>
                            </channel>
                        </channels>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

**Key points**:
- Profile is activated by `-Dcomponent.override` (NOT by `override.manifest`)
- Profile uses `${override.manifest}` property for the filename
- Property has a default value (`component-override.yaml`)
- User can override the filename with `-Doverride.manifest=<custom>.yaml`

**Step 4: Create manifest files**:

Create `component-override.yaml` (default SNAPSHOT override):

```yaml
schemaVersion: "1.1.0"
id: "soteria-override"
name: "Soteria SNAPSHOT Override"
description: "Override Soteria with 4.0.3-SNAPSHOT for development testing"

streams:
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria"
    version: "4.0.3-SNAPSHOT"
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria.spi.bean.decorator.weld"
    version: "4.0.3-SNAPSHOT"
```

Create `elytron-snapshot.yaml` (example alternate):

```yaml
schemaVersion: "1.1.0"
id: "elytron-override"
name: "WildFly Elytron SNAPSHOT Override"
description: "Override WildFly Elytron components with SNAPSHOT versions for development testing"

streams:
  - groupId: "org.wildfly.security"
    artifactId: "wildfly-elytron-auth"
    version: "2.6.0.Final-SNAPSHOT"
  - groupId: "org.wildfly.security"
    artifactId: "wildfly-elytron-http"
    version: "2.6.0.Final-SNAPSHOT"
  # Add more Elytron artifacts as needed
```

### Three-Tier Usage

**Mode 1: Default (No Override)**

```bash
mvn clean package
```

Uses only WildFly EE channel → release versions (e.g., Soteria 4.0.2, Elytron 2.6.0.Final).

**Mode 2: Simple Override (Standard SNAPSHOT)**

```bash
mvn clean package -Dcomponent.override
```

Activates `component-override` profile → uses `component-override.yaml` → provisions Soteria 4.0.3-SNAPSHOT.

**Mode 3: Advanced Override (Custom Manifest)**

```bash
# Test with Elytron SNAPSHOT
mvn clean package -Dcomponent.override -Doverride.manifest=elytron-snapshot.yaml

# Test with Soteria SNAPSHOT (explicitly)
mvn clean package -Dcomponent.override -Doverride.manifest=soteria-snapshot.yaml
```

Activates profile, uses custom manifest filename.

### Benefits

- **Generic**: Not tied to a specific component (Soteria, Elytron, etc.)
- **Simple common case**: `-Dcomponent.override` is easy to remember
- **Flexible**: Can maintain multiple manifests and switch between them
- **Standard naming**: `component-override.yaml` is the conventional default
- **No no-op manifest needed**: Profile only activates when requested
- **No pom edits**: All three modes work via command-line properties

### Why This Pattern

Earlier attempts:
1. **Property-based with no-op manifest**: Used `-Doverride.manifest=no-override.yaml` with `streams: []`. This fails validation because the manifest schema requires at least 1 stream.
2. **Property-based filename-only**: Activated by `-Doverride.manifest=<filename>`, but this couples the activation to the filename, making "simple override" awkward (you have to type the filename every time).

This pattern:
- Separates activation (`component.override`) from parameterization (`override.manifest`)
- Default manifest filename lives in a property (overridable)
- Profile only activates when explicitly requested (no channel overhead in default builds)

## Common Pitfalls

### ❌ Wrong Channel Configuration Parameter Name

The wildfly-maven-plugin uses `<channels>`, not `<overrideFeatures>`.

**Incorrect** (silently ignored with a warning):
```xml
<configuration>
    <overrideFeatures>
        <overrideFeature>
            <manifest>${elytron.override.manifest}</manifest>
        </overrideFeature>
    </overrideFeatures>
</configuration>
```

Maven output will show:
```
[WARNING] Parameter 'overrideFeatures' is unknown for plugin 'wildfly-maven-plugin:X.X.X:package'
```

The provisioned server will contain default artifacts instead of SNAPSHOT versions, and the override is silently ignored.

**Correct**:
```xml
<configuration>
    <channels>
        <channel>
            <manifest>
                <url>file://${elytron.override.manifest}</url>
            </manifest>
        </channel>
    </channels>
</configuration>
```

Note the `<url>` wrapper around the file path.

### ❌ Incompatible Manifest Schema Version

The manifest schema version must be compatible with your wildfly-maven-plugin version.

**Plugin version 6.0.1.Final and earlier**:
```yaml
schemaVersion: "1.1.0"  # ✅ Supported
```

**Schema version 2.1.0**:
```yaml
schemaVersion: "2.1.0"  # ❌ Not yet supported in 6.0.1.Final
```

Using an unsupported schema version causes:
```
[ERROR] Unknown schema version: 2.1.0
```

Always use `schemaVersion: "1.1.0"` for compatibility with current WildFly Maven Plugin versions. Check the [WildFly Channel specification](https://github.com/wildfly/wildfly-channel/blob/main/doc/spec.adoc) for schema version details.

### ❌ Reusing a Server After Changing Channels

Changing manifests, channel entries or stream versions does not update an already provisioned server. Document a separate stop and reprovision step for the project's configured home. If that home lives under `target/`, Maven clean can remove it; a visible `runtime/` home needs deliberate replacement. Do not automatically erase a running or user-modified server.

**User inspection after explicit reprovisioning** (paths below assume the advanced example's `target/server` layout):
```bash
# Check the actual provisioned artifact version
find target/server/modules -name "*elytron-http-oidc*.jar"
# Should show the SNAPSHOT version, not the release version

# Verify module.xml references the correct version
cat target/server/modules/system/layers/base/org/wildfly/security/elytron-http-oidc/main/module.xml | grep resource-root
```

### ❌ Missing `<url>` Wrapper for File-Based Manifests

When using a local file path, wrap it in `<url>`:

**Incorrect**:
```xml
<manifest>
    <file>${project.basedir}/override.yaml</file>
</manifest>
```

**Correct**:
```xml
<manifest>
    <url>file://${project.basedir}/override.yaml</url>
</manifest>
```

Or use an absolute path:
```xml
<manifest>
    <url>file:///home/user/project/override.yaml</url>
</manifest>
```

## Summary: Development Workflow Checklist

- [ ] Selected workflows match the user's requested scope; user instructions to run them are separate from documentation examples
- [ ] Latest LTS local JDK selected, or the concrete need for an older JDK explained
- [ ] Released server default and a single property override support normal release/SNAPSHOT selection
- [ ] User-owned WildFly source builds are documented as external manual preparation, never an implicit reproducer build or verification action
- [ ] Normal Maven repository/settings are used; special repositories are configured only when required
- [ ] Version switching, visible server homes and clean/reprovision behaviour are explained
- [ ] Remote debugging is documented when relevant, including a foreground launcher and optional suspend-at-boot behaviour
- [ ] If component overrides were requested, the needed artifact streams were identified and only the relevant profiles/manifests were added
- [ ] README separates the manual operations and explains their effects, progress views and state lifetime
- [ ] Agent checks stayed within authorized syntax/compilation scope; runtime or multi-mode checks were performed only when explicitly requested


---

**Last Updated**: 2026-11-25
**Version**: 1.4
