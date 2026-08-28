# Jakarta EE Security Patterns for WildFly Reproducers

## Overview

This guide captures patterns specific to creating reproducers for Jakarta EE Security (Soteria) issues in WildFly. For general WildFly reproducer patterns, see [wildfly-deployment-reproducer-guide.md](wildfly-deployment-reproducer-guide.md).

**Living Document**: Update this guide as new patterns emerge from Jakarta EE Security work.

## Authentication Mechanism Configuration

### HTTP BASIC Authentication

Use `@BasicAuthenticationMechanismDefinition` to enable HTTP BASIC authentication:

```java
import jakarta.annotation.security.DeclareRoles;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.security.enterprise.authentication.mechanism.http.BasicAuthenticationMechanismDefinition;

@BasicAuthenticationMechanismDefinition(realmName = "TestRealm")
@DeclareRoles({"Users", "Admins"})
@ApplicationScoped
public class SecurityConfig {
}
```

**Key points**:
- Can be placed on any CDI bean or servlet
- `@ApplicationScoped` makes it a CDI managed bean (required for CDI scanning)
- `@DeclareRoles` defines the application roles
- `realmName` appears in the HTTP challenge

**Servlet protection**:
```java
@WebServlet("/secured")
@ServletSecurity(@HttpConstraint(rolesAllowed = "Users"))
public class SecuredServlet extends HttpServlet {
    // ...
}
```

### FORM Authentication

Use `@FormAuthenticationMechanismDefinition` for form-based authentication:

```java
import jakarta.security.enterprise.authentication.mechanism.http.FormAuthenticationMechanismDefinition;
import jakarta.security.enterprise.authentication.mechanism.http.LoginToContinue;

@FormAuthenticationMechanismDefinition(
    loginToContinue = @LoginToContinue(
        loginPage = "/login.html",
        errorPage = "/error.html"
    )
)
@DeclareRoles({"Users", "Admins"})
@ApplicationScoped
public class FormSecurityConfig {
}
```

**Required files**:
- `src/main/webapp/login.html` — Login form with `j_security_check` action
- `src/main/webapp/error.html` — Error page shown on authentication failure

**Login form structure**:
```html
<!DOCTYPE html>
<html>
<head><title>Login</title></head>
<body>
    <h2>Login</h2>
    <form method="POST" action="j_security_check">
        <label>Username: <input type="text" name="j_username"></label><br>
        <label>Password: <input type="password" name="j_password"></label><br>
        <button type="submit">Login</button>
    </form>
</body>
</html>
```

**Important**: The form action must be `j_security_check` and the input names must be `j_username` and `j_password` — these are Jakarta EE Security standard names.

### CDI IdentityStore Requirement

Jakarta EE Security authentication mechanisms require a CDI `IdentityStore` bean for credential validation:

```java
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.security.enterprise.credential.Credential;
import jakarta.security.enterprise.credential.UsernamePasswordCredential;
import jakarta.security.enterprise.identitystore.CredentialValidationResult;
import jakarta.security.enterprise.identitystore.IdentityStore;

import java.util.Set;

import static jakarta.security.enterprise.identitystore.CredentialValidationResult.INVALID_RESULT;

@ApplicationScoped
public class TestIdentityStore implements IdentityStore {

    @Override
    public CredentialValidationResult validate(Credential credential) {
        if (credential instanceof UsernamePasswordCredential upc) {
            if ("testuser".equals(upc.getCaller()) && 
                "password1!".equals(upc.getPasswordAsString())) {
                return new CredentialValidationResult("testuser", Set.of("Users"));
            }
        }
        return INVALID_RESULT;
    }
}
```

**Key points**:
- Must be `@ApplicationScoped` for CDI discovery
- Returns `CredentialValidationResult` with caller principal and roles
- Return `INVALID_RESULT` for invalid credentials
- Each WAR can have its own `IdentityStore` with different credentials

**Why this is needed**: Soteria's `IdentityStoreHandler` looks up the CDI `IdentityStore` bean when validating credentials. Without it, you get:

