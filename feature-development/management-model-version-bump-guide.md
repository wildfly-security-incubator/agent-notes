# WildFly Version Mapping and Management Model Guide

## Overview

Use this guide to establish version relationships before changing management configuration or reviewing compatibility claims, and to perform subsystem model version bumps.

It covers:

1. **Version Verification**: Distinguish product, Core, root management model, subsystem model and XML schema versions using sources from the exact release or commit.
2. **Host Excludes**: Verify which target management version a release alias selects.
3. **Subsystem Implementation and Review**: Determine whether a model version bump is needed, then update its registration, transformer chain and tests.

A management model version bump is a distinct step that must be performed when the subsystem's management model changes, separate from schema changes or model content modifications.

Schema changes and model attribute implementation are covered by the [schema version guide](schema-version-bump-guide.md) and [feature implementation guide](feature-implementation-guide.md).

**Living Document**: This guide should be kept up to date as we work on version bumps and review PRs. If you discover discrepancies, missing steps, or better practices, please update this document to reflect the current reality.

**Note**: While this guide uses elytron-oidc-client as the primary example, the patterns apply to all WildFly subsystems. Different subsystems may use different version management patterns (see "Subsystem Version Patterns" below).

## Version Relationships and Authoritative Sources

**WildFly product releases, WildFly Core releases, root management models, subsystem models and XML schemas are separate version sequences. Never infer one from another, including by assuming a fixed numerical offset.**

Use the following files at the exact target tag or commit. Paths are relative to the named GitHub repository; subsystem filenames vary, so the OIDC files are concrete examples.

