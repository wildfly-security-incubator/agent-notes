# Management Model Version Bump Guide

## Overview

This guide serves two purposes:

1. **Implementation Guide**: Describes the process for bumping the management model version for WildFly subsystems
2. **PR Review Checklist**: Ensures that PRs with management model changes have the correct version bump

A management model version bump is a distinct step that must be performed when the subsystem's management model changes, separate from schema changes or model content modifications.

**Important**: This guide covers ONLY the version bump process. Schema changes and model attribute modifications are separate tasks that should be documented separately.

**Living Document**: This guide should be kept up to date as we work on version bumps and review PRs. If you discover discrepancies, missing steps, or better practices, please update this document to reflect the current reality.

**Note**: While this guide uses elytron-oidc-client as the primary example, the patterns apply to all WildFly subsystems. Different subsystems may use different version management patterns (see "Subsystem Version Patterns" below).

## Purpose of Management Model Versioning

The management model version tracks the evolution of the subsystem's management API. When the model version is bumped:

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

## Pre-Bump Checklist

Before bumping the management model version, verify:

- [ ] **Current Version**: Identify the current model version (check `CURRENT` constant in `ElytronOidcClientSubsystemModel`)
- [ ] **Last Released Version**: Check the model version in the last WildFly .Final tag to determine if a bump has already occurred since the last release
  - Use: `git show <last-final-tag>:elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcClientSubsystemModel.java`
  - If the current version matches the last .Final tag version, a bump is needed
  - If the current version is already higher, a bump may not be needed (verify the reason for the existing bump)
- [ ] **Target WildFly Version**: Determine which WildFly version this bump targets
- [ ] **Previous Version Mapping**: Understand which WildFly versions map to which model versions
- [ ] **Reason for Bump**: Document why the version bump is needed (new attributes, removed attributes, behavioral changes)
- [ ] **Schema Status**: Confirm whether schema changes are being made separately or in conjunction

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
- **The `fromX()` methods MUST be defined in ASCENDING order in the file** (from2, from3, from4, from5, from6)
- This is the opposite of how they are CALLED in `registerTransformers()` (which calls them in descending order: from6, from5, from4...)
- **Incorrect ordering will cause confusion during code review and makes the code harder to maintain**
- When adding a new `fromX()` method, place it at the END of the transformer methods, just before the closing brace of the class
- Example: When adding `from6()`, it should be defined AFTER `from5()` in the file, even though it's called BEFORE `from5()` in `registerTransformers()`

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

**Note on Method Ordering**: The transformer methods are called in descending order (from6, from5, from4...) in `registerTransformers()`, but they must be defined in the file in ascending order (from2, from3, from4, from5, from6). This maintains code readability and follows the natural progression of version history.

**Important Notes**:
- The transformer method defines how to transform FROM the new version TO the previous version
- If this is a pure version bump with no model changes, the method body can be empty
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
4. **Transformer Method Definition Order**: The `fromX()` method definitions in the file are in ascending order (from2, from3, from4, from5, from6)
5. **Target Versions Array**: All intermediate versions are included in the array
6. **Comments**: WildFly version comments are accurate and helpful

## Common Patterns and Best Practices

### Pattern 1: Pure Version Bump (Standard Practice)

**This is the recommended approach**: Perform the version bump as a separate, isolated change.

When bumping the version without any model changes:
- Add the new version constant
- Update `CURRENT`
- Add an empty transformer method
- This reserves the version number for future use

**Commit Strategy**:
- **Minimum**: Version bump should be in its own commit to make PR review easier
- **Recommended**: Create a dedicated topic branch for the version bump and publish to `https://github.com/wildfly-security-incubator/wildfly`
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
- [ ] **Check Transformer Chain**: Ensure the transformer chain is properly updated if version was bumped
- [ ] **Validate Version Comments**: Ensure version constants have "WildFly X.0-onwards" comments
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

## Next Steps After Version Bump

After completing the management model version bump:

1. **Schema Changes**: If needed, bump the schema version separately
2. **Model Changes**: Add/modify attributes, operations, or capabilities
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
- [ ] Add WildFly version comment to new constant
- [ ] Add static import for new version in `ElytronOidcSubsystemTransformers.java`
- [ ] Create new `fromX()` transformer method in ascending order (after the previous `fromX-1()` method)
- [ ] Add `fromX()` call to transformer chain in `registerTransformers()` (in descending order, before the previous `fromX-1()` call)
- [ ] Add previous version to target versions array in `buildAndRegister()`
- [ ] Update transformer tests if needed (typically only when adding new controller version to test against)
- [ ] (Optional) Update or create `VERSIONS.md` with the new model version and target WildFly version
- [ ] Verify compilation succeeds
- [ ] Run transformer tests to verify backward compatibility
- [ ] Verify transformer chain call order is correct (descending: from6, from5, from4...)
- [ ] Verify transformer method definition order is correct (ascending: from2, from3, from4, from5, from6)
- [ ] Document the reason for the version bump

---

**Document Version**: 1.0
**Created**: 2026-05-27
**Target Issue**: WFLY-21934
**Related Docs**:
- `oidc-promotion-tracker.md` - Overall project tracker
- `wfly-21934-model-schema-work.md` - Detailed work log (to be created)