```
java.lang.IllegalStateException: Could not find beans for Type=interface jakarta.security.enterprise.identitystore.IdentityStore
```

## WildFly Application Security Domain Configuration

### The `integrated-jaspi` Setting

WildFly's `application-security-domain` has an `integrated-jaspi` attribute that controls how JASPIC (which Soteria uses) integrates with Elytron:

- **`integrated-jaspi=true` (default)**: Elytron handles credential validation; JASPIC SAM handles authentication protocol only
- **`integrated-jaspi=false`**: JASPIC SAM handles both authentication protocol AND credential validation

**For Jakarta EE Security reproducers, always use `integrated-jaspi=false`**:

```cli
/subsystem=undertow/application-security-domain=ApplicationDomain:write-attribute(name=integrated-jaspi,value=false)
```

**Why**: With `integrated-jaspi=true`, Soteria's SAM is registered but Elytron handles the actual credential validation. This creates a mismatch:
- Soteria still expects a CDI `IdentityStore` to exist (for its internal logic)
- But Elytron is doing the validation using its own security realm
- If the Elytron realm doesn't have the user or the CDI `IdentityStore` is missing, authentication fails with confusing errors

With `integrated-jaspi=false`, Soteria handles the complete authentication flow including credential validation via the CDI `IdentityStore`.

**Minimal CLI script** for Jakarta EE Security reproducers:

```cli
# File: configure-security.cli
embed-server --std-out=echo

/subsystem=undertow/application-security-domain=ApplicationDomain:write-attribute(name=integrated-jaspi,value=false)

stop-embedded-server
```

This is all that's needed — no custom Elytron realms, security domains, or role decoders.

### Default Security Domain

WildFly's default `application-security-domain` is named `ApplicationDomain` (previously `other` in older versions). WARs automatically bind to this domain unless overridden via `jboss-web.xml`.

For Jakarta EE Security reproducers, you typically don't need `jboss-web.xml` — just configure `ApplicationDomain` with `integrated-jaspi=false`.

## CDI Bean Discovery

Jakarta EE Security annotations are CDI-based, so you must enable CDI scanning:

**Option 1: Empty beans.xml** (all classes scanned):
```xml
<!-- src/main/webapp/WEB-INF/beans.xml -->
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee 
                           https://jakarta.ee/xml/ns/jakartaee/beans_4_0.xsd"
       version="4.0" bean-discovery-mode="all">
</beans>
```

**Option 2: Annotated mode** (only classes with bean-defining annotations):
```xml
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee 
                           https://jakarta.ee/xml/ns/jakartaee/beans_4_0.xsd"
       version="4.0" bean-discovery-mode="annotated">
</beans>
```

For reproducers, `bean-discovery-mode="all"` is simpler — it ensures all classes are scanned without requiring specific annotations.

## Multi-WAR EAR Issues (Known Bugs)

### Bug 1: Shared CdiExtension

**Symptom**: When an EAR contains multiple WARs and any WAR uses Jakarta EE Security, Soteria's SAM is incorrectly installed in ALL WARs, including those using pure WildFly Elytron or no security at all.

**Root cause**: WildFly shares a single CDI portable extension instance (`org.glassfish.soteria.cdi.CdiExtension`) across all WAR subdeployments within an EAR. The extension's `httpAuthenticationMechanismFound` flag becomes true when processing the first WAR with Jakarta EE Security, and stays true for all subsequent WARs.

**Evidence in logs**:
```
Initializing Soteria 4.0.2 for context '/secured'   <-- Expected
Initializing Soteria 4.0.2 for context '/form'      <-- Expected
Initializing Soteria 4.0.2 for context '/digest'    <-- BUG: pure Elytron WAR
```

**How to reproduce**:
1. Create an EAR with multiple WARs
2. One or more WARs use Jakarta EE Security (`@BasicAuthenticationMechanismDefinition`, etc.)
3. One or more WARs use pure WildFly Elytron (web.xml security constraints, no EE Security annotations)
4. Deploy and check server logs for "Initializing Soteria" messages

