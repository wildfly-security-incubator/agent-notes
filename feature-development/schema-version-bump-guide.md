# Schema Version Bump Guide

## Overview

This guide serves two purposes:

1. **Implementation Guide**: Describes the process for bumping schema versions in WildFly subsystems
2. **PR Review Checklist**: Ensures that PRs with schema changes have the correct version bump and proper schema file management

A schema version bump is required when the XML configuration format changes, separate from (but often related to) management model version changes.

**Primary Examples**: This guide uses `elytron-oidc-client` as the primary example, with `elytron` subsystem patterns shown in the **Subsystem Variations** section.

**Important**: This guide covers ONLY the schema version bump process. Management model version bumps and attribute modifications are separate tasks documented in [`management-model-version-bump-guide.md`](management-model-version-bump-guide.md).

**Living Document**: This guide should be kept up to date as we work on schema version bumps and review PRs. If you discover discrepancies, missing steps, or better practices, please update this document to reflect the current reality.

## Purpose of Schema Versioning

The schema version tracks the evolution of the subsystem's XML configuration format. When the schema version is bumped:

1. **Configuration Compatibility**: Older configuration files can be parsed and migrated to newer formats
2. **XML Validation**: Each schema version provides XSD validation for its configuration format
3. **Change Documentation**: The schema history provides a clear record of configuration format changes
4. **Backward Compatibility**: Parsers for older schema versions remain available for migration

## Key Files Involved

The schema version bump touches these core files:

1. **[`ElytronOidcSubsystemSchema.java`](../wildfly/elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcSubsystemSchema.java)** - Defines schema versions and namespace URIs
2. **[`ElytronOidcExtension.java`](../wildfly/elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcExtension.java)** - Registers schema parsers
3. **Schema XSD files** - XML schema definition files in `src/main/resources/schema/`
4. **Parser classes** - Classes that parse each schema version (e.g., `ElytronOidcSubsystemParser_X_Y.java`)
5. **[`VERSIONS.md`](../wildfly/elytron-oidc-client/VERSIONS.md)** - Version history documentation (must be updated)

## Pre-Bump Checklist

Before bumping the schema version, verify:

- [ ] **Current Schema Version**: Identify the current schema version (check `CURRENT` constant in `ElytronOidcSubsystemSchema`)
- [ ] **Last Released Version**: Check ALL schema versions at ALL stability levels in the last WildFly .Final tag
  - Use: `git show <last-final-tag>:elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcSubsystemSchema.java`
  - **Critical**: Once a WildFly .Final release is published, those schema versions are **frozen**
  - Compare each stability level separately:
    - DEFAULT: Has it been bumped since last .Final?
    - COMMUNITY: Has it been bumped since last .Final?
    - PREVIEW: Has it been bumped since last .Final?
    - EXPERIMENTAL: Has it been bumped since last .Final?
  - If a schema at a specific stability level matches the last .Final tag, a bump is needed for that level
  - If already bumped since last .Final, verify you're not creating a second bump at the same stability level