| Version or mapping | GitHub repository and file to query | What establishes the value |
|--------------------|-------------------------------------|----------------------------|
| WildFly product release and selected Core artifact | `wildfly/wildfly`: [`pom.xml`](https://github.com/wildfly/wildfly/blob/42.0.0.Final/pom.xml) | Product tag/project version and `version.org.wildfly.core`, following dependency management if the selection is indirect |
| Root management model | `wildfly/wildfly-core`: [`version/src/main/java/org/jboss/as/version/Version.java`](https://github.com/wildfly/wildfly-core/blob/34.0.0.Final/version/src/main/java/org/jboss/as/version/Version.java) | `MANAGEMENT_MAJOR_VERSION`, `MANAGEMENT_MINOR_VERSION`, `MANAGEMENT_MICRO_VERSION` |
| Current root model construction | `wildfly/wildfly-core`: [`controller/src/main/java/org/jboss/as/controller/ModelVersion.java`](https://github.com/wildfly/wildfly-core/blob/34.0.0.Final/controller/src/main/java/org/jboss/as/controller/ModelVersion.java) | `ModelVersion.CURRENT` is constructed from the management constants in `Version` |
| Host-release alias and exclusion selection | `wildfly/wildfly-core`: [`host-controller/src/main/java/org/jboss/as/domain/controller/resources/HostExcludeResourceDefinition.java`](https://github.com/wildfly/wildfly-core/blob/35.0.0.Beta2/host-controller/src/main/java/org/jboss/as/domain/controller/resources/HostExcludeResourceDefinition.java) | `KnownRelease` maps the alias; `getVersionKey()` resolves it into the target management version |
| Kernel version enum values | `wildfly/wildfly-core`: [`host-controller/src/main/java/org/jboss/as/domain/controller/transformers/KernelAPIVersion.java`](https://github.com/wildfly/wildfly-core/blob/35.0.0.Beta2/host-controller/src/main/java/org/jboss/as/domain/controller/transformers/KernelAPIVersion.java) | Constructor arguments define a `ModelVersion`; `VERSION_35_0(35, 0, 0)` means management model 35.0.0, not Core release 35 |
| Subsystem management model | `wildfly/wildfly`: [`elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcClientSubsystemModel.java`](https://github.com/wildfly/wildfly/blob/42.0.0.Final/elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcClientSubsystemModel.java) and [`ElytronOidcExtension.java`](https://github.com/wildfly/wildfly/blob/42.0.0.Final/elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcExtension.java) in the same directory | Follow the model's `CURRENT` value through `registerSubsystem()`; for Core-owned subsystems, query their implementation in `wildfly/wildfly-core` |
| XML schema version | `wildfly/wildfly`: [`elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcSubsystemSchema.java`](https://github.com/wildfly/wildfly/blob/42.0.0.Final/elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcSubsystemSchema.java) and parser/writer registration in `ElytronOidcExtension.java` | Schema namespaces and the schema selected for the relevant stability level; these are independent of subsystem model versions |

The links above are pinned examples, not a table of current releases. For another release, change the tag in each repository independently. Subsystems delivered by external feature packs must be checked in the repository/tag providing that component.

### Query Exact Tags with GitHub MCP

Discover the available GitHub tools first. A file-content tool such as `get_file_contents` can retrieve historical source without a local checkout. For example:

```json
{
  "owner": "wildfly",
  "repo": "wildfly",
  "path": "pom.xml",
  "ref": "refs/tags/42.0.0.Final"
}
```

Read the selected Core dependency, then query its own tag:

```json
{
  "owner": "wildfly",
  "repo": "wildfly-core",
  "path": "version/src/main/java/org/jboss/as/version/Version.java",
  "ref": "refs/tags/34.0.0.Final"
}
```

Use the same operation for the other paths in the table. To compare releases, request the same file at each exact tag and compare the relevant symbols. To inspect a PR or development build, use its exact commit SHA; the GitHub file-content tool also accepts `sha` where available. Record the resolved commit and cite the tagged or commit-pinned source. A file retrieved from default `main` does not establish its contents in a historical release. A release name or release-note summary alone does not prove a management version.

If GitHub MCP is unavailable or fails, use GitHub's tag selector or a direct `https://github.com/<owner>/<repo>/blob/<tag-or-sha>/<path>` link. When a local checkout already exists, `git show <tag-or-sha>:<path>` provides the same source check without switching branches. Follow applicable workspace and tool-access rules.

### Use DeepWiki to Locate and Explain the Code

Where DeepWiki is available, `read_wiki_structure` can locate relevant topics and `ask_wiki_question` can help trace the version relationships. A focused question can name both repositories and the target releases:

```json
{
  "repoName": ["wildfly/wildfly", "wildfly/wildfly-core"],
  "question": "Which files and symbols establish the Core dependency, root management version and host-release mapping for WildFly 42.0.0.Final? Identify the indexed revision and distinguish it from verified source at Core 34.0.0.Final."
}
```

**Treat DeepWiki as a discovery and explanation aid. Verify release-specific values in the exact tagged files using GitHub MCP or direct source access.** The question tool has no tag/ref parameter; mentioning a tag in the question does not pin the indexed source to it. Check cited revisions rather than assuming that an indexed branch represents the requested release. If a file or tag is missing from the answer, retrieve it directly rather than filling the gap from version-number patterns.

## Host-Exclude Verification Process

A `host-exclude` resource describes extensions and configuration to exclude for target hosts. Its resource name is a label; the `host-release` alias or explicit management version determines the version key used for matching hosts. The release first introducing an extension is not itself the exclusion target: earlier hosts without the extension need it excluded.

Before changing an alias or explicit selector:

1. Identify the **target host's** exact WildFly release and its selected Core dependency from tagged source. Check build overrides or provisioned artifact versions if assessing a customized server.
2. Read that target Core release's `Version.java` management constants and confirm the `ModelVersion.CURRENT` construction. Record all three components.
3. Identify the **domain controller's** Core version that consumes the host-exclude configuration. In that revision, trace the proposed alias through `HostExcludeResourceDefinition.KnownRelease`, `KernelAPIVersion` and `getVersionKey()`. The target's Core tag and the consuming controller's Core tag can differ.
4. Compare the resolved version key with the target's verified management version. Preserve the intended extension exclusions. Prefer a supported release alias when it selects the correct target; if it is absent or incorrect, establish that with source evidence before choosing an explicit selector.
5. Record the product tag, target Core tag, management version, consuming controller revision and resolved alias value. Verify each side independently before declaring a mapping incorrect or disagreeing with a review comment. An alias definition plus a Core artifact number is not sufficient evidence.

### Verified Example: WildFly 42

| Check | Tagged source | Result |
|-------|---------------|--------|
| Target Core selection | [WildFly 42.0.0.Final POM](https://github.com/wildfly/wildfly/blob/42.0.0.Final/pom.xml) | Core **34.0.0.Final** |
| Target root management version | [Core 34.0.0.Final Version.java](https://github.com/wildfly/wildfly-core/blob/34.0.0.Final/version/src/main/java/org/jboss/as/version/Version.java) | **35.0.0** |
| Alias in a newer consuming controller | [Core 35.0.0.Beta2 HostExcludeResourceDefinition.java](https://github.com/wildfly/wildfly-core/blob/35.0.0.Beta2/host-controller/src/main/java/org/jboss/as/domain/controller/resources/HostExcludeResourceDefinition.java) | `WildFly42.0` maps to `KernelAPIVersion.VERSION_35_0` |
| Meaning of the enum constant | [Core 35.0.0.Beta2 KernelAPIVersion.java](https://github.com/wildfly/wildfly-core/blob/35.0.0.Beta2/host-controller/src/main/java/org/jboss/as/domain/controller/transformers/KernelAPIVersion.java) | `VERSION_35_0(35, 0, 0)` resolves to management **35.0.0** |

For this controller, the correct selector for WildFly 42 is:

```xml
<param name="host-release" value="WildFly42.0"/>
```

An explicit management version of **34.0.0** would target the wrong management model. The fact that the target uses a Core artifact numbered 34 does not change that conclusion. This example establishes these particular tagged values, not a rule that management versions always equal Core release numbers plus one.

## Purpose of Management Model Versioning

The remaining implementation and review sections concern **subsystem** management model versions. A subsystem model version tracks that subsystem's management API independently of the root management model. When the subsystem model version is bumped:

1. **Backward Compatibility**: Older WildFly versions can still communicate with newer versions through transformers
2. **Version Tracking**: Each WildFly release can be associated with a specific model version
3. **Change Documentation**: The version history provides a clear record of when changes were introduced

## Subsystem Version Patterns

WildFly subsystems use one of two patterns for managing model versions. Identify which pattern your subsystem uses before proceeding.

### Pattern A: Enum-based SubsystemModel (Recommended for New Subsystems)

Uses an enum implementing the `SubsystemModel` interface with three-part versioning (major.minor.micro).

**Example subsystems**: elytron-oidc-client, io, remoting, discovery

**File structure**:
```java
// Example: ElytronOidcClientSubsystemModel.java
enum ElytronOidcClientSubsystemModel implements SubsystemModel {
    VERSION_1_0_0(1, 0, 0),
    VERSION_2_0_0(2, 0, 0),
    VERSION_3_0_0(3, 0, 0), // WildFly 32.0-onwards
    ;
    static final ElytronOidcClientSubsystemModel CURRENT = VERSION_3_0_0;
    
    private final ModelVersion version;
    
    ElytronOidcClientSubsystemModel(int major, int minor, int micro) {
        this.version = ModelVersion.create(major, minor, micro);
    }
    
    @Override
    public ModelVersion getVersion() {
        return version;
    }
}
```

**Characteristics**:
- Dedicated `*SubsystemModel.java` file (e.g., `ElytronOidcClientSubsystemModel.java`)
- Enum constants with three-part versioning
- `CURRENT` static field points to latest version
- Extension class calls `CURRENT.getVersion()` for registration

### Pattern B: Direct ModelVersion Constants (Legacy Pattern)

Uses simple static final constants with single-part versioning (major only).

**Example subsystems**: elytron, core management

**File structure**:
```java
// Example: ElytronExtension.java (version constants at top of Extension class)
public class ElytronExtension implements Extension {
    static final ModelVersion ELYTRON_18_0_0 = ModelVersion.create(18);
    static final ModelVersion ELYTRON_19_0_0 = ModelVersion.create(19);
    static final ModelVersion ELYTRON_20_0_0 = ModelVersion.create(20);
    
    private static final ModelVersion ELYTRON_CURRENT = ELYTRON_20_0_0;
    
    @Override
    public void initialize(ExtensionContext context) {
        final SubsystemRegistration subsystem = context.registerSubsystem(
            SUBSYSTEM_NAME, 
            ELYTRON_CURRENT
        );
        // ...
    }
}
```

**Characteristics**:
- Version constants defined directly in the Extension class
- Single-part version numbering (major only)
- `CURRENT` private static field (or directly used constant)
- Simpler but less structured than enum pattern

### Which Pattern to Use?

- **For new subsystems**: Use Pattern A (enum-based) - it's more structured and provides better type safety
- **For existing subsystems**: Continue with the pattern already in use for consistency
- **When in doubt**: Check how other files in the subsystem reference versions

The rest of this guide primarily uses **Pattern A** (enum-based) examples. For **Pattern B** subsystems, adapt the steps accordingly:
- Version constants are added to the Extension class instead of a separate SubsystemModel file
- Registration uses the `CURRENT` constant directly instead of `CURRENT.getVersion()`
- Transformer imports come from the Extension class instead of a SubsystemModel class

## Key Files Involved

The management model version bump touches these core files:

**For Pattern A (enum-based) subsystems**:
1. **`*SubsystemModel.java`** (e.g., `ElytronOidcClientSubsystemModel.java`) - Defines model versions as enum constants
2. **`*Extension.java`** (e.g., `ElytronOidcExtension.java`) - Registers the current version
3. **`*SubsystemTransformers.java`** (e.g., `ElytronOidcSubsystemTransformers.java`) - Handles backward compatibility

**For Pattern B (direct constants) subsystems**:
1. **`*Extension.java`** (e.g., `ElytronExtension.java`) - Defines model version constants AND registers the current version
2. **`*SubsystemTransformers.java`** (e.g., `ElytronSubsystemTransformers.java`) - Handles backward compatibility

**Optional**:
4. **`VERSIONS.md`** - Version history documentation (see "Optional: Version History Documentation" section below)

## When Is a Model Version Bump Needed?

**Critical Check**: ALWAYS check the last tagged .Final WildFly release to determine if a bump is needed.

### Why This Check Matters

- **Prevents double-bumping**: Multiple developers working on features for the same WildFly release
- **Coordinates team work**: One bump creates shared base branch for multiple features
- **Avoids merge conflicts**: Only one version bump should exist per development cycle

### How to Check

**Step 1: Find the last .Final WildFly release tag**
```bash
# List recent WildFly tags
git tag -l "*Final" | tail -5

# Example output:
# 32.0.0.Final
# 33.0.0.Final
# 34.0.0.Final  ← Most recent
```

**Step 2: Check the model version in that tag**

**For Pattern A (enum-based) subsystems**:
```bash
# Example for elytron-oidc-client subsystem
git show 34.0.0.Final:elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcClientSubsystemModel.java | grep "CURRENT"

# Example output:
# static final ElytronOidcClientSubsystemModel CURRENT = VERSION_3_0_0;
```

**For Pattern B (direct constants) subsystems**:
```bash
# Example for elytron subsystem
git show 34.0.0.Final:elytron/src/main/java/org/wildfly/extension/elytron/ElytronExtension.java | grep "ELYTRON_CURRENT\|ELYTRON_[0-9]"

# Example output:
# static final ModelVersion ELYTRON_19_0_0 = ModelVersion.create(19);
# private static final ModelVersion ELYTRON_CURRENT = ELYTRON_19_0_0;
```

**Step 3: Check the current model version on your branch/main**
```bash
# Pattern A example
grep "CURRENT" elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcClientSubsystemModel.java

# Pattern B example
grep "ELYTRON_CURRENT" elytron/src/main/java/org/wildfly/extension/elytron/ElytronExtension.java
```

**Step 4: Compare and decide**

| Last .Final Tag | Current Branch | Decision |
|-----------------|----------------|----------|
| VERSION_3_0_0 | VERSION_3_0_0 | ✅ **Bump needed** - no bump since last release |
| VERSION_3_0_0 | VERSION_4_0_0 | ❌ **Bump NOT needed** - already bumped this cycle |
| ELYTRON_19_0_0 | ELYTRON_19_0_0 | ✅ **Bump needed** - no bump since last release |
| ELYTRON_19_0_0 | ELYTRON_20_0_0 | ❌ **Bump NOT needed** - already bumped this cycle |

**If Bump NOT Needed**:
- Someone else already created the bump for this development cycle
- Check Zulip #wildfly-elytron for announcement
- Use existing bump branch as base for your feature
- Do NOT create another bump

**If Bump Needed**:
- Post in Zulip #wildfly-elytron before proceeding
- Coordinate with team to avoid duplicate work
- Proceed with bump following this guide

### Pre-Bump Checklist

Before bumping the management model version, verify:

- [ ] **CRITICAL: Last .Final Tag Check** - Followed the 4-step check above to confirm bump is needed
- [ ] **Current Version**: Identified the current model version (check `CURRENT` constant)
- [ ] **Last Released Version**: Confirmed current version matches last .Final tag (bump needed)
- [ ] **Zulip Coordination**: Posted in #wildfly-elytron to coordinate version bumps
- [ ] **Target WildFly Version**: Determined which WildFly version this bump targets (e.g., 35.0.0)
- [ ] **Previous Version Mapping**: Understood which WildFly versions map to which model versions
- [ ] **Reason for Bump**: Documented why the version bump is needed (new attributes, removed attributes, behavioral changes)
- [ ] **Schema Status**: Confirmed whether schema changes are being made separately or in conjunction

## Version Bump Process

### Step 1: Add New Version Constant

**File**: `ElytronOidcClientSubsystemModel.java`

**What to Check**:
- Current version enum values and their WildFly version comments
- The `CURRENT` constant pointing to the latest version
- Version numbering pattern (major.minor.micro)

**What to Do**:
1. Add a new enum constant following the existing pattern
2. Include a comment indicating the target WildFly version
3. Update the `CURRENT` constant to point to the new version

**Example Pattern**:
```java
enum ElytronOidcClientSubsystemModel implements SubsystemModel {
    VERSION_1_0_0(1, 0, 0),
    VERSION_2_0_0(2, 0, 0),
    VERSION_3_0_0(3, 0, 0), // WildFly 32.0-onwards
    VERSION_4_0_0(4, 0, 0), // WildFly 33.0-onwards
    VERSION_5_0_0(5, 0, 0), // WildFly 40.0-onwards
    VERSION_6_0_0(6, 0, 0), // WildFly 41.0-onwards  <-- NEW
    ;
    static final ElytronOidcClientSubsystemModel CURRENT = VERSION_6_0_0; // <-- UPDATE
```

**Considerations**:
- **Always bump the major version** (e.g., 5.0.0 → 6.0.0) unless explicitly instructed otherwise
- The three-part version format (major.minor.micro) is used, but in practice only major versions are incremented
- Minor/micro version bumps are rare and should only be done if specifically required

### Step 2: Verify Extension Registration

**File**: `ElytronOidcExtension.java`

**What to Check**:
- The `initialize()` method registers the subsystem with `ElytronOidcClientSubsystemModel.CURRENT.getVersion()`
- This should automatically pick up the new version from Step 1

**What to Do**:
1. Verify the registration uses `CURRENT.getVersion()` (not a hardcoded version)
2. No changes should be needed if the pattern is correct

**Code to Verify**:
```java
@Override
public void initialize(ExtensionContext context) {
    final SubsystemRegistration subsystem = context.registerSubsystem(
        SUBSYSTEM_NAME,
        ElytronOidcClientSubsystemModel.CURRENT.getVersion()  // <-- Should use CURRENT
    );
    // ...
}
```

### Step 3: Add Transformer Chain Entry

**File**: `ElytronOidcSubsystemTransformers.java`

**What to Check**:
- Existing transformer chain methods (e.g., `from2()`, `from3()`, `from4()`, `from5()`)
- The `registerTransformers()` method that builds the chain
- The array of target versions in `buildAndRegister()`

**What to Do**:
1. Create a new `fromX()` method for the new version (where X is the new version number)
2. Add the method call to the chain in `registerTransformers()`
3. Add the previous version to the target versions array in `buildAndRegister()`

**CRITICAL: Method Ordering**:
- **Check the existing order in the file first and follow it.** The elytron-oidc-client convention below is ascending; the **wildfly-core `elytron` subsystem's `ElytronSubsystemTransformers.java` defines them in DESCENDING order** (`from20`, `from19`, `from18`, … newest first), so a new `from21()` goes above `from20()` there.
- In elytron-oidc-client, place the new method after the previous `fromX()` method. In wildfly-core Elytron, place it before the previous method, following the file's descending definition order.
- This is the opposite of how they are CALLED in `registerTransformers()` (which calls them in descending order: from6, from5, from4...)
- **Incorrect ordering will cause confusion during code review and makes the code harder to maintain**

**Example Pattern**:
```java
@Override
public void registerTransformers(SubsystemTransformerRegistration registration) {
    ChainedTransformationDescriptionBuilder chainedBuilder =
        TransformationDescriptionBuilder.Factory.createChainedSubystemInstance(
            registration.getCurrentSubsystemVersion()
        );

    // Methods are CALLED in descending order (newest to oldest)
    // 6.0.0 (WildFly 41) to 5.0.0 (WildFly 40)  <-- NEW
    from6(chainedBuilder);
    // 5.0.0 (WildFly 40) to 4.0.0 (WildFly 33)
    from5(chainedBuilder);
    // ... existing chains ...

    chainedBuilder.buildAndRegister(registration, new ModelVersion[] {
        VERSION_5_0_0.getVersion(),  // <-- ADD previous version
        VERSION_4_0_0.getVersion(),
        VERSION_3_0_0.getVersion(),
        VERSION_2_0_0.getVersion(),
        VERSION_1_0_0.getVersion()
    });
}

// Methods are DEFINED in ascending order (oldest to newest)
// ... from2(), from3(), from4(), from5() defined above ...

private static void from6(ChainedTransformationDescriptionBuilder chainedBuilder) {
    ResourceTransformationDescriptionBuilder builder =
        chainedBuilder.createBuilder(
            VERSION_6_0_0.getVersion(),
            VERSION_5_0_0.getVersion()
        );

    // Transformer rules will be added here when model changes are made
    // For a pure version bump with no model changes, this can be empty
}
```

**Note on Method Ordering**: The transformer methods are called in descending order (from6, from5, from4...) in `registerTransformers()`. In elytron-oidc-client they are defined in the file in ascending order (from2, from3, from4, from5, from6), which follows the natural progression of version history. The wildfly-core `elytron` subsystem defines them in descending order instead; always follow the existing file.

**Important Notes**:
- The transformer method defines how to transform FROM the new version TO the previous version
- For a pure bump, "empty transformer" means a builder for the new-version → previous-version delta with no feature-specific rules. Follow the subsystem's existing `fromX()` pattern and call `chainedBuilder.createBuilder(newVersion, previousVersion)`.
- Also add the previous version to `buildAndRegister()`'s target array. Check this array explicitly: successful tests against older targets do not establish that the immediate previous version is registered. The chained builder can supply empty deltas automatically, so a missing explicit builder may also be hidden until feature rules are added.
- If model changes are being made, transformation rules must be added (but that's a separate task)

### Step 4: Update Import Statements

**File**: `ElytronOidcSubsystemTransformers.java`

**What to Check**:
- Static imports at the top of the file for version constants

**What to Do**:
1. Add a static import for the new version constant

**Example**:
```java
import static org.wildfly.extension.elytron.oidc.ElytronOidcClientSubsystemModel.VERSION_1_0_0;
import static org.wildfly.extension.elytron.oidc.ElytronOidcClientSubsystemModel.VERSION_2_0_0;
import static org.wildfly.extension.elytron.oidc.ElytronOidcClientSubsystemModel.VERSION_3_0_0;
import static org.wildfly.extension.elytron.oidc.ElytronOidcClientSubsystemModel.VERSION_4_0_0;
import static org.wildfly.extension.elytron.oidc.ElytronOidcClientSubsystemModel.VERSION_5_0_0;
import static org.wildfly.extension.elytron.oidc.ElytronOidcClientSubsystemModel.VERSION_6_0_0; // <-- ADD
```

### Step 5: Update Transformer Tests

**File**: `ElytronOidcClientSubsystemTransformerTestCase.java` (in test directory)

**What to Check**:
- The `parameters()` method that defines test controller versions and model versions
- **Check for new controller versions**: Have any new `ModelTestControllerVersion` entries been added since this file was last modified?
- This test ensures transformers work correctly for backward compatibility

**How to Check for New Controller Versions**:
1. Check the `ModelTestControllerVersion` enum in wildfly-core:
   ```bash
   # If you have wildfly-core checked out locally:
   grep -A 2 "enum ModelTestControllerVersion" <path-to-wildfly-core>/model-test/src/main/java/org/jboss/as/model/test/ModelTestControllerVersion.java

   # Or view online:
   # https://github.com/wildfly/wildfly-core/blob/main/model-test/src/main/java/org/jboss/as/model/test/ModelTestControllerVersion.java
   ```
2. Compare available versions with those currently used in the transformer test
3. Check git history of the transformer test file to see when it was last updated:
   ```bash
   git log --oneline -- wildfly/elytron-oidc-client/src/test/java/org/wildfly/extension/elytron/oidc/ElytronOidcClientSubsystemTransformerTestCase.java
   ```

**Elytron subsystem (wildfly-core) variation**: `SubsystemTransformerTestCase` (`elytron/src/test/java/org/wildfly/extension/elytron/`) has no `parameters()` method. It has one explicit `@Test` per controller version (e.g. `testTransformerEAP810()`, `testTransformerWFLY41()`). When a new `ModelTestControllerVersion` is added (e.g. `WILDFLY_41_0_0` was added by WFCORE-7659), add a matching test method there. Adding the enum constant itself is a wildfly-core-wide change (`model-test` `ModelTestControllerVersion`, `core-model-test` `TransformersTestParameter`), so treat it as its own change rather than part of the Elytron bump.

**What to Do**:
1. **Always check**: Review available `ModelTestControllerVersion` values to see if new controller versions have been added
2. If a new controller version exists that should be tested, add a new test parameter entry
3. Determine which model version the new controller version should transform to (typically the previous stable version)

**Example Pattern**:
```java
@Parameterized.Parameters
public static Collection<Object[]> parameters() {
    return List.<Object[]>of(
        new Object[] { ModelTestControllerVersion.EAP_8_0_0, ElytronOidcClientSubsystemModel.VERSION_2_0_0.getVersion() },
        // Add new entry if a new controller version is available:
        // new Object[] { ModelTestControllerVersion.EAP_8_1_0, ElytronOidcClientSubsystemModel.VERSION_5_0_0.getVersion() }
    );
}
```

**Important Notes**:
- **Always perform this check** during version bumps and PR reviews to ensure we haven't skipped a controller version
- Transformer tests verify that the subsystem can transform its model to older versions
- New test entries should be added when there's a new target controller version to test transformation against
- Missing controller version tests can lead to compatibility issues in production
- The test uses XML files like `elytron-oidc-client-transform.xml` and `elytron-oidc-client-reject.xml` which may need updates if model changes are made (but not for a pure version bump)

## Verification Steps

After making the changes, verify:

1. **Compilation**: The code compiles without errors
2. **Version Consistency**: All references to the new version are consistent
3. **Transformer Chain Call Order**: The transformer chain calls in `registerTransformers()` are properly ordered (newest to oldest: from6, from5, from4...)
4. **Transformer Method Definition Order**: The `fromX()` method definitions follow the existing file's order (ascending in elytron-oidc-client: from2, from3, from4, from5, from6; descending in wildfly-core `elytron`: from20, from19, from18...)
5. **Target Versions Array**: All intermediate versions are included in the array
6. **Comments**: WildFly version comments are accurate and helpful (following subsystem conventions)
7. **Focused-Test Dependency Preparation**: A focused `mvn -pl <module> test` (such as `mvn -pl elytron test`) requires matching sibling SNAPSHOT artifacts in `~/.m2`. In a fresh checkout or after a project version bump, run a reactor `mvn install -DskipTests` first (or `mvn install -DallTests -DskipTests` if visiting all modules in an `allTests` profile build is needed) to populate local artifacts before running focused tests. `-am test` does not install sibling artifacts for later resolution.

## Common Patterns and Best Practices

### Pattern 1: Pure Version Bump (Standard Practice)

**This is the recommended approach**: Perform the version bump as a separate, isolated change.

When bumping the version without any model changes:
- Add the new version constant
- Update `CURRENT`
- Add an empty transformer method (calling `chainedBuilder.createBuilder(newVersion, previousVersion)` with no feature rules)
- Add the previous version to the target array in `buildAndRegister()`
- This reserves the version number for future use

**Commit Strategy**:
- **Minimum**: Version bump should be in its own commit to make PR review easier
- **Recommended**: Create a dedicated topic branch for the version bump and publish to the `wildfly-security-incubator` fork of the repository you are working in (e.g. `https://github.com/wildfly-security-incubator/wildfly-core` for the Elytron subsystem, `.../wildfly` for elytron-oidc-client)
  - This allows others to share the commit without duplication
  - Keeps the version bump isolated and reusable
  - Makes it easier to coordinate changes across multiple contributors

### Pattern 2: Version Bump Followed by Model Changes (Separate Steps)

**Important**: Avoid combining version bump and model changes in the same commit or branch.

**Recommended Workflow**:
1. **Step 1**: Create version bump topic branch
   - Perform pure version bump (Pattern 1)
   - Commit and publish to wildfly-security-incubator
   - This can be shared by others working on related changes
2. **Step 2**: Create model changes branch
   - Base on the version bump branch
   - Add transformation rules in the `fromX()` method
   - Make model attribute changes
   - Document the model changes

**Why Separate**:
- Easier PR review (version bump is mechanical, model changes need scrutiny)
- Version bump can be shared across multiple feature branches
- Cleaner git history
- Easier to revert if needed


## PR Review Guidelines

When reviewing PRs that modify the elytron-oidc-client subsystem management model:

### Version Bump Verification Checklist

- [ ] **Check Last .Final Tag**: Verify the model version in the last WildFly .Final release
  - Command: `git show <last-final-tag>:elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcClientSubsystemModel.java`
  - If current main branch already has a higher version than the last .Final tag, the version is correct
  - If current main branch has the same version as the last .Final tag, a version bump is required
- [ ] **Verify Version Bump is Separate**: If a version bump is included, it should be in its own commit
- [ ] **Check Transformer Chain**: Ensure the transformer chain is properly updated if version was bumped, including `buildAndRegister()` target array
- [ ] **Validate Version Comments**: Verify that the target WildFly release is accurately recorded, following the subsystem's existing conventions. For Elytron's direct model constants, the transformer-chain release comment and `VERSIONS.md` provide this information; an inline comment on every constant is not required.
- [ ] **No Model Changes Without Version Bump**: Reject PRs that modify the model without a corresponding version bump (unless the version was already bumped in main since the last .Final tag)

### Common Review Scenarios

**Scenario 1: PR adds new model attributes**
- ✅ Version must be bumped (unless already bumped in main since last .Final)
- ✅ Transformer rules must be added for the new attributes
- ✅ Version bump should be in separate commit

**Scenario 2: PR only bumps version (to wildfly-security-incubator)**
- ✅ Version bump commits should be published to wildfly-security-incubator for sharing
- ✅ Transformer method can be empty (reserves version for future use)
- ❌ **Do not merge bump-only PRs to WildFly main** - version bumps should only be merged when accompanied by at least one feature that modifies the model
- ✅ Other contributors can cherry-pick or rebase on the shared version bump commit

**Scenario 3: PR modifies model but version already bumped in main**
- ✅ Check that version in main is higher than last .Final tag
- ✅ Transformer rules must be added for the changes
- ✅ No additional version bump needed
- ❌ **Reject if PR includes redundant version bump** (version already bumped in main since last .Final)

**Scenario 4: Multiple PRs need version bump**
- ✅ First PR to merge should include the version bump
- ✅ Subsequent PRs should rebase on the version bump
- ✅ This is why publishing version bump to wildfly-security-incubator is important

### Review Rejection Criteria

**Reject if**:
- Model changes without version bump (and version not already bumped in main)
- Version bump combined with model changes in same commit
- Missing transformer chain updates
- Incorrect version numbering (should be major version bump)

## Files NOT Modified in a Pure Version Bump

The following files are NOT modified during a pure management model version bump:

- **`ElytronOidcSubsystemSchema.java`** - Schema versions are separate from model versions
- **`ElytronOidcSubsystemDefinition.java`** - Resource definitions don't reference model versions directly
- **`ElytronOidcSubsystemAdd.java`** - Add handlers don't reference model versions directly
- **Schema XSD files** - Schema files are versioned independently
- **Transformer Test XML files** - Test XML files (`elytron-oidc-client-transform.xml`, `elytron-oidc-client-reject.xml`) are only updated when model changes are made, not for pure version bumps

## Relationship to Schema Versions

**Important Distinction**:
- **Management Model Version**: Tracks the management API (attributes, operations, capabilities)
- **Schema Version**: Tracks the XML configuration format

These are related but separate:
- A model version bump may or may not require a schema version bump
- A schema version bump typically requires a model version bump
- They can be bumped independently or together depending on the changes

## Relationship to Stability Levels

**Critical Understanding**:
- **Model versions are independent of stability levels**
- If a change is made at ANY stability level (preview, community, default), the model version must be bumped (if not already bumped since the last .Final tag)
- A single model version can contain features at different stability levels
- **Stability level promotion DOES require a model version bump** if the feature was in a prior WildFly release at a lower stability level
- The model version bump is needed because the feature's availability changes across stability levels

**Examples**:
- Adding a preview-stability attribute → Requires model version bump
- Adding a community-stability attribute → Requires model version bump
- Adding a default-stability attribute → Requires model version bump
- Promoting an existing attribute from preview to community (feature was in prior release) → Requires model version bump
- Promoting an attribute from community to default (feature was in prior release) → Requires model version bump

**Key Principle**: The model version tracks changes to the management model structure AND changes to feature availability across stability levels. Any change that affects what features are available at what stability levels requires a version bump.

## Attribute Definition Best Practices

When adding or modifying management model attributes as part of your changes, follow these best practices to ensure proper API metadata and user experience.

### Stability Level Restrictions

**When**: Adding new attributes that are not available at DEFAULT stability

**Critical**: New management model attributes must include explicit stability level restrictions when they are not available at DEFAULT stability. Nested fields in `ObjectTypeAttributeDefinition` automatically inherit the parent's stability level - only the parent needs explicit `.setStability()`.

**Implementation**:
```java
import static org.jboss.as.version.Stability.COMMUNITY;

static final ObjectTypeAttributeDefinition BRUTE_FORCE_PROTECTION = 
    new ObjectTypeAttributeDefinition.Builder(
        ElytronDescriptionConstants.BRUTE_FORCE_PROTECTION,
        BF_ENABLED, BF_MAX_FAILED_ATTEMPTS, BF_LOCKOUT_INTERVAL,
        BF_SESSION_TIMEOUT, BF_MAX_CACHED_SESSIONS)
        .setRequired(false)
        .setStability(Stability.COMMUNITY)  // Critical - restricts to COMMUNITY+
        .setRestartAllServices()
        .build();
```

**Important Notes**:
- System properties can continue working at DEFAULT stability even when management attributes are restricted to COMMUNITY
- Child attributes automatically inherit parent's stability level
- Common stability levels: `Stability.DEFAULT`, `Stability.COMMUNITY`, `Stability.PREVIEW`, `Stability.EXPERIMENTAL`

### Measurement Units on Time and Size Attributes

**When**: Defining time-based or size-based attributes

**Critical**: Time-based and size-based management model attributes must explicitly specify measurement units using `.setMeasurementUnit()` for proper API metadata and tooling support.

**Required for**:
- Time-based attributes: SECONDS, MINUTES, HOURS, DAYS, MILLISECONDS
- Size-based attributes: BYTES, KILOBYTES, MEGABYTES, GIGABYTES
- Any numeric attribute with implicit units

**Implementation**:
```java
import org.jboss.as.controller.client.helpers.MeasurementUnit;

static final SimpleAttributeDefinition BF_LOCKOUT_INTERVAL = 
    new SimpleAttributeDefinitionBuilder(
        ElytronDescriptionConstants.BF_LOCKOUT_INTERVAL, 
        ModelType.INT)
        .setRequired(false)
        .setAllowExpression(true)
        .setMeasurementUnit(MeasurementUnit.MINUTES)  // CRITICAL - adds metadata
        .setValidator(new IntRangeValidator(-1, Integer.MAX_VALUE, true, true))
        .setRestartAllServices()
        .build();
```

**Benefits**:
1. Management API exposes units programmatically
2. Tools can auto-convert between units (display as hours for large values)
3. Self-documenting API
4. Consistent with WildFly platform best practices
5. Enables better validation in management tools

**Common Units**:
- `MeasurementUnit.MINUTES` - for timeout/interval attributes
- `MeasurementUnit.SECONDS` - for short durations
- `MeasurementUnit.MILLISECONDS` - for very short durations
- `MeasurementUnit.BYTES` - for size attributes

**Cross-Reference**: Must match resource bundle description (see "Resource Bundle Best Practices" below) and XSD documentation.

### Static Imports for Shared Attribute Constants

**When**: Referencing attributes defined in another class in ATTRIBUTES arrays

**Pattern**: Use static imports for shared attribute constants to improve readability and consistency, unless naming conflicts exist.

**Check for Conflicts First**:
```bash
# Check if target class defines conflicting constant
grep -n "BRUTE_FORCE_PROTECTION" PropertiesRealmDefinition.java
```

**If No Conflict** - Use Static Import:
```java
// Add static import
import static org.wildfly.extension.elytron.RealmDefinitions.BRUTE_FORCE_PROTECTION;

// Use unqualified reference
static final AttributeDefinition[] ATTRIBUTES = new AttributeDefinition[] { 
    USERS_PROPERTIES, GROUPS_PROPERTIES, GROUPS_ATTRIBUTE,
    BRUTE_FORCE_PROTECTION  // Clean, matches other array elements
};
```

**If Conflict Exists** - Keep Qualified Reference:
```java
// No static import
static final AttributeDefinition[] ATTRIBUTES = new AttributeDefinition[] { 
    USERS_PROPERTIES, GROUPS_PROPERTIES, GROUPS_ATTRIBUTE,
    RealmDefinitions.BRUTE_FORCE_PROTECTION  // Qualified to avoid conflict
};
```

## Resource Bundle Best Practices

Resource bundles (`LocalDescriptions.properties`) provide user-facing descriptions for management model attributes. Accuracy and consistency are critical for proper user configuration.

### Measurement Unit Accuracy

**Critical**: Resource bundle descriptions must accurately reflect measurement units (minutes vs milliseconds) and match the actual attribute definitions.

**Common Issue**: Describing values in wrong units (e.g., saying "milliseconds" when values are actually in minutes - off by factor of 60,000).

**Fix Pattern**:
```properties
# WRONG - says milliseconds when attribute uses minutes
elytron.custom-realm.brute-force-protection.lockout-interval=The duration (in milliseconds) that an account is locked out...

# CORRECT - says minutes, matches attribute definition
elytron.custom-realm.brute-force-protection.lockout-interval=Duration in minutes to lock out an account after max failed attempts. If not specified or set to -1, uses the WildFly Elytron library default (15 minutes).
```

**Verification Steps**:
1. Check attribute definition for `.setMeasurementUnit()` value
2. Ensure resource bundle description matches
3. Apply same wording pattern across all component types
4. Include library default values for consistency

**Impact**: Users could configure values off by 60,000x if description is wrong.

### Wording Consistency Across Component Types

**Critical**: Resource bundle entries for the same attribute across different component types must use consistent wording patterns and include library default information.

**Standard Pattern** for optional attributes with library defaults:
```properties
# Enabled field
elytron.jdbc-realm.brute-force-protection.enabled=Whether brute force protection is enabled for this realm.

# Numeric field with -1 sentinel
elytron.jdbc-realm.brute-force-protection.max-failed-attempts=Maximum number of failed authentication attempts before lockout. If not specified or set to -1, uses the WildFly Elytron library default (10).

# Time-based field with -1 sentinel
elytron.jdbc-realm.brute-force-protection.lockout-interval=Duration in minutes to lock out an account after max failed attempts. If not specified or set to -1, uses the WildFly Elytron library default (15 minutes).
```

**Information to Include**:
1. Brief description of what the field does
2. Measurement unit (minutes, seconds, etc.)
3. Sentinel value behavior (-1 = use default)
4. Library default value in parentheses

**Consistency Check**:
```bash
# Extract all descriptions for same field across component types
grep "brute-force-protection.enabled" LocalDescriptions.properties
```

All should use the same wording pattern. Inconsistent wording across realm types creates confusion for users.

## Subsystem-Specific Patterns

### Custom Component Integration (Elytron)

**Elytron Subsystem Pattern**: Custom components (custom-realm, custom-modifiable-realm) require coordinated implementation across Parser, XSD, and Runtime layers, including interface updates to pass management model context.

**Problem**: Custom components use a transformer interface which may lack access to `OperationContext` and `ModelNode`, making management model configuration non-functional.

**Three-Layer Fix Required**:

#### 1. Runtime Layer - Update Transformer Interface

**File**: `CustomComponentDefinition.java`

```java
public interface CustomComponentTransformer<T> {
    T prepareTransformer(
        OperationContext context,  // ADD THESE TWO
        ModelNode model,           // PARAMETERS
        String className,
        Map<String, String> configuration
    );
}
```

**File**: `ComponentAddHandler.java` - Update call site:

```java
// performRuntime() method
T transformer = transformerSupplier.prepareTransformer(
    context,  // was: null
    model,    // was: new ModelNode()
    className,
    configuration
);
```

#### 2. Parser Layer - Add Attribute to Custom Parsers

**File**: `RealmParser.java`

```java
static PersistentResourceXMLBuilder customRealmAttributes_19_0_community(
        PersistentResourceXMLBuilder builder) {
    return customRealmAttributes(builder)
        .addAttribute(RealmDefinitions.BRUTE_FORCE_PROTECTION,
            AttributeParser.OBJECT_PARSER, AttributeMarshaller.ATTRIBUTE_OBJECT);
}

private final PersistentResourceXMLDescription customRealmParser_19_0_community = 
    customRealmAttributes_19_0_community(
        builder(PathElement.pathElement(CUSTOM_REALM))
    ).build();
```

#### 3. XSD Layer - Add Element to Schema

**File**: `wildfly-elytron_community_19_0.xsd`

```xml
<xs:complexType name="customRealmType">
    <xs:sequence>
        <xs:element name="configuration" type="configurationType" minOccurs="0" maxOccurs="1"/>
        <xs:element name="brute-force-protection" type="bruteForceProtectionType" minOccurs="0"/>
    </xs:sequence>
    <!-- ... -->
</xs:complexType>
```

**Critical**: All three layers must be implemented together. Missing any layer causes silent failure (management model accepts values but runtime ignores them).

**Verification**: Test that configuration through both CLI and XML properly reaches the runtime implementation.

## Next Steps After Version Bump

After completing the management model version bump:

1. **Schema Changes**: If needed, bump the schema version separately
2. **Model Changes**: Add/modify attributes, operations, or capabilities
   - Follow "Attribute Definition Best Practices" above
   - Update resource bundles following "Resource Bundle Best Practices" above
   - For Elytron custom components, apply "Custom Component Integration" pattern
3. **Transformer Rules**: Add transformation rules for any model changes
4. **Testing**: Update and run tests to verify backward compatibility
5. **Documentation**: Update release notes and migration guides

## Optional: Version History Documentation

Consider adding a `VERSIONS.md` file at the subsystem root to document version history. This is particularly useful for:

- **Subsystems with frequent model changes**: Helps track what changed and when
- **Teams with multiple contributors**: Provides shared context for features and their target releases
- **Complex version histories**: Documents the purpose of each bump and related Jira issues

**When NOT to add VERSIONS.md**:
- Subsystems with infrequent version bumps may not need additional documentation
- Version history can alternatively be tracked through git log and commit messages
- Some teams prefer keeping documentation in Confluence or other systems

**Example structure** (see elytron-oidc-client/VERSIONS.md or elytron/VERSIONS.md):

```markdown
# [Subsystem Name] Version History

## Model Versions

| Model Version | WildFly Version | Notes |
|---------------|-----------------|-------|
| 3.0.0         | 42.0+           | Added feature X (WFLY-12345) |
| 2.0.0         | 41.0+           | Added feature Y (WFLY-12344) |
| 1.0.0         | 40.0+           | Initial release |

## Schema Versions

Schema versions are maintained independently from model versions.
See [Subsystem]SubsystemSchema.java for current schema versions.
```

If your subsystem would benefit from this documentation, create `<subsystem-root>/VERSIONS.md` following the pattern above.

## Troubleshooting

### Issue: Focused Subsystem Test Dependency Resolution Failure

**Symptoms**: `mvn -pl <module> test` fails with unresolved SNAPSHOT dependencies for sibling modules
**Solution**: Run a reactor build `mvn install -DskipTests` (or `mvn install -DallTests -DskipTests` if visiting all modules across all profiles is needed) once to install SNAPSHOT artifacts into your local repository (`~/.m2`), then rerun the focused subsystem test. Note that `mvn test -pl <module> -am` compiles upstream dependencies in the reactor but does not install them for subsequent independent artifact resolution.

### Issue: Compilation Errors After Version Bump

**Symptoms**: Missing imports, unresolved references
**Solution**: Verify all static imports are added and version constants are properly defined

### Issue: Transformer Chain Order Confusion

**Symptoms**: Unclear which version transforms to which
**Solution**: Remember the pattern: `fromX()` transforms FROM version X TO version X-1

### Issue: Missing Version in Target Array

**Symptoms**: Runtime errors about missing transformers
**Solution**: Ensure all intermediate versions are in the `buildAndRegister()` array

## Reference Examples

### Weld Subsystem

The Weld subsystem provides a good reference example:
- **Model**: `wildfly/weld/subsystem/src/main/java/org/jboss/as/weld/WeldExtension.java`
- **Transformers**: `wildfly/weld/subsystem/src/main/java/org/jboss/as/weld/WeldTransformers.java`

### XTS Subsystem

Another reference for simpler subsystems:
- **Location**: `wildfly/xts/`

## Summary Checklist

For a pure management model version bump:

- [ ] Add new version constant to `ElytronOidcClientSubsystemModel.java`
- [ ] Update `CURRENT` constant to point to new version
- [ ] Document target WildFly version (WildFly version comment for enum constants, or transformer comment and `VERSIONS.md` for direct constants)
- [ ] Add static import for new version in `ElytronOidcSubsystemTransformers.java`
- [ ] Create new `fromX()` transformer method following the file's existing order (elytron-oidc-client: ascending, after `fromX-1()`; wildfly-core `elytron`: descending, before `fromX-1()`)
- [ ] Add `fromX()` call to transformer chain in `registerTransformers()` (in descending order, before the previous `fromX-1()` call)
- [ ] Add previous version to target versions array in `buildAndRegister()`
- [ ] Update transformer tests if needed (typically only when adding new controller version to test against)
- [ ] (Optional) Update or create `VERSIONS.md` with the new model version and target WildFly version
- [ ] Verify compilation succeeds
- [ ] Run transformer tests to verify backward compatibility
- [ ] Verify transformer chain call order is correct (descending: from6, from5, from4...)
- [ ] Verify transformer method definition order matches the existing file (elytron-oidc-client ascending: from2, from3, ...; wildfly-core `elytron` descending: from20, from19, ...)
- [ ] Document the reason for the version bump

---

**Document Version**: 1.4
**Created**: 2026-05-27
**Last Updated**: 2026-11-18
**Tracking Issue**: [agent-notes #23](https://github.com/wildfly-security-incubator/agent-notes/issues/23)
**Related Docs**:

- [Schema version guide](schema-version-bump-guide.md)
- [Feature implementation guide](feature-implementation-guide.md)

## Revision History

### Version 1.4 (2026-11-18)

Expanded the guide to cover release/Core/model/schema relationships, authoritative tagged source, GitHub MCP and DeepWiki lookup options, and host-exclude verification (#23). Existing subsystem version bump procedures remain in this guide.

### Version 1.3 (2026-11-02)
Corrections found while preparing the WildFly 43 Elytron bumps (#21):
- `fromX()` definition order: follow the existing file (elytron-oidc-client ascending; wildfly-core `elytron` descending)
- Documented that the Elytron `SubsystemTransformerTestCase` has one explicit `@Test` per `ModelTestControllerVersion` rather than a `parameters()` method
- Incubator publish location is the `wildfly-security-incubator` fork of the repository being worked in

### Version 1.2 (2026-09-12)
Added critical version bump check requirements:
- **"When Is a Model Version Bump Needed?" section**: 4-step process to check last .Final tag
- **Prevents double-bumping**: Explains why checking last release is critical
- **Decision table**: Last .Final vs Current → Bump needed or not
- **Enhanced Pre-Bump Checklist**: Emphasizes last .Final tag check as CRITICAL first step

### Version 1.1 (2026-09-11)
Added best practices sections based on lessons from WFCORE-7193:
- **Attribute Definition Best Practices**: Stability level restrictions, measurement units, static imports
- **Resource Bundle Best Practices**: Measurement unit accuracy, wording consistency
- **Subsystem-Specific Patterns**: Custom component integration (Elytron)

### Version 1.0 (2026-05-27)
Initial version covering management model version bump process