### Bug 2: AmbiguousResolutionException

**Symptom**: When an EAR contains multiple WARs with different Jakarta EE Security authentication mechanisms, CDI throws `AmbiguousResolutionException` when trying to inject `HttpAuthenticationMechanism`.

**Root cause**: Same as Bug 1 — the shared `CdiExtension` registers all authentication mechanisms in the same CDI context. When any WAR tries to resolve `HttpAuthenticationMechanism`, CDI finds multiple beans and cannot determine which one to inject.

**Error**:
```
jakarta.enterprise.inject.spi.DeploymentException: WELD-001409: Ambiguous dependencies for type HttpAuthenticationMechanism
  Possible dependencies:
    - BasicAuthenticationMechanismDefinition (from war-secured)
    - FormAuthenticationMechanismDefinition (from war-form)
```

**How to reproduce**:
1. Create an EAR with multiple WARs
2. Each WAR uses a different Jakarta EE Security authentication mechanism
   - WAR 1: `@BasicAuthenticationMechanismDefinition`
   - WAR 2: `@FormAuthenticationMechanismDefinition`
3. Each WAR has its own `IdentityStore` and protected resources
4. Deploy — all Jakarta EE Security WARs will fail with HTTP 500

**Current behavior**: This makes it impossible to have multiple Jakarta EE Security authentication mechanisms in a single EAR.

### Workaround (None)

There is no workaround for these bugs without fixing Soteria. The issues are fundamental to how WildFly shares CDI portable extensions across EAR subdeployments.

## Required Galleon Layers

For Jakarta EE Security reproducers, include these layers:

```xml
<layers>
    <layer>ee-core-profile-server</layer>  <!-- Undertow, CDI, Naming -->
    <layer>ee-security</layer>              <!-- Soteria + JASPIC support -->
    <layer>core-tools</layer>               <!-- jboss-cli.sh for lifecycle -->
</layers>
```

**Do NOT add**:
- `undertow-https` — Causes keystore warnings and is not needed for HTTP BASIC/FORM testing
- Custom Elytron realms or security domains in CLI scripts (unless testing Elytron integration)

## Debugging Jakarta EE Security

### Key Classes for Breakpoints

Set breakpoints in these classes when debugging Jakarta EE Security initialization:

1. **`org.glassfish.soteria.cdi.CdiExtension`**
   - CDI portable extension that scans for authentication mechanism annotations
   - Look at `processBean()` method where `httpAuthenticationMechanismFound` is set
   - This runs during CDI scanning (before ServletContainerInitializer)

2. **`org.glassfish.soteria.servlet.SamRegistrationInstaller`**
   - `ServletContainerInitializer` that registers the JASPIC SAM
   - `onStartup()` method is the entry point
   - Check the `foundTypes` parameter (contains scanned classes)

3. **`org.glassfish.soteria.mechanisms.jaspic.Jaspic.registerSAM()`**
   - Actual SAM registration with JASPIC
   - Called from `SamRegistrationInstaller.onStartup()`

4. **`org.glassfish.soteria.mechanisms.HttpBridgeServerAuthModule`**
   - The JASPIC ServerAuthModule implementation
   - `validateRequest()` is called on each HTTP request

### Using `--suspend` Mode

Jakarta EE Security initialization happens during server boot:
1. CDI scanning (`CdiExtension` processing)
2. ServletContainerInitializer execution (`SamRegistrationInstaller.onStartup()`)
3. JASPIC SAM registration

To debug this initialization, start the server with `--suspend`:

```bash
./start-server.sh --debug --suspend
```

The server will pause before boot. Attach your debugger, set breakpoints in the classes above, then resume.

**Without `--suspend`**: By the time the server is fully started, initialization has already completed and your breakpoints in `CdiExtension` or `SamRegistrationInstaller` will never hit.

## Soteria Version Override

### Component Artifacts

Soteria consists of two Maven artifacts:

```yaml
streams:
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria"
    version: "4.0.3-SNAPSHOT"
  - groupId: "org.glassfish.soteria"
    artifactId: "soteria.spi.bean.decorator.weld"
    version: "4.0.3-SNAPSHOT"
```