- [ ] **One Bump Per Stability Level Per Release**: Verify that no more than one schema bump exists at each stability level since the last .Final release
  - **Rule**: Each stability level should have at most ONE schema version bump between WildFly releases
  - If a bump already exists at your target stability level since last .Final, you should reuse that version
  - Multiple features can share the same schema version bump
  - See [Schema Version Bump Constraints](#schema-version-bump-constraints) for details
- [ ] **Target WildFly Version**: Determine which WildFly version this bump targets
- [ ] **Reason for Bump**: Document WHY the schema bump is needed. Common reasons:
  - Adding new XML elements/attributes at any stability level
  - Promoting features from lower to higher stability (e.g., Preview → Community)
  - Removing or deprecating XML elements/attributes
  - Changing XML structure or semantics
  - See [Understanding Stability Levels and Schema Bumps](#understanding-stability-levels-and-schema-bumps) for detailed scenarios
- [ ] **Model Version Status**: Confirm whether a management model version bump is also needed (typically yes if schema changes)
- [ ] **Stability Level**: Determine the stability level of the new schema version
  - What is the lowest stability level of features being added or promoted?
  - Experimental → Preview → Community → Default (higher levels are subsets of lower)
  - The schema annotation should match the lowest stability level of new/promoted features
- [ ] **Lower Stability Level Impact**: If adding a feature at COMMUNITY or DEFAULT, check if PREVIEW or EXPERIMENTAL schemas exist and need updating

## Schema Version Bump Constraints

### The "One Bump Per Stability Level Per Release" Rule

**Critical Rule**: Each stability level should have **at most ONE schema version bump** between WildFly .Final releases.

**Why This Rule Exists**:
1. **Release Coordination**: Keeps schema versions aligned with WildFly releases
2. **Version Management**: Prevents version number explosion
3. **Feature Batching**: Encourages grouping related features in a single release
4. **Compatibility**: Simplifies backward compatibility testing

### Checking for Existing Bumps

**Step 1: Identify the last WildFly .Final release**
```bash
# Find the most recent .Final tag
git tag -l "*.Final" | sort -V | tail -1
# Example output: 40.0.0.Final
```

**Step 2: Check schema versions at each stability level in that tag**
```bash
# View the schema file from the last .Final tag
git show 40.0.0.Final:elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcSubsystemSchema.java

# Look for the highest version at each stability level:
# - DEFAULT (no annotation or @Stability(StabilityLevel.DEFAULT))
# - COMMUNITY (@Stability(StabilityLevel.COMMUNITY))
# - PREVIEW (@Stability(StabilityLevel.PREVIEW))
# - EXPERIMENTAL (@Stability(StabilityLevel.EXPERIMENTAL))
```

**Step 3: Compare with current main branch**
```bash
# View current schema versions
cat elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcSubsystemSchema.java
```

**Step 4: Determine if a bump is needed or already exists**

| Stability Level | Last .Final | Current Main | Action |
|----------------|-------------|--------------|--------|
| DEFAULT | 2.0 | 2.0 | ✅ Bump needed (if adding DEFAULT feature) |
| DEFAULT | 2.0 | 3.0 | ⚠️ Bump exists - reuse 3.0, don't create 4.0 |
| PREVIEW | 5.0 | 5.0 | ✅ Bump needed (if adding PREVIEW feature) |
| PREVIEW | 5.0 | 6.0 | ⚠️ Bump exists - reuse 6.0, don't create 7.0 |

### When to Reuse an Existing Bump

**Scenario**: You want to add a PREVIEW feature, but PREVIEW schema was already bumped since last .Final.

**Example**:
- Last .Final (WildFly 40): PREVIEW schema is VERSION_5_0
- Current main: PREVIEW schema is VERSION_6_0 (already bumped for WildFly 41)
- Your feature: Adding `token-signature-algorithm` at PREVIEW

**Correct Action**:
1. ✅ **Reuse VERSION_6_0** - add your feature to the existing 6.0 schema
2. ✅ Update the XSD for VERSION_6_0 to include your new element
3. ✅ Update the parser for VERSION_6_0 to handle your new element
4. ❌ **Do NOT create VERSION_7_0** - this would violate the one-bump-per-level rule

**Why Reuse**:
- Multiple features can share the same schema version
- All features added between WildFly 40 and 41 at PREVIEW level go into VERSION_6_0
- This keeps version numbers manageable and aligned with releases

### When a New Bump IS Needed

**Scenario**: You want to add a PREVIEW feature, and no PREVIEW bump exists since last .Final.

**Example**:
- Last .Final (WildFly 40): PREVIEW schema is VERSION_5_0
- Current main: PREVIEW schema is still VERSION_5_0 (no bump yet)
- Your feature: Adding `token-signature-algorithm` at PREVIEW

**Correct Action**:
1. ✅ **Create VERSION_6_0** with PREVIEW annotation
2. ✅ This is the first (and should be only) PREVIEW bump for WildFly 41
3. ✅ Publish this bump to wildfly-security-incubator so others can share it
4. ✅ Other developers adding PREVIEW features for WildFly 41 should reuse your VERSION_6_0

### Frozen Schemas After .Final Release

**Critical Understanding**: Once a WildFly .Final release is published, the schema versions in that release are **frozen** and must not be modified.

**What "Frozen" Means**:
- ❌ Cannot add new elements/attributes to released schema versions
- ❌ Cannot change XSD files for released schema versions
- ❌ Cannot modify parsers for released schema versions
- ✅ Can only add NEW schema versions for future releases

**Example**:
- WildFly 40.0.0.Final released with PREVIEW VERSION_5_0
- VERSION_5_0 is now frozen
- To add PREVIEW features for WildFly 41, must create VERSION_6_0
- Cannot modify VERSION_5_0 even if you find a bug (must fix in VERSION_6_0)

### Coordination Across Developers

**Best Practice**: When starting work on a feature for the next WildFly release:

1. **Check for existing bump commits** in wildfly-security-incubator
2. **Communicate with other developers** working on features at the same stability level
3. **Share the schema bump commit** so everyone uses the same version
4. **Add your feature to the shared schema version** rather than creating a new bump

**Example Workflow**:
```bash
# Developer A (first to work on PREVIEW features for WildFly 41)
git checkout -b preview-schema-bump-wf41
# Create VERSION_6_0 at PREVIEW
git commit -m "Bump PREVIEW schema to 6.0 for WildFly 41"
git push origin preview-schema-bump-wf41
# Create PR to wildfly-security-incubator

# Developer B (also working on PREVIEW features for WildFly 41)
git fetch origin
git checkout preview-schema-bump-wf41
# Rebase their feature work on top of the shared bump
# Add their feature to VERSION_6_0
```

## Relationship Between Schema and Model Versions

**Critical Understanding**:
- **Schema versions track XML configuration format**
- **Model versions track management API structure**
- These are related but independent versioning systems

**Common Patterns**:
- Schema change → Usually requires model version bump
- Model change → May or may not require schema bump (depends on whether XML configuration is affected)
- Stability promotion → May require both schema and model version bumps

## Understanding Stability Levels and Schema Bumps

### Stability Level Hierarchy

WildFly features progress through a stability level hierarchy:

```
Experimental → Preview → Community → Default
(lowest)                              (highest)
```

**Key Principles**:
- **Higher stability levels are subsets of lower levels**: Default ⊂ Community ⊂ Preview ⊂ Experimental
- **Features can be added at any stability level**
- **Features can be promoted incrementally** (e.g., Preview → Community → Default)
- **Features can skip levels** (e.g., Preview → Default)
- **Each stability level includes all features from higher levels**

### When Schema Bumps Are Required

Understanding WHY you're bumping a schema is critical for determining the correct approach. Schema bumps are required in these scenarios:

**Important**: In all scenarios below, the schema version bump should be performed as a **separate commit** (and ideally a separate topic branch) from the actual XML format changes or feature additions. This makes PR review easier and allows multiple developers to share the same version bump commit.

#### Scenario 1: Adding New XML Elements/Attributes at ANY Stability Level

**When**: Introducing new configuration options in the XML format
**Schema Bump**: ✅ Required
**Model Bump**: ✅ Required
**Reason**: The XML format structure has changed

**Workflow**:
1. **First**: Create schema version bump commit (separate from feature implementation)
2. **Then**: Implement the new XML element/attribute in a separate commit

**Example**:
```xml
<!-- Old schema (5.0) -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:5.0">
    <secure-deployment name="app">
        <client-id>my-app</client-id>
    </secure-deployment>
</subsystem>

<!-- New schema (6.0) - added new element at PREVIEW stability -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:6.0">
    <secure-deployment name="app">
        <client-id>my-app</client-id>
        <token-signature-algorithm>RS256</token-signature-algorithm> <!-- NEW -->
    </secure-deployment>
</subsystem>
```

**Schema Version Annotation**:
```java
@Stability(StabilityLevel.PREVIEW)
VERSION_6_0(6, 0), // WildFly 41.0-onwards - Added token-signature-algorithm
```

**Commit Strategy**: Publish the schema bump commit to wildfly-security-incubator so other developers adding PREVIEW features can share it.

#### Scenario 2: Promoting Features to Higher Stability Level

**When**: Moving existing features from lower to higher stability (e.g., Preview → Community)
**Schema Bump**: ✅ Required
**Model Bump**: ✅ Required
**Reason**: The feature's availability changes across stability levels

**Workflow**:
1. **First**: Create schema version bump commit with updated stability annotation
2. **Then**: Make any feature-specific changes in separate commits (if needed)

**Example**:
```java
// Schema 5.0 - feature at PREVIEW
@Stability(StabilityLevel.PREVIEW)
VERSION_5_0(5, 0), // WildFly 40.0-onwards - token-signature-algorithm at PREVIEW

// Schema 6.0 - same feature promoted to COMMUNITY
@Stability(StabilityLevel.COMMUNITY)
VERSION_6_0(6, 0), // WildFly 41.0-onwards - token-signature-algorithm promoted to COMMUNITY
```

**Why Bump?**:
- Users running at COMMUNITY stability can now use this feature
- The XML format is available to a broader audience
- The schema version documents when the feature became available at each stability level

**Commit Strategy**: This is especially important for stability promotions - multiple developers may be promoting different features to the same stability level. Publishing the schema bump commit to wildfly-security-incubator allows everyone to share the same version bump.

#### Scenario 3: Removing or Deprecating XML Elements/Attributes

**When**: Removing configuration options from the XML format
**Schema Bump**: ✅ Required
**Model Bump**: ✅ Required
**Reason**: The XML format structure has changed

**Example**:
```xml
<!-- Old schema (5.0) - deprecated element still present -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:5.0">
    <secure-deployment name="app">
        <legacy-option>value</legacy-option> <!-- Deprecated -->
    </secure-deployment>
</subsystem>

<!-- New schema (6.0) - deprecated element removed -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:6.0">
    <secure-deployment name="app">
        <!-- legacy-option no longer supported -->
    </secure-deployment>
</subsystem>
```

#### Scenario 4: Changing XML Structure or Semantics

**When**: Modifying how XML elements are organized or interpreted
**Schema Bump**: ✅ Required
**Model Bump**: ✅ Required
**Reason**: The XML format structure or meaning has changed

**Example**:
```xml
<!-- Old schema (5.0) - flat structure -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:5.0">
    <provider name="keycloak">
        <auth-server-url>https://...</auth-server-url>
        <ssl-required>external</ssl-required>
    </provider>
</subsystem>

<!-- New schema (6.0) - nested structure -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:6.0">
    <provider name="keycloak">
        <connection>
            <auth-server-url>https://...</auth-server-url>
            <ssl-required>external</ssl-required>
        </connection>
    </provider>
</subsystem>
```

### When Schema Bumps Are NOT Required

#### Scenario: Management-Only Changes (No XML Impact)

**When**: Adding management operations or capabilities that don't affect XML configuration
**Schema Bump**: ❌ Not Required
**Model Bump**: ✅ Required
**Reason**: XML format unchanged, only management API changed

**Example**: Adding a new management operation like `reload-provider` that doesn't require XML configuration changes.

### Stability Level Annotations on Schema Versions

**Purpose**: The `@Stability` annotation on a schema version indicates the **minimum stability level** required to use that schema.

**Rules**:
1. **No annotation** = Available at DEFAULT stability (and all lower levels)
2. **`@Stability(StabilityLevel.COMMUNITY)`** = Available at COMMUNITY and lower (Preview, Experimental)
3. **`@Stability(StabilityLevel.PREVIEW)`** = Available at PREVIEW and lower (Experimental)
4. **`@Stability(StabilityLevel.EXPERIMENTAL)`** = Available only at EXPERIMENTAL

**Example Progression**:
```java
enum ElytronOidcSubsystemSchema implements PersistentSubsystemSchema<ElytronOidcSubsystemSchema> {
    VERSION_1_0(1, 0), // WildFly 26 - DEFAULT stability (no annotation)
    VERSION_2_0(2, 0), // WildFly 27 - DEFAULT stability (no annotation)

    @Stability(StabilityLevel.PREVIEW)
    VERSION_3_0(3, 0), // WildFly 32 - New features at PREVIEW

    @Stability(StabilityLevel.PREVIEW)
    VERSION_4_0(4, 0), // WildFly 33 - More features at PREVIEW

    @Stability(StabilityLevel.PREVIEW)
    VERSION_5_0(5, 0), // WildFly 40 - Additional features at PREVIEW

    @Stability(StabilityLevel.COMMUNITY)
    VERSION_6_0(6, 0), // WildFly 41 - Features promoted to COMMUNITY

    // Future: VERSION_7_0 with no annotation when promoted to DEFAULT
}
```

### Schema Versioning Strategy Across Stability Levels

**Critical Understanding**: Schema versions are **independent across stability levels**. The version numbers in schema namespaces (e.g., `5.0`, `6.0`) are not directly related across different stability levels.

**Key Principles**:
1. **Not all stability levels always have a "current" schema**: If all features at a stability level have been promoted, we don't need a current schema at that level
2. **Schema forking follows the stability hierarchy**: When adding features, fork from the nearest lower stability level
3. **Bidirectional impact**: Adding features at higher stability levels may require forking lower stability level schemas
4. **Version numbers are independent but coordinated**: Each stability level has its own version sequence, but we try to align version numbers when forking

#### Version Numbering When Forking Schemas

**General Rule**: When forking a schema from one stability level to another, try to use the same version number if possible, but use the next available version if that number is already taken.

##### Case 1: Forking Within Same Stability Level (Simple)

**Scenario**: Adding a new PREVIEW feature when PREVIEW schema already exists

**Example**:
- Current PREVIEW schema: `VERSION_4_0` → `wildfly-elytron-oidc-client_4_0.xsd`
- New PREVIEW schema: `VERSION_5_0` → `wildfly-elytron-oidc-client_5_0.xsd`

**Rule**: Simply increment to the next major version (4.0 → 5.0)

##### Case 2: Forking Across Stability Levels (Ideal Case)

**Scenario**: Adding a COMMUNITY feature by forking DEFAULT schema, and the version number is available

**Example**:
- Current DEFAULT schema: `VERSION_2_0` → `wildfly-elytron-oidc-client_2_0.xsd`
- No COMMUNITY schema exists yet
- Version 2.0 is not used by any COMMUNITY schema
- New COMMUNITY schema: `VERSION_2_0` → `wildfly-elytron-oidc-client_2_0.xsd` (with COMMUNITY annotation)

**Rule**: Use the same version number as the source schema when forking across stability levels, if that version is available at the target stability level.

**Why**: This makes it clear that the COMMUNITY 2.0 schema is based on DEFAULT 2.0 schema.

##### Case 3: Forking Across Stability Levels (Version Conflict)

**Scenario**: Adding a COMMUNITY feature by forking DEFAULT schema, but the version number is already taken

**Example**:
- Current DEFAULT schema: `VERSION_2_0` → `wildfly-elytron-oidc-client_2_0.xsd`
- Current COMMUNITY schema: `VERSION_2_0` → `wildfly-elytron-oidc-client_2_0.xsd` (already exists!)
- Version 2.0 is already used by COMMUNITY
- New COMMUNITY schema: `VERSION_3_0` → `wildfly-elytron-oidc-client_3_0.xsd` (with COMMUNITY annotation)

**Rule**: Use the next available major version at the target stability level (2.0 is taken, so use 3.0)

**Why**: Schema versions are developed independently at each stability level, so version numbers can diverge.

##### Version Number Selection Algorithm

When creating a new schema version:

```
1. Determine target stability level (EXPERIMENTAL, PREVIEW, COMMUNITY, DEFAULT)

2. If forking within same stability level:
   → Use next major version (current + 1)

3. If forking from different stability level:
   a. Check source schema version (e.g., DEFAULT 2.0)
   b. Check if that version is available at target level
   c. If available:
      → Use same version number (e.g., COMMUNITY 2.0)
   d. If not available:
      → Use next available major version at target level

4. Document the fork relationship in comments:
   VERSION_2_0(2, 0), // WildFly 41 - COMMUNITY - forked from DEFAULT 2.0
```

##### Practical Examples

**Example 1: Clean Fork**
```java
// DEFAULT schemas
VERSION_1_0(1, 0), // WildFly 26
VERSION_2_0(2, 0), // WildFly 32

// COMMUNITY schemas (forked from DEFAULT)
@Stability(StabilityLevel.COMMUNITY)
VERSION_2_0(2, 0), // WildFly 41 - forked from DEFAULT 2.0
```

**Example 2: Version Conflict**
```java
// DEFAULT schemas
VERSION_1_0(1, 0), // WildFly 26
VERSION_2_0(2, 0), // WildFly 32

// COMMUNITY schemas (developed independently)
@Stability(StabilityLevel.COMMUNITY)
VERSION_2_0(2, 0), // WildFly 40 - initial COMMUNITY schema
@Stability(StabilityLevel.COMMUNITY)
VERSION_3_0(3, 0), // WildFly 41 - forked from DEFAULT 2.0, but 2.0 was taken
```

**Example 3: Multiple Stability Levels**
```java
// DEFAULT schemas
VERSION_1_0(1, 0), // WildFly 26
VERSION_2_0(2, 0), // WildFly 32

// COMMUNITY schemas
@Stability(StabilityLevel.COMMUNITY)
VERSION_2_0(2, 0), // WildFly 40 - forked from DEFAULT 2.0

// PREVIEW schemas
@Stability(StabilityLevel.PREVIEW)
VERSION_2_0(2, 0), // WildFly 40 - forked from COMMUNITY 2.0
@Stability(StabilityLevel.PREVIEW)
VERSION_3_0(3, 0), // WildFly 41 - additional PREVIEW features
```

##### Key Takeaways

1. **Same stability level**: Always increment (4.0 → 5.0)
2. **Cross stability level**: Try to match source version, but use next available if taken
3. **Independence**: Each stability level has its own version sequence
4. **Documentation**: Always comment the fork relationship and WildFly version
5. **Coordination**: Check existing versions before choosing a number

#### Adding a New Feature: Step-by-Step Decision Process

Use this table to determine which schemas need to be created/updated when adding a new feature:

| Adding Feature At | Check For Current Schema At | Fork From | Also Update |
|-------------------|----------------------------|-----------|-------------|
| **EXPERIMENTAL** | Experimental | Fork Experimental | N/A (lowest level) |
| | (none exists) | Fork Preview → make Experimental | N/A |
| | (none exists) | Fork Community → make Experimental | N/A |
| | (none exists) | Fork Default → make Experimental | N/A |
| **PREVIEW** | Preview | Fork Preview | Experimental (if exists) |
| | (none exists) | Fork Community → make Preview | Experimental (if exists) |
| | (none exists) | Fork Default → make Preview | Experimental (if exists) |
| **COMMUNITY** | Community | Fork Community | Preview, Experimental (if exist) |
| | (none exists) | Fork Default → make Community | Preview, Experimental (if exist) |
| **DEFAULT** | Default | Fork Default | Community, Preview, Experimental (if exist) |

#### Detailed Workflow Examples

##### Example 1: Adding a PREVIEW Feature

**Scenario**: You want to add `token-signature-algorithm` at PREVIEW stability.

**Step 1: Check for current PREVIEW schema**
```bash
# Look at ElytronOidcSubsystemSchema.java
# Find the highest version with @Stability(StabilityLevel.PREVIEW)
```

**Case A: Current PREVIEW schema exists (e.g., VERSION_5_0)**
1. Fork VERSION_5_0 → create VERSION_6_0 with PREVIEW annotation
2. Copy XSD file: `wildfly-elytron-oidc-client_5_0.xsd` → `wildfly-elytron-oidc-client_6_0.xsd`
3. Update namespace in new XSD to `6.0`
4. Add new element to XSD
5. Update or create parser for VERSION_6_0
6. No need to check other stability levels (PREVIEW is lower than COMMUNITY/DEFAULT)

**Case B: No current PREVIEW schema exists**
1. Check for current COMMUNITY schema (e.g., VERSION_4_0 with COMMUNITY annotation)
2. Fork VERSION_4_0 → create VERSION_5_0 with PREVIEW annotation
3. Copy XSD file: `wildfly-elytron-oidc-client_4_0.xsd` → `wildfly-elytron-oidc-client_5_0.xsd`
4. Update namespace in new XSD to `5.0`
5. Add new element to XSD
6. Create parser for VERSION_5_0
7. No need to check other stability levels

**Case C: No COMMUNITY schema exists either**
1. Fork the latest DEFAULT schema (always exists)
2. Create new version with PREVIEW annotation
3. Follow same steps as Case B

##### Example 2: Adding a COMMUNITY Feature

**Scenario**: You want to add `client-authentication-method` at COMMUNITY stability.

**Step 1: Check for current COMMUNITY schema**
```bash
# Find the highest version with @Stability(StabilityLevel.COMMUNITY) or no annotation
```

**Step 2: Check for lower stability level schemas**
```bash
# Check if PREVIEW or EXPERIMENTAL schemas exist
# These will ALSO need to be updated since they are subsets
```

**Case A: Current COMMUNITY schema exists, and PREVIEW schema exists**
1. Fork COMMUNITY schema (e.g., VERSION_4_0 → VERSION_5_0)
2. Add new element to COMMUNITY XSD and parser
3. **Also fork PREVIEW schema** (e.g., VERSION_3_0 → VERSION_4_0)
4. Add same new element to PREVIEW XSD and parser
5. Both schemas now support the new feature

**Case B: Current COMMUNITY schema exists, no PREVIEW schema**
1. Fork COMMUNITY schema only
2. Add new element to COMMUNITY XSD and parser
3. No lower stability schemas to update

**Case C: No COMMUNITY schema exists**
1. Fork DEFAULT schema → create COMMUNITY schema
2. Check for PREVIEW/EXPERIMENTAL schemas
3. If they exist, fork them too and add the feature

##### Example 3: Adding a DEFAULT Feature

**Scenario**: You want to add `ssl-required` at DEFAULT stability.

**Step 1: Fork DEFAULT schema** (always exists)
1. Fork current DEFAULT schema (e.g., VERSION_2_0 → VERSION_3_0)
2. Add new element to DEFAULT XSD and parser

**Step 2: Check ALL lower stability levels**
1. Check for COMMUNITY schema → if exists, fork and add feature
2. Check for PREVIEW schema → if exists, fork and add feature
3. Check for EXPERIMENTAL schema → if exists, fork and add feature

**Result**: All stability levels now support the new feature (since higher levels are subsets of lower)

#### Schema Forking Checklist

When forking a schema from one stability level to create a schema at a different stability level:

- [ ] Copy the XSD file and update namespace version
- [ ] Update `targetNamespace` in XSD to new version
- [ ] Add new schema version constant to `ElytronOidcSubsystemSchema.java`
- [ ] Add or update `@Stability` annotation to match target stability level
- [ ] Update `CURRENT` if this is the new current version
- [ ] Create or update parser class
- [ ] Register parser in `ElytronOidcExtension.java`
- [ ] Add test resources for new schema version
- [ ] Update test cases

#### Why This Matters

**Scenario**: Two developers are working on features for WildFly 41:
- Developer A: Adding `token-signature-algorithm` at PREVIEW
- Developer B: Adding `client-authentication-method` at COMMUNITY

**Without understanding the strategy**:
- Developer A creates PREVIEW schema 6.0
- Developer B creates COMMUNITY schema 6.0
- Conflict! Both used version 6.0 but for different stability levels

**With understanding the strategy**:
- Developer A checks: Current PREVIEW is 5.0, creates 6.0 at PREVIEW
- Developer B checks: Current COMMUNITY is 4.0, creates 5.0 at COMMUNITY
- Developer B also checks: PREVIEW 6.0 exists, so also creates PREVIEW 7.0 with the COMMUNITY feature
- No conflicts! Each stability level has independent version numbering

### Decision Tree: Do I Need a Schema Bump?

```
Is the XML configuration format changing?
├─ YES → Schema bump required
│  ├─ New elements/attributes? → Bump + annotate with feature's stability level
│  │                           → Check lower stability levels and bump those too
│  ├─ Removed elements/attributes? → Bump + document removal
│  ├─ Changed structure? → Bump + document change
│  └─ Promoting feature stability? → Bump + update annotation
│
└─ NO → Schema bump NOT required
   └─ Management-only changes → Model bump only
```

### Common Mistakes to Avoid

1. **Forgetting to bump schema when promoting stability**
   - ❌ Wrong: Promoting feature from PREVIEW to COMMUNITY without schema bump
   - ✅ Correct: New schema version with updated stability annotation

2. **Using wrong stability annotation**
   - ❌ Wrong: Adding PREVIEW feature but annotating schema as COMMUNITY
   - ✅ Correct: Schema annotation matches the lowest stability level of new features

3. **Bumping schema without clear reason**
   - ❌ Wrong: "Bumping schema for WildFly 41" (no explanation)
   - ✅ Correct: "Bumping schema to promote token-signature-algorithm from PREVIEW to COMMUNITY"

4. **Not coordinating schema and model bumps**
   - ❌ Wrong: Bumping schema but forgetting to bump model version
   - ✅ Correct: Both schema and model versions bumped in coordinated commits

## Version Bump Process

### Step 1: Add New Schema Version Constant

**File**: `ElytronOidcSubsystemSchema.java`

**What to Check**:
- Current schema enum values and their namespace URIs
- The `CURRENT` constant pointing to the latest version
- Version numbering pattern (major.minor)
- Stability level annotations on each version

**What to Do**:
1. Add a new enum constant following the existing pattern
2. Define the namespace URI for the new version
3. Include a comment indicating the target WildFly version
4. Add appropriate stability level annotation
5. Update the `CURRENT` constant to point to the new version

**Example Pattern**:
```java
enum ElytronOidcSubsystemSchema implements PersistentSubsystemSchema<ElytronOidcSubsystemSchema> {
    VERSION_1_0(1, Stability.DEFAULT),
    VERSION_2_0(2, Stability.DEFAULT),
    VERSION_2_0_PREVIEW(2, 0, Stability.PREVIEW), // WildFly 32.0-present
    VERSION_3_0_PREVIEW(3, 0, Stability.PREVIEW), // WildFly 33.0-present
    VERSION_4_0_PREVIEW(4, 0, Stability.PREVIEW), // WildFly 40.0-present
    VERSION_2_0_COMMUNITY(2, 0, Stability.COMMUNITY), // WildFly 41.0-present <-- NEW
    ;

    // CRITICAL: Update CURRENT map to include new schema version
    static final Map<Stability, ElytronOidcSubsystemSchema> CURRENT =
        Feature.map(EnumSet.of(VERSION_4_0_PREVIEW, VERSION_2_0_COMMUNITY, VERSION_2_0)); // <-- UPDATE

    private final VersionedNamespace<IntVersion, ElytronOidcSubsystemSchema> namespace;

    ElytronOidcSubsystemSchema(int major, Stability stability) {
        this.namespace = SubsystemSchema.createSubsystemURN(
            ElytronOidcExtension.SUBSYSTEM_NAME,
            stability,
            new IntVersion(major)
        );
    }

    ElytronOidcSubsystemSchema(int major, int minor, Stability stability) {
        this.namespace = SubsystemSchema.createSubsystemURN(
            ElytronOidcExtension.SUBSYSTEM_NAME,
            stability,
            new IntVersion(major, minor)
        );
    }

    @Override
    public VersionedNamespace<IntVersion, ElytronOidcSubsystemSchema> getNamespace() {
        return this.namespace;
    }
}
```

**Critical: Understanding the CURRENT Map**

The `CURRENT` map is **NOT** a single version pointer - it's a map of the current schema version for each stability level:

```java
static final Map<Stability, ElytronOidcSubsystemSchema> CURRENT =
    Feature.map(EnumSet.of(VERSION_4_0_PREVIEW, VERSION_2_0_COMMUNITY, VERSION_2_0));
```

**What This Means**:
- **PREVIEW** → Uses `VERSION_4_0_PREVIEW` when server runs at PREVIEW stability
- **COMMUNITY** → Uses `VERSION_2_0_COMMUNITY` when server runs at COMMUNITY stability
- **DEFAULT** → Uses `VERSION_2_0` when server runs at DEFAULT stability
- **EXPERIMENTAL** → Not listed (no current EXPERIMENTAL schema)

**Why This Matters**:
- When the server persists configuration, it uses the schema version for its current stability level
- If a stability level isn't in the CURRENT map, it falls back to the next higher stability level
- Example: If running at COMMUNITY but no COMMUNITY schema exists, it uses DEFAULT schema

**When Adding a New Schema Version**:
1. Add the new version constant to the enum
2. **Update the CURRENT map** to include the new version in the EnumSet
3. The map automatically associates each version with its stability level

**When Removing a Schema Version from CURRENT**:

**Critical Rule**: Once ALL features at a stability level have been promoted to a higher stability level, you **MUST** remove that schema version from the CURRENT map.

**Why This Matters**:
- Keeping obsolete schemas in CURRENT wastes resources
- Server would persist using a schema that has no unique features
- Indicates to developers that this stability level is no longer actively used

**Example Scenario**:
```java
// Before: PREVIEW has unique features
static final Map<Stability, ElytronOidcSubsystemSchema> CURRENT =
    Feature.map(EnumSet.of(VERSION_4_0_PREVIEW, VERSION_2_0_COMMUNITY, VERSION_2_0));

// All PREVIEW features promoted to COMMUNITY
// PREVIEW schema no longer needed

// After: Remove PREVIEW from CURRENT
static final Map<Stability, ElytronOidcSubsystemSchema> CURRENT =
    Feature.map(EnumSet.of(VERSION_2_0_COMMUNITY, VERSION_2_0));
```

**When to Remove**:
- ✅ All features unique to that stability level have been promoted
- ✅ The schema at that level is identical to the next higher stability level
- ✅ No active development is adding new features at that stability level

**When NOT to Remove**:
- ❌ Even one feature remains at that stability level
- ❌ Active development is adding features at that stability level
- ❌ The schema differs from the next higher stability level

**PR Review Checkpoint**:
- When reviewing feature promotion PRs, check if this is the LAST feature at a stability level
- If yes, verify that the CURRENT map is updated to remove that stability level
- This is easy to miss but important for maintenance

**Example: Complete Promotion Cycle**

**Phase 1: PREVIEW features exist**
```java
VERSION_4_0_PREVIEW(4, 0, Stability.PREVIEW), // Has token-signature-algorithm
VERSION_2_0_COMMUNITY(2, 0, Stability.COMMUNITY),
VERSION_2_0(2, Stability.DEFAULT),

CURRENT = Feature.map(EnumSet.of(VERSION_4_0_PREVIEW, VERSION_2_0_COMMUNITY, VERSION_2_0));
```

**Phase 2: Promote token-signature-algorithm to COMMUNITY**
```java
VERSION_4_0_PREVIEW(4, 0, Stability.PREVIEW), // Now empty, same as COMMUNITY
VERSION_3_0_COMMUNITY(3, 0, Stability.COMMUNITY), // Now has token-signature-algorithm
VERSION_2_0(2, Stability.DEFAULT),

// MUST remove VERSION_4_0_PREVIEW from CURRENT since it's now empty
CURRENT = Feature.map(EnumSet.of(VERSION_3_0_COMMUNITY, VERSION_2_0));
```

**Phase 3: Later, promote to DEFAULT**
```java
VERSION_4_0_PREVIEW(4, 0, Stability.PREVIEW), // Historical
VERSION_3_0_COMMUNITY(3, 0, Stability.COMMUNITY), // Now empty, same as DEFAULT
VERSION_3_0(3, Stability.DEFAULT), // Now has token-signature-algorithm

// MUST remove VERSION_3_0_COMMUNITY from CURRENT since it's now empty
CURRENT = Feature.map(EnumSet.of(VERSION_3_0));
```

**Considerations**:
- **Always bump the major version** (e.g., 5.0 → 6.0) unless explicitly instructed otherwise
- Minor version bumps are rare and should only be done for backward-compatible additions
- Stability level must be passed to constructor (not annotation like model versions)
- **Enum constant naming**: Include stability level in name for non-DEFAULT (e.g., `VERSION_2_0_COMMUNITY`, `VERSION_4_0_PREVIEW`)
- **Not all stability levels always present**: Only include in CURRENT map if that stability level has a current schema
- **Maintenance**: Remove schemas from CURRENT when all features promoted to higher stability

### Step 2: Create New Schema XSD File

**Location**: `src/main/resources/schema/`

**What to Check**:
- Existing XSD files and their naming pattern
- The most recent XSD file to use as a template
- Namespace URI in the XSD file

**XSD File Naming Convention**:
- DEFAULT stability: `wildfly-elytron-oidc-client_X_Y.xsd` (e.g., `wildfly-elytron-oidc-client_2_0.xsd`)
- COMMUNITY stability: `wildfly-elytron-oidc-client_community_X_Y.xsd` (e.g., `wildfly-elytron-oidc-client_community_2_0.xsd`)
- PREVIEW stability: `wildfly-elytron-oidc-client_preview_X_Y.xsd` (e.g., `wildfly-elytron-oidc-client_preview_4_0.xsd`)
- EXPERIMENTAL stability: `wildfly-elytron-oidc-client_experimental_X_Y.xsd` (if used)

**Namespace URI Convention**:
The namespace URI is automatically generated by `SubsystemSchema.createSubsystemURN()` based on the stability level:
- DEFAULT: `urn:wildfly:elytron-oidc-client:X.Y` (e.g., `urn:wildfly:elytron-oidc-client:2.0`)
- COMMUNITY: `urn:wildfly:elytron-oidc-client:community:X.Y` (e.g., `urn:wildfly:elytron-oidc-client:community:2.0`)
- PREVIEW: `urn:wildfly:elytron-oidc-client:preview:X.Y` (e.g., `urn:wildfly:elytron-oidc-client:preview:4.0`)
- EXPERIMENTAL: `urn:wildfly:elytron-oidc-client:experimental:X.Y` (if used)

**Important**: The stability level is inserted between the subsystem name and version number for non-DEFAULT schemas.

**Determining the Correct Source Schema to Copy**:

**Critical Rule for COMMUNITY/PREVIEW schemas**: Before copying, check if the previous schema at the same stability level is still in the `CURRENT` map.

- **If previous schema NOT in CURRENT** → Features were promoted to DEFAULT → Copy from current DEFAULT schema
- **If previous schema IS in CURRENT** → Previous schema still active → Copy from previous schema at same stability level

**Example**: Creating COMMUNITY 19.0 schema
```java
// Check ElytronSubsystemSchema.java
static final Map<Stability, ElytronSubsystemSchema> CURRENT = 
    Feature.map(EnumSet.of(VERSION_19_0, VERSION_19_0_COMMUNITY));
// VERSION_18_0_COMMUNITY is NOT in CURRENT → features were promoted

// Therefore:
// ✅ Copy from VERSION_19_0 (DEFAULT) - contains promoted features
// ❌ Do NOT copy from VERSION_18_0_COMMUNITY - would lose promotions
```

**Why This Matters**:
- Features in COMMUNITY schemas are promoted to DEFAULT over time
- Once promoted, the COMMUNITY schema is removed from CURRENT
- New COMMUNITY schema must include all DEFAULT features plus any new COMMUNITY features
- Copying from old COMMUNITY would drop the promoted features

**Verification Steps**:
1. Check `CURRENT` map in `*SubsystemSchema.java`
2. Look for your target stability level's previous version
3. If missing → copy from current DEFAULT
4. If present → copy from previous version at same stability

**What to Do**:
1. Determine correct source schema using the rule above
2. Copy the source XSD file (e.g., `wildfly-elytron-oidc-client_5_0.xsd` or current DEFAULT)
3. Rename following the naming convention for the target stability level
3. Update the `targetNamespace` attribute to match the namespace URI convention
4. Update the default namespace (`xmlns`) to match the targetNamespace
5. Update any schema documentation/comments to reference the new version
6. Make any necessary structural changes to the schema (this is where XML format changes are defined)

**Example XSD Headers**:

**DEFAULT Stability**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:wildfly:elytron-oidc-client:3.0"
           xmlns="urn:wildfly:elytron-oidc-client:3.0"
           elementFormDefault="qualified"
           attributeFormDefault="unqualified"
           version="1.0">
    <!-- Schema content -->
</xs:schema>
```

**COMMUNITY Stability**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:wildfly:elytron-oidc-client:community:2.0"
           xmlns="urn:wildfly:elytron-oidc-client:community:2.0"
           elementFormDefault="qualified"
           attributeFormDefault="unqualified"
           version="1.0">
    <!-- Schema content -->
</xs:schema>
```

**PREVIEW Stability**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:wildfly:elytron-oidc-client:preview:5.0"
           xmlns="urn:wildfly:elytron-oidc-client:preview:5.0"
           elementFormDefault="qualified"
           attributeFormDefault="unqualified"
           version="1.0">
    <!-- Schema content -->
</xs:schema>
```

**Important Notes**:
- The `targetNamespace` must exactly match the namespace URI from `ElytronOidcSubsystemSchema`
- If this is a pure version bump with no XML format changes, the schema content can be identical to the previous version
- If XML format changes are being made, update the XSD accordingly (new elements, attributes, types, etc.)

### Step 3: Create or Update Parser Class

**Location**: `src/main/java/org/wildfly/extension/elytron/oidc/`

**What to Check**:
- Existing parser classes (e.g., `ElytronOidcSubsystemParser_5_0.java`)
- Whether the new schema requires a new parser or can reuse an existing one
- Parser registration in `ElytronOidcExtension.java`

**What to Do**:

**Option A: Schema Changes Require New Parser**
1. Create a new parser class (e.g., `ElytronOidcSubsystemParser_6_0.java`)
2. Extend the previous parser or implement from scratch
3. Implement parsing logic for any new XML elements/attributes
4. Register the new parser in `ElytronOidcExtension.java`

**Option B: No Schema Changes (Pure Version Bump with Explicit Registration)**
1. Reuse the existing parser class
2. Update the parser registration to handle the new schema version

**Option C: No Schema Changes (Pure Version Bump with Automatic Routing)**

Some subsystems use parser routing methods that automatically handle new schema versions through `since()` checks.

**No changes needed** if:
- Parser routing uses `since()` to check schema version
- New schema version is higher than the check
- Example: `if (this.since(VERSION_19_0))` automatically includes VERSION_19_0_COMMUNITY

**Example** (from Elytron subsystem):
```java
private void addTlsParser(PersistentResourceXMLBuilder builder) {
    TlsParser tlsParser = new TlsParser();
    if (this.since(ElytronSubsystemSchema.VERSION_19_0)) {
        builder.addChild(tlsParser.tlsParser_19_0);
        // ↑ This automatically applies to VERSION_19_0_COMMUNITY too
        // because VERSION_19_0_COMMUNITY.since(VERSION_19_0) returns true
    } else if (this.since(ElytronSubsystemSchema.VERSION_18_0_COMMUNITY) 
               && this.enables(getDynamicClientSSLContextDefinition())) {
        builder.addChild(tlsParser.tlsParserCommunity_18_0);
    }
    // ...
}
```

**When to use which option**:
- **Option A**: XML format changed → new elements/attributes need custom parsing
- **Option B**: Pure bump, manual registration → register new version to existing parser
- **Option C**: Pure bump, automatic routing → no parser changes at all

**Example New Parser Class**:
```java
public class ElytronOidcSubsystemParser_6_0 extends ElytronOidcSubsystemParser_5_0 {

    @Override
    public void readElement(XMLExtendedStreamReader reader, List<ModelNode> operations)
            throws XMLStreamException {
        // If extending previous parser, call super for unchanged behavior
        super.readElement(reader, operations);

        // Add parsing logic for new elements/attributes here
    }

    // Override other methods as needed for schema changes
}
```

**Example Parser Reuse** (in `ElytronOidcSubsystemSchema.java`):
```java
@Override
public PersistentResourceXMLDescription getXMLDescription() {
    // If schema structure is unchanged, reuse previous parser
    return VERSION_5_0.getXMLDescription();
}
```

### Step 4: Register Parser in Extension

**File**: `ElytronOidcExtension.java`

**What to Check**:
- The `initializeParsers()` method that registers schema parsers
- Existing parser registrations

**What to Do**:
1. Add registration for the new schema version
2. Associate it with the appropriate parser class

**Example Pattern**:
```java
@Override
public void initializeParsers(ExtensionParsingContext context) {
    context.setSubsystemXmlMapping(SUBSYSTEM_NAME,
        ElytronOidcSubsystemSchema.VERSION_1_0.getNamespace().getUri(),
        ElytronOidcSubsystemParser_1_0::new);
    context.setSubsystemXmlMapping(SUBSYSTEM_NAME,
        ElytronOidcSubsystemSchema.VERSION_2_0.getNamespace().getUri(),
        ElytronOidcSubsystemParser_2_0::new);
    // ... existing registrations ...
    context.setSubsystemXmlMapping(SUBSYSTEM_NAME,
        ElytronOidcSubsystemSchema.VERSION_5_0.getNamespace().getUri(),
        ElytronOidcSubsystemParser_5_0::new);
    context.setSubsystemXmlMapping(SUBSYSTEM_NAME,
        ElytronOidcSubsystemSchema.VERSION_6_0.getNamespace().getUri(),
        ElytronOidcSubsystemParser_6_0::new);  // <-- ADD
}
```

**Alternative Pattern** (if reusing parser):
```java
context.setSubsystemXmlMapping(SUBSYSTEM_NAME,
    ElytronOidcSubsystemSchema.VERSION_6_0.getNamespace().getUri(),
    ElytronOidcSubsystemParser_5_0::new);  // <-- Reuse existing parser
```

### Step 5: Update Schema Writer

**File**: `ElytronOidcSubsystemSchema.java` or dedicated writer class

**What to Check**:
- How the current schema version writes XML configuration
- Whether a new writer is needed or existing writer can be reused

**What to Do**:
1. Implement or update the `PersistentResourceXMLDescription` for the new schema version
2. Ensure the writer outputs XML that validates against the new XSD
3. Handle any new attributes or elements introduced in the schema

**Example Pattern**:
```java
@Override
public PersistentResourceXMLDescription getXMLDescription() {
    PersistentResourceXMLBuilder builder = PersistentResourceXMLBuilder.Factory
        .create(ElytronOidcSubsystemDefinition.INSTANCE);

    // Configure XML writing for subsystem elements
    builder.addChild(/* secure-deployment configuration */);
    builder.addChild(/* secure-server configuration */);
    builder.addChild(/* provider configuration */);
    builder.addChild(/* realm configuration */);

    return builder.build();
}
```

### Step 6: Create Test XML File

**Location**: `src/test/resources/org/wildfly/extension/elytron/oidc/`

**What to Check**:
- Existing test XML files and their naming pattern
- The most recent test XML file to use as a template

**Test XML File Naming Convention**:
- DEFAULT stability: `elytron-oidc-client-X.Y.xml` (e.g., `elytron-oidc-client-2.0.xml`)
- COMMUNITY stability: `elytron-oidc-client-community-X.Y.xml` (e.g., `elytron-oidc-client-community-2.0.xml`)
- PREVIEW stability: `elytron-oidc-client-preview-X.Y.xml` (e.g., `elytron-oidc-client-preview-4.0.xml`)
- EXPERIMENTAL stability: `elytron-oidc-client-experimental-X.Y.xml` (if used)

**What to Do**:

**CRITICAL**: Always copy an existing test XML file and update only the namespace. Never write test XML from scratch.

1. **Copy the appropriate source test XML file**:
   - For pure version bumps within same stability: Copy the previous version at that stability level
   - For forking across stability levels: Copy from the source stability level you're forking from
   - Example: Creating COMMUNITY 2.0 from DEFAULT 2.0 → Copy `elytron-oidc-client-2.0.xml`

2. **Rename following the naming convention** for the target stability level

3. **Update ONLY the namespace** in the `<subsystem>` element to match the new schema's namespace URI
   - Use `sed` or similar tool to ensure accurate replacement
   - Example: `sed -i 's|urn:wildfly:elytron-oidc-client:2\.0|urn:wildfly:elytron-oidc-client:community:2.0|g' filename.xml`

4. **Do NOT modify the test content** unless you're adding new features that require testing

**Why This Matters**:
- Copying ensures you get the correct XML structure (attributes vs elements, etc.)
- Copying preserves comprehensive test coverage
- Manual writing is error-prone and may miss important test cases
- The test content should match the schema capabilities exactly

**Example Commands**:

**Creating COMMUNITY 2.0 test from DEFAULT 2.0**:
```bash
# Copy the DEFAULT 2.0 test file
cp elytron-oidc-client-2.0.xml elytron-oidc-client-community-2.0.xml

# Update only the namespace
sed -i 's|urn:wildfly:elytron-oidc-client:2\.0|urn:wildfly:elytron-oidc-client:community:2.0|g' elytron-oidc-client-community-2.0.xml
```

**Creating PREVIEW 5.0 test from PREVIEW 4.0**:
```bash
# Copy the previous PREVIEW test file
cp elytron-oidc-client-preview-4.0.xml elytron-oidc-client-preview-5.0.xml

# Update only the namespace version
sed -i 's|urn:wildfly:elytron-oidc-client:preview:4\.0|urn:wildfly:elytron-oidc-client:preview:5.0|g' elytron-oidc-client-preview-5.0.xml
```

**Result - Only Namespace Changes**:

**DEFAULT Stability**:
```xml
<subsystem xmlns="urn:wildfly:elytron-oidc-client:2.0">
    <!-- All test configuration from source file preserved -->
</subsystem>
```

**COMMUNITY Stability** (copied from DEFAULT):
```xml
<subsystem xmlns="urn:wildfly:elytron-oidc-client:community:2.0">
    <!-- Exact same test configuration as DEFAULT 2.0 -->
</subsystem>
```

**PREVIEW Stability**:
```xml
<subsystem xmlns="urn:wildfly:elytron-oidc-client:preview:5.0">
    <!-- All test configuration from PREVIEW 4.0 preserved -->
</subsystem>
```

**Important Notes**:
- The test framework (`AbstractSubsystemSchemaTest`) automatically discovers and tests ALL schema versions using `EnumSet.allOf(ElytronOidcSubsystemSchema.class)`
- Each schema version MUST have a corresponding test XML file
- The test XML file name must match the schema version and stability level
- The namespace in the test XML must exactly match the schema's namespace URI

### Step 7: Update Test Resources and Test Cases (If Needed)

**Critical Understanding**: Many subsystems use a parameterized test approach that automatically tests ALL schema versions. When you add a new schema version, the test framework automatically picks it up.

**Subsystem Test Pattern Detection**:
- **Look for**: Test class extending `AbstractSubsystemSchemaTest`
- **Look for**: `@RunWith(Parameterized.class)` annotation
- **Look for**: `EnumSet.allOf(YourSubsystemSchema.class)` in test parameters

**If your subsystem uses this pattern**:
- ✅ Only create test XML file
- ✅ No test code changes needed
- ✅ Test automatically runs for new schema version

**If your subsystem does NOT use this pattern**:
- Check existing test structure
- May need to add explicit test methods
- See **Subsystem Variations** section below

#### Test Structure Overview (elytron-oidc-client example)

**Test Class**: `ElytronOidcClientSubsystemTestCase.java`
- Uses `@RunWith(Parameterized.class)` to run tests for all schema versions
- Automatically discovers all schema versions from `ElytronOidcSubsystemSchema` enum
- Each schema version is tested independently

**Test Resources Location**: `src/test/resources/org/wildfly/extension/elytron/oidc/`

**Naming Convention**:
- DEFAULT schemas: `elytron-oidc-client-X.Y.xml` (e.g., `elytron-oidc-client-2.0.xml`)
- PREVIEW schemas: `elytron-oidc-client-preview-X.Y.xml` (e.g., `elytron-oidc-client-preview-4.0.xml`)
- COMMUNITY schemas: `elytron-oidc-client-community-X.Y.xml` (would follow same pattern)
- EXPERIMENTAL schemas: `elytron-oidc-client-experimental-X.Y.xml` (would follow same pattern)

#### What Test Files Are Required

**For Each New Schema Version**, you need:

1. **Test XML file** with representative configuration
   - Must use the correct namespace URI for the schema version
   - Should include examples of all major configuration elements
   - Should demonstrate any new features added in this version

2. **No code changes to test class** (automatic discovery)
   - The parameterized test automatically includes your new schema version
   - No need to add new test methods

#### Step-by-Step: Adding Test Resources

**Step 1: Determine the test file name**

Based on stability level and version:
- DEFAULT 3.0 → `elytron-oidc-client-3.0.xml`
- COMMUNITY 2.0 → `elytron-oidc-client-community-2.0.xml`
- PREVIEW 5.0 → `elytron-oidc-client-preview-5.0.xml`
- EXPERIMENTAL 1.0 → `elytron-oidc-client-experimental-1.0.xml`

**Step 2: Copy from the most similar existing test file**

Choose the source file based on your scenario:
- **Same stability level, incremented version**: Copy from previous version at same stability
  - Example: Creating PREVIEW 5.0 → Copy `elytron-oidc-client-preview-4.0.xml`
- **Forking to new stability level**: Copy from the source stability level
  - Example: Creating COMMUNITY 2.0 from DEFAULT 2.0 → Copy `elytron-oidc-client-2.0.xml`

**Step 3: Update the namespace URI**

```xml
<!-- Old (PREVIEW 4.0) -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:preview:4.0">

<!-- New (PREVIEW 5.0) -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:preview:5.0">
```

**Namespace URI Format**:
- DEFAULT: `urn:wildfly:elytron-oidc-client:X.Y`
- PREVIEW: `urn:wildfly:elytron-oidc-client:preview:X.Y`
- COMMUNITY: `urn:wildfly:elytron-oidc-client:community:X.Y`
- EXPERIMENTAL: `urn:wildfly:elytron-oidc-client:experimental:X.Y`

**Step 4: Add/modify configuration for new features**

If you're adding new XML elements/attributes:
```xml
<subsystem xmlns="urn:wildfly:elytron-oidc-client:preview:5.0">
    <secure-deployment name="myApp.war">
        <client-id>my-app</client-id>
        <!-- NEW FEATURE: Add the new element -->
        <token-signature-algorithm>RS256</token-signature-algorithm>
    </secure-deployment>
</subsystem>
```

**Step 5: Ensure comprehensive test coverage**

Your test XML should include:
- Examples of all major configuration elements (`realm`, `provider`, `secure-deployment`, `secure-server`)
- Various credential types (`secret`, `jwt`, `secret-jwt`)
- Different configuration patterns (inline realm, provider reference, etc.)
- Any new features introduced in this schema version

#### Example: Adding PREVIEW 5.0 Test File

**Scenario**: Adding `token-signature-algorithm` feature at PREVIEW stability

**File**: `src/test/resources/org/wildfly/extension/elytron/oidc/elytron-oidc-client-preview-5.0.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<subsystem xmlns="urn:wildfly:elytron-oidc-client:preview:5.0">
    <realm name="main">
        <auth-server-url>http://localhost:8080/auth</auth-server-url>
        <ssl-required>EXTERNAL</ssl-required>
    </realm>

    <provider name="keycloak">
        <provider-url>http://localhost:8080/realms/WildFly</provider-url>
    </provider>

    <secure-deployment name="myApp.war">
        <realm>main</realm>
        <resource>myAppId</resource>
        <credential name="secret" secret="0aa31d98-e0aa-404c-b6e0-e771dba1e798" />
        <!-- NEW FEATURE in 5.0 -->
        <token-signature-algorithm>RS256</token-signature-algorithm>
    </secure-deployment>

    <secure-deployment name="anotherApp.war">
        <client-id>another-app</client-id>
        <provider>keycloak</provider>
        <credential name="secret" secret="password" />
        <!-- NEW FEATURE in 5.0 -->
        <token-signature-algorithm>HS256</token-signature-algorithm>
    </secure-deployment>
</subsystem>
```

#### Example: Adding COMMUNITY 2.0 Test File (Forked from DEFAULT)

**Scenario**: Forking DEFAULT 2.0 to create COMMUNITY 2.0

**File**: `src/test/resources/org/wildfly/extension/elytron/oidc/elytron-oidc-client-community-2.0.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- Forked from DEFAULT 2.0, now at COMMUNITY stability -->
<subsystem xmlns="urn:wildfly:elytron-oidc-client:community:2.0">
    <!-- Copy configuration from elytron-oidc-client-2.0.xml -->
    <!-- Update namespace URI to community:2.0 -->
    <realm name="main">
        <auth-server-url>http://localhost:8080/auth</auth-server-url>
    </realm>
    <!-- ... rest of configuration ... -->
</subsystem>
```

#### Verification Steps

After adding test resources:

1. **Run the subsystem test**:
   ```bash
   mvn test -Dtest=ElytronOidcClientSubsystemTestCase
   ```

2. **Verify automatic discovery**:
   - The test should run for your new schema version
   - Look for test output showing your schema version being tested
   - Example: `Testing schema: VERSION_5_0`

3. **Check for parsing errors**:
   - Ensure XML parses correctly
   - Verify namespace URI is recognized
   - Confirm all elements validate against XSD

4. **Verify round-trip**:
   - Test reads XML → creates model → writes XML
   - Output XML should match input XML structure
   - Namespace URI should be preserved

#### Common Test Issues and Solutions

**Issue**: Test file not found
- **Cause**: Incorrect file naming or location
- **Solution**: Verify file name matches pattern and is in `src/test/resources/org/wildfly/extension/elytron/oidc/`

**Issue**: Namespace URI not recognized
- **Cause**: Mismatch between XSD namespace and test XML namespace
- **Solution**: Ensure test XML namespace exactly matches XSD `targetNamespace`

**Issue**: Parser not registered
- **Cause**: Parser not registered in `ElytronOidcExtension.initializeParsers()`
- **Solution**: Add parser registration (see Step 4 above)

**Issue**: Elements not parsing correctly
- **Cause**: Parser doesn't handle new elements
- **Solution**: Update parser to handle new XML elements

#### Test File Checklist

When adding a new schema version test file:

- [ ] File name follows naming convention (includes stability level if not DEFAULT)
- [ ] File located in `src/test/resources/org/wildfly/extension/elytron/oidc/`
- [ ] Namespace URI matches schema version and stability level
- [ ] Includes representative examples of all major configuration elements
- [ ] Demonstrates any new features added in this version
- [ ] XML is well-formed and valid
- [ ] Test runs successfully with `mvn test -Dtest=ElytronOidcClientSubsystemTestCase`
- [ ] Round-trip test passes (read → model → write)

#### No Changes Needed

**These files typically do NOT need changes** when adding a schema version:

- `ElytronOidcClientSubsystemTestCase.java` - Uses automatic discovery
- `ElytronOidcClientSubsystemTransformerTestCase.java` - Only updated when adding transformer tests (model version related)
- Other test classes - Unless they specifically test schema-related functionality

## Verification Steps

After making the changes, verify:

1. **Compilation**: The code compiles without errors
2. **XSD Validation**: The new XSD file is well-formed and valid
3. **Namespace Consistency**: All namespace URIs match across files
4. **Parser Registration**: The new schema version is properly registered
5. **Test Coverage**: Tests exist for the new schema version
6. **Backward Compatibility**: Old schema versions can still be parsed
7. **Schema Migration**: Old configurations can be read and written in new format

## Common Patterns and Best Practices

### Pattern 1: Pure Schema Version Bump (Standard Practice)

**This is the recommended approach**: Perform the schema version bump as a separate, isolated change.

When bumping the schema version without any XML format changes:
- Add the new schema version constant
- Update `CURRENT`
- Copy the previous XSD file to the new version (update namespace only)
- Reuse the existing parser
- This reserves the schema version number for future use

**Commit Strategy**:
- **Minimum**: Schema bump should be in its own commit to make PR review easier
- **Recommended**: Create a dedicated topic branch for the schema bump and publish to `https://github.com/wildfly-security-incubator/wildfly`
  - This allows others to share the commit without duplication
  - Keeps the schema bump isolated and reusable
  - Makes it easier to coordinate changes across multiple contributors

### Pattern 2: Schema Bump with XML Format Changes (Separate Steps)

**Important**: Avoid combining schema version bump and XML format changes in the same commit.

**Recommended Workflow**:
1. **Step 1**: Create schema version bump topic branch
   - Perform pure schema version bump (Pattern 1)
   - Commit and publish to wildfly-security-incubator
   - This can be shared by others working on related changes
2. **Step 2**: Create XML format changes branch
   - Base on the schema version bump branch
   - Update XSD with new elements/attributes
   - Create or update parser to handle new XML format
   - Update writer to output new XML format
   - Document the XML format changes

**Why Separate**:
- Easier PR review (version bump is mechanical, format changes need scrutiny)
- Schema version bump can be shared across multiple feature branches
- Cleaner git history
- Easier to revert if needed

### Pattern 3: Schema Bump for Stability Level Promotion (Separate Steps)

**Important**: Stability level promotions should follow the same separation pattern as other schema bumps.

When promoting features to a higher stability level:

**Recommended Workflow**:
1. **Step 1**: Create schema version bump topic branch
   - Add new schema version constant with updated stability annotation
   - Update `CURRENT`
   - Copy XSD file with updated namespace (content may be identical)
   - Reuse existing parser if XML format unchanged
   - Commit and publish to wildfly-security-incubator
   - This can be shared by others working on related stability promotions
2. **Step 2**: If needed, create feature changes branch
   - Base on the schema version bump branch
   - Make any additional feature modifications
   - Document the stability promotion

**Why This Matters**:
- Multiple developers may be promoting different features to the same stability level
- Sharing the schema version bump commit avoids conflicts
- Makes it clear which changes are mechanical (version bump) vs. substantive (feature changes)
- Easier to coordinate when multiple PRs target the same WildFly release

**Example Scenario**:
- Developer A is promoting `token-signature-algorithm` from PREVIEW to COMMUNITY
- Developer B is promoting `client-authentication-method` from PREVIEW to COMMUNITY
- Both can share the same schema version bump commit (6.0 with COMMUNITY annotation)
- Each developer's feature-specific changes go in separate commits on their own branches

## PR Review Guidelines

When reviewing PRs that modify the elytron-oidc-client subsystem schema:

### Schema Version Bump Verification Checklist

- [ ] **Check Last .Final Tag**: Verify the schema version in the last WildFly .Final release
  - Command: `git show <last-final-tag>:elytron-oidc-client/src/main/java/org/wildfly/extension/elytron/oidc/ElytronOidcSubsystemSchema.java`
  - If current main branch already has a higher version than the last .Final tag, the version is correct
  - If current main branch has the same version as the last .Final tag, a version bump is required
- [ ] **Verify Schema Bump is Separate**: If a schema version bump is included, it should be in its own commit
- [ ] **Check XSD File**: Ensure new XSD file exists with correct namespace URI
- [ ] **Validate Parser Registration**: Ensure the new schema version is registered in `ElytronOidcExtension`
- [ ] **Check Stability Annotations**: Verify stability level annotations match feature stability
- [ ] **Verify Test Coverage**: Ensure tests exist for the new schema version
- [ ] **No XML Format Changes Without Schema Bump**: Reject PRs that modify XML format without a corresponding schema version bump (unless the version was already bumped in main since the last .Final tag)

### Common Review Scenarios

**Scenario 1: PR adds new XML elements/attributes**
- ✅ Schema version must be bumped (unless already bumped in main since last .Final)
- ✅ XSD file must be updated with new elements/attributes
- ✅ Parser must be updated to handle new XML format
- ✅ Writer must be updated to output new XML format
- ✅ Schema version bump should be in separate commit

**Scenario 2: PR only bumps schema version (to wildfly-security-incubator)**
- ✅ Schema version bump commits should be published to wildfly-security-incubator for sharing
- ✅ XSD file can be identical to previous version (namespace updated)
- ✅ Parser can be reused from previous version
- ❌ **Do not merge bump-only PRs to WildFly main** - schema version bumps should only be merged when accompanied by at least one feature that modifies the XML format
- ✅ Other contributors can cherry-pick or rebase on the shared schema version bump commit

**Scenario 3: PR modifies XML format but schema version already bumped in main**
- ✅ Check that schema version in main is higher than last .Final tag
- ✅ XSD file must be updated for the changes
- ✅ Parser must be updated to handle the changes
- ✅ No additional schema version bump needed
- ❌ **Reject if PR includes redundant schema version bump** (version already bumped in main since last .Final)

**Scenario 4: Multiple PRs need schema version bump**
- ✅ First PR to merge should include the schema version bump
- ✅ Subsequent PRs should rebase on the schema version bump
- ✅ This is why publishing schema version bump to wildfly-security-incubator is important

### Review Rejection Criteria

**Reject if**:
- XML format changes without schema version bump (and version not already bumped in main)
- Schema version bump combined with XML format changes in same commit
- Missing XSD file for new schema version
- Incorrect namespace URI in XSD or schema constant
- Missing parser registration
- Incorrect stability level annotation
- Missing test coverage for new schema version

## Files Modified in a Schema Version Bump

### Always Modified (Pure Version Bump)
- **`ElytronOidcSubsystemSchema.java`** - Add new version constant, update CURRENT
- **New XSD file** - Copy previous version with updated namespace
- **`ElytronOidcExtension.java`** - Register parser for new schema version

### Modified When XML Format Changes
- **XSD file** - Update with new elements/attributes/types
- **Parser class** - Create new or update existing to handle new XML format
- **Writer implementation** - Update to output new XML format
- **Test XML files** - Add test resources for new schema version
- **Test classes** - Add test cases for new schema version

### NOT Modified in a Pure Schema Version Bump
- **`ElytronOidcClientSubsystemModel.java`** - Model versions are separate (though often bumped together)
- **`ElytronOidcSubsystemTransformers.java`** - Transformers are for model versions, not schema versions
- **Resource definition classes** - Unless XML format changes require it

## Relationship to Management Model Versions

**Critical Understanding**:
- **Schema versions and model versions are independent but related**
- Schema changes typically require model version bumps
- Model changes may not require schema version bumps (if XML format unchanged)

**Coordination Pattern**:
1. Determine if both schema and model versions need bumping
2. If both needed, bump them in separate commits on the same branch
3. Schema version bump commit first, then model version bump commit
4. This makes it clear which changes affect XML format vs. management API

## Relationship to Stability Levels

**See the comprehensive section**: [Understanding Stability Levels and Schema Bumps](#understanding-stability-levels-and-schema-bumps) above for detailed information on:
- The stability level hierarchy (Experimental → Preview → Community → Default)
- When schema bumps are required for stability promotions
- How to annotate schema versions with stability levels
- Decision trees and examples for different scenarios

**Quick Reference**:
- **Stability hierarchy**: Experimental → Preview → Community → Default (higher levels are subsets of lower)
- **Adding features at any level** → Schema bump required with appropriate annotation
- **Promoting features to higher stability** → Schema bump required with updated annotation
- **Schema annotation** indicates minimum stability level required to use that schema version

## Next Steps After Schema Version Bump

After completing the schema version bump:

1. **Model Version Bump**: If needed, bump the management model version separately
2. **XML Format Changes**: Update XSD, parser, and writer for any XML format changes
3. **Transformer Rules**: If model version was bumped, add transformation rules
4. **Testing**: Update and run tests to verify schema parsing and validation
5. **Documentation**: Update configuration guides and migration documentation

## Troubleshooting

### Issue: XSD Validation Errors

**Symptoms**: XML files don't validate against new XSD
**Solution**: Verify namespace URI matches exactly between XSD and schema constant

### Issue: Parser Not Found

**Symptoms**: Runtime errors about missing parser for schema version
**Solution**: Verify parser is registered in `ElytronOidcExtension.initializeParsers()`

### Issue: Namespace URI Mismatch

**Symptoms**: Configuration files not recognized or parsed incorrectly
**Solution**: Ensure namespace URI is consistent across:
- `ElytronOidcSubsystemSchema` enum constant
- XSD file `targetNamespace` attribute
- Parser registration in `ElytronOidcExtension`
- Test XML files

### Issue: Old Schema Versions Not Parsing

**Symptoms**: Backward compatibility broken for old configurations
**Solution**: Ensure all old schema version parsers remain registered and functional

## Reference Examples

### Weld Subsystem

The Weld subsystem provides a good reference example:
- **Schema**: `wildfly/weld/subsystem/src/main/resources/schema/jboss-as-weld_*.xsd`
- **Extension**: `wildfly/weld/subsystem/src/main/java/org/jboss/as/weld/WeldExtension.java`
- **Parsers**: `wildfly/weld/subsystem/src/main/java/org/jboss/as/weld/WeldSubsystem*Parser.java`

### XTS Subsystem

Another reference for simpler subsystems:
- **Location**: `wildfly/xts/`
- **Schema files**: `wildfly/xts/src/main/resources/schema/jboss-as-xts_*.xsd`

## Summary Checklist

For a pure schema version bump:

- [ ] Add new schema version constant to `ElytronOidcSubsystemSchema.java`
- [ ] Update `CURRENT` constant to point to new version
- [ ] Add WildFly version comment to new constant
- [ ] Add stability level annotation if needed
- [ ] Create new XSD file by copying previous version
- [ ] Update namespace URI in new XSD file
- [ ] Register parser for new schema version in `ElytronOidcExtension.java`
- [ ] Create or reuse parser class for new schema version
- [ ] Add test XML files for new schema version
- [ ] Update test cases to cover new schema version
- [ ] **Update `VERSIONS.md`** with the new schema version(s) and target WildFly version
- [ ] Verify compilation succeeds
- [ ] Run tests to verify schema parsing and validation
- [ ] Verify backward compatibility (old schemas still parse)
- [ ] Document the reason for the schema version bump

## Subsystem Variations

Different WildFly subsystems use different patterns for schema versioning. This section documents key variations.

### elytron-oidc-client Pattern (Primary Example)

**Characteristics**:
- Each schema version has its own parser class (e.g., `ElytronOidcSubsystemParser_5_0.java`)
- Parsers registered in `ElytronOidcExtension.initializeParsers()`
- Test class extends `AbstractSubsystemSchemaTest` with `@Parameterized`
- Tests auto-discover all schema versions via `EnumSet.allOf()`

**Pure Version Bump**:
- Create new parser class OR reuse via `getXMLDescription()` method
- Explicitly register in `initializeParsers()`

### elytron Subsystem Pattern

**Characteristics**:
- Parser routing through `addXxxParser()` methods using `since()` checks
- Parsers are fields in parser classes (e.g., `TlsParser.tlsParser_19_0`)
- Test class: `ElytronMixedStabilitySubsystemParsingTestCase` extends `AbstractSubsystemSchemaTest`
- Tests auto-discover schema versions via `EnumSet.allOf()`

**Pure Version Bump**:
- **No parser changes needed** - routing through `since()` automatically includes new version
- Example: `if (this.since(VERSION_19_0))` automatically handles `VERSION_19_0_COMMUNITY`

**Key Difference**: Parser reuse is automatic, not explicit

### VERSIONS.md Structure

**With Multiple Stability Levels**:

```markdown
## Schema Versions

Schema versions are maintained independently from model versions and may exist at different stability levels.

### DEFAULT Stability

| Schema Version | WildFly Version | Notes |
|----------------|-----------------|-------|
| 19.0           | 42.0+           | Promoted feature X to DEFAULT |
| 18.0           | 32.0+           | Added feature Y |

### COMMUNITY Stability

| Schema Version | WildFly Version | Notes |
|----------------|-----------------|-------|
| 19.0           | 42.0+           | Pure version bump for WildFly 42 features |
| 18.0           | 32.0+           | Added feature Z at COMMUNITY |

### PREVIEW Stability

No current PREVIEW schema versions.
```

**Single Stability Level** (all DEFAULT):

```markdown
## Schema Versions

| Schema Version | WildFly Version | Notes |
|----------------|-----------------|-------|
| 5.0            | 40.0+           | Added feature X |
| 4.0            | 33.0+           | Added feature Y |
```

### File Naming Patterns

**elytron-oidc-client**:
- Schema: `wildfly-elytron-oidc-client_X_Y.xsd`
- Schema (community): `wildfly-elytron-oidc-client_community_X_Y.xsd`
- Test XML: `elytron-oidc-client-X.Y.xml`
- Test XML (community): `elytron-oidc-client-community-X.Y.xml`

**elytron**:
- Schema: `wildfly-elytron_X_Y.xsd`
- Schema (community): `wildfly-elytron_community_X_Y.xsd`
- Test XML: `elytron-subsystem-X.Y.xml`
- Test XML (community): `elytron-subsystem-community-X.Y.xml`
- Old versions: `legacy-elytron-subsystem-X.Y.xml`

### Checklist Adjustments by Subsystem

**For elytron subsystem**:
- ❌ Skip "Create or update parser class" if pure version bump
- ❌ Skip "Register parser in Extension" if pure version bump
- ✅ Parser routing via `since()` handles it automatically

**For elytron-oidc-client subsystem**:
- ✅ Always update parser (create new or reuse via `getXMLDescription()`)
- ✅ Always register in `initializeParsers()`

**For both subsystems**:
- ✅ Test XML files always required
- ✅ Tests auto-discover via `EnumSet.allOf()`
- ✅ No test code changes needed

---

**Document Version**: 1.1
**Created**: 2026-05-27
**Last Updated**: 2026-09-02
**Updates**:
- 2026-09-02: Added Subsystem Variations section (elytron vs elytron-oidc-client patterns)
- 2026-09-02: Added critical schema inheritance rule for COMMUNITY schema creation
- 2026-09-02: Added parser reuse patterns (automatic via `since()` checks)
- 2026-09-02: Enhanced test auto-discovery documentation
- 2026-09-02: Added VERSIONS.md examples with multiple stability levels

**Related Docs**:
- `management-model-version-bump-guide.md` - Management model version bump guide