**Always override both** when testing a Soteria SNAPSHOT — they are versioned together.

**Finding the artifacts**: Check the provisioned server's module structure:

```bash
ls -lh ear/target/server/modules/system/layers/base/org/glassfish/soteria/main/

# Output:
soteria-4.0.2.jar
soteria.spi.bean.decorator.weld-4.0.2.jar
```

### Building Soteria from Source

Soteria SNAPSHOT versions are not published. To test with 4.0.3-SNAPSHOT:

```bash
git clone https://github.com/eclipse-ee4j/soteria.git
cd soteria
git checkout 4.0
mvn clean install -DskipTests
```

This installs the SNAPSHOT to your local `~/.m2/repository`.

### Verifying the Override

After building with `-Dcomponent.override`:

```bash
# Check JAR versions
ls -lh ear/target/server/modules/system/layers/base/org/glassfish/soteria/main/*.jar

# Or check server logs
grep "Initializing Soteria" ear/target/server/standalone/log/server.log
```

You should see `4.0.3-SNAPSHOT` in the JAR filenames and log messages.

## Common Pitfalls

### ❌ Using `integrated-jaspi=true` with Jakarta EE Security

This causes credential validation to be handled by Elytron instead of Soteria, leading to mismatches:
- Missing `IdentityStore` errors
- Elytron realm configuration requirements
- Authorization failures with valid credentials

✅ **Fix**: Always set `integrated-jaspi=false` for Jakarta EE Security reproducers.

### ❌ Missing `beans.xml`

Without `beans.xml`, CDI scanning may not pick up the authentication mechanism annotations or `IdentityStore` beans.

✅ **Fix**: Include an empty `beans.xml` with `bean-discovery-mode="all"` in `src/main/webapp/WEB-INF/`.

### ❌ Multiple authentication mechanisms in one WAR

Defining both `@BasicAuthenticationMechanismDefinition` and `@FormAuthenticationMechanismDefinition` in the same WAR causes `AmbiguousResolutionException`.

✅ **Fix**: Each WAR should define only ONE authentication mechanism.

### ❌ Testing EAR without reproducing the multi-WAR bug

If your reproducer only has one Jakarta EE Security WAR, you won't reproduce the shared `CdiExtension` bug.

✅ **Fix**: Include at least two WARs — one with Jakarta EE Security and one without (or with a different security approach) — to demonstrate the SAM installation in all WARs.

### ❌ Overriding only one Soteria artifact

If you only override `soteria` without `soteria.spi.bean.decorator.weld`, you'll have version mismatches.

✅ **Fix**: Always override both artifacts together in your manifest.

## Summary Checklist for Jakarta EE Security Reproducers

- [ ] Galleon layers include `ee-security` and `ee-core-profile-server`
- [ ] CLI script sets `integrated-jaspi=false` on `ApplicationDomain`
- [ ] Each WAR has `beans.xml` with `bean-discovery-mode="all"`
- [ ] Each WAR with Jakarta EE Security has:
  - [ ] One `@*AuthenticationMechanismDefinition` annotation
  - [ ] One `IdentityStore` CDI bean
  - [ ] At least one servlet with `@ServletSecurity`
- [ ] For multi-WAR EAR reproducers testing the shared CdiExtension bug:
  - [ ] At least two WARs in the EAR
  - [ ] One or more WARs use Jakarta EE Security
  - [ ] One or more WARs use a different approach (pure Elytron or no security)
- [ ] README documents:
  - [ ] Which bug(s) are being reproduced
  - [ ] Expected vs actual behavior table
  - [ ] How to check server logs for evidence (`grep "Initializing Soteria"`)
  - [ ] Debugging tips (key classes, `--suspend` mode)
- [ ] If testing with Soteria SNAPSHOT:
  - [ ] `component-override.yaml` includes both Soteria artifacts
  - [ ] README documents the Soteria build-from-source requirement

---

**Last Updated**: 2026-08-28  
**Version**: 1.0
