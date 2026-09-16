# Feature Implementation Guide for WildFly Subsystems

## Overview

This guide covers the implementation of new features in WildFly subsystems, focusing on the Elytron subsystem patterns learned from WFCORE-7193 (brute force protection configuration).

**Scope**: This guide covers feature implementation AFTER version bumps are complete. Version bumps are a prerequisite - see the version bump guides first.

**Relationship to Other Guides**:
- **Prerequisites**: Read `management-model-version-bump-guide.md` and `schema-version-bump-guide.md` first
- **Testing**: See `subsystem-schema-test-requirements.md` for test requirements
- **Workflow**: Version bumps → Feature implementation (this guide) → Testing

**Important Distinction**:
- **New features** (most common): Model attributes are the FIRST activation of the feature
- **Replacing system properties** (special case): Feature already exists via system properties, model adds management API (WFCORE-7193 pattern)

## Prerequisites: Version Bumps Come First

### Critical: Coordinate Version Bumps in Zulip

**BEFORE starting any version bump work:**

1. **Post in #wildfly-elytron channel on Zulip**
   - Announce you're planning version bumps for WildFly [version]
   - Check if anyone else is already working on bumps
   - Coordinate to create shared base branches

2. **Why This Matters**:
   - Version bumps create common base branches for multiple features
   - Multiple developers depend on these base branches
   - Duplicate bump work wastes effort and creates merge conflicts
   - Team members may be waiting for your bump to start their feature work

3. **Expected Outcome**:
   - One person creates the bump branches
   - Bump branches pushed to wildfly-security-incubator for team use
   - Other team members branch from the shared bumps for their features

### Check If Version Bumps Are Needed

**BEFORE coordinating in Zulip or starting any bump work**, check if bumps are needed:

**Model Version Check**:
1. Find last .Final WildFly tag: `git tag -l "*Final" | tail -5`
2. Check model version in that tag (see `management-model-version-bump-guide.md` "When Is a Model Version Bump Needed?" section)
3. Compare to current main branch
4. If **same** → bump needed
5. If **different** → already bumped, use existing bump branch

**Schema Version Check** (per stability level):
1. Find last .Final WildFly tag (same as above)
2. Check schema version for your target stability level in that tag (see `schema-version-bump-guide.md` "When Is a Schema Version Bump Needed?" section)
3. Compare to current main branch for that stability level
4. If **same** → bump needed for that stability level
5. If **different** → already bumped, use existing bump branch

**Critical**: Check model and schema versions INDEPENDENTLY - one may need bump while other doesn't!

### Version Bump Workflow

**Step 1: Model Version Bump** (if needed based on check above)
- Follow `management-model-version-bump-guide.md`
- Creates base branch (e.g., `WFCORE-7688`)
- Push to incubator for team sharing

**Step 2: Schema Version Bumps** (if needed based on check above, per stability level)
- Follow `schema-version-bump-guide.md` for each stability level
- Branch from model bump (not from each other)
- Creates `WFCORE-7690` (community schema) and `WFCORE-7691` (preview schema)
- Push both to incubator

**Step 3: Feature Implementation** (this guide)
- Branch from appropriate bump branch based on feature's stability level
- Implement feature following patterns below
- Feature PR targets the schema bump branch

**Example from WFCORE-7193**:
```
main
  └─ WFCORE-7688 (model 19→20 bump)
       ├─ WFCORE-7690 (community:19.0 schema)
       │    └─ WFCORE-7193 (brute force feature, community stability)
       └─ WFCORE-7691 (preview:19.0 schema)
            └─ WFCORE-XXXX (other preview feature)
```

### Verify Prerequisites Before Implementation

- [ ] **Checked last .Final tag** for model version (same = bump needed, different = use existing)
- [ ] **Checked last .Final tag** for schema version at target stability level (per stability check)
- [ ] Model version bump complete (if needed) or using existing bump branch
- [ ] Schema version bump complete for target stability level (if needed) or using existing bump branch
- [ ] Bump branches pushed to incubator (if you created them)
- [ ] Team aware of available base branches (Zulip post reviewed)
- [ ] You've branched from correct base for your feature's stability level
- [ ] No duplicate bump work in progress (confirmed via Zulip)

## Design Decisions

Before implementing, make key design decisions that affect the entire implementation.

### Decision 1: OBJECT Attribute vs Child Resource

**OBJECT Attribute Pattern** (WFCORE-7193 choice):
- Single attribute with nested fields
- XML renders as: `<element attr1="value1" attr2="value2"/>`
- Use when: Logically cohesive group of settings, no identity/name needed

```java
// WFCORE-7193 example
static final ObjectTypeAttributeDefinition BRUTE_FORCE_PROTECTION = 
    new ObjectTypeAttributeDefinition.Builder(
        ElytronDescriptionConstants.BRUTE_FORCE_PROTECTION,
        BF_ENABLED,           // nested SimpleAttributeDefinition
        BF_MAX_FAILED_ATTEMPTS,
        BF_LOCKOUT_INTERVAL,
        BF_SESSION_TIMEOUT,
        BF_MAX_CACHED_SESSIONS)
        .setRequired(false)
        .setStability(Stability.COMMUNITY)
        .setRestartAllServices()
        .build();
```

**Child Resource Pattern** (alternative):
- Separate child resource with `.addChild()`
- XML renders as: `<child-resource name="..."><settings/></child-resource>`
- Use when: Multiple named instances, independent lifecycle, resource operations

**Decision Factors**:
- Does it need a "name" attribute? → Child resource
- Single instance per parent? → OBJECT attribute
- Group of related settings? → OBJECT attribute
- Needs add/remove operations? → Child resource

### Decision 2: XML Structure - Attributes vs Elements

When using `ObjectTypeAttributeDefinition`, you must choose XML rendering:

**Attribute-Based (WFCORE-7193 choice)**:
```xml
<brute-force-protection enabled="true" max-failed-attempts="5" 
    lockout-interval="10" session-timeout="15" max-cached-sessions="1000"/>
```

**Implementation**:
- Parser: `AttributeParser.OBJECT_PARSER`
- Marshaller: `AttributeMarshaller.ATTRIBUTE_OBJECT`
- XSD: Uses `xs:attribute` entries

**Element-Based** (alternative):
```xml
<brute-force-protection>
    <enabled>true</enabled>
    <max-failed-attempts>5</max-failed-attempts>
    ...
</brute-force-protection>
```

**Implementation**:
- Parser: Custom `PersistentResourceXMLParser` subclass
- Marshaller: Custom marshaller
- XSD: Uses `xs:element` entries

**Decision Factors**:
- Simple scalar values? → Attributes (more compact)
- Complex nested structures? → Elements (more flexible)
- Existing subsystem patterns? → Match for consistency
- XSD constraints? → `xs:sequence` with `maxOccurs="unbounded"` requires elements

### Decision 3: Apply to Single vs Multiple Component Types

**Multiple Component Types** (WFCORE-7193 pattern):
- Same attribute applied to 10 realm types
- Shared attribute definition in common class (`RealmDefinitions.java`)
- Each component adds to its `ATTRIBUTES` array
- Parser routing handles all types

**Single Component Type**:
- Attribute defined in component's own definition class
- Only that component references it

**Decision Factors**:
- Is the feature conceptually applicable to multiple component types?
- Does the attribute have identical semantics across types?
- Would component-specific variants be confusing?

## String Handling

WildFly has strict patterns for string constants and user-facing descriptions.

### String Constants

**Rule**: ALL string literals used in code must be constants in the subsystem's description constants class.

**Location**: `[Subsystem]DescriptionConstants.java` (e.g., `ElytronDescriptionConstants.java`)

**Pattern**:
```java
// ElytronDescriptionConstants.java
public interface ElytronDescriptionConstants {
    // Existing constants...
    
    // WFCORE-7193 additions (6 constants)
    String BRUTE_FORCE_PROTECTION = "brute-force-protection";
    String BF_ENABLED = "enabled";
    String BF_MAX_FAILED_ATTEMPTS = "max-failed-attempts";
    String BF_LOCKOUT_INTERVAL = "lockout-interval";
    String BF_SESSION_TIMEOUT = "session-timeout";
    String BF_MAX_CACHED_SESSIONS = "max-cached-sessions";
}
```

**Usage**:
```java
// Use constants, NOT string literals
static final SimpleAttributeDefinition BF_ENABLED = 
    new SimpleAttributeDefinitionBuilder(
        ElytronDescriptionConstants.BF_ENABLED,  // ✅ Constant
        ModelType.BOOLEAN)
    // NOT: "enabled"  ❌ String literal
```

**Count Your Constants**:
- 1 for the OBJECT attribute name
- N for each nested field (5 in WFCORE-7193)
- Total: 6 constants for WFCORE-7193

### Resource Bundle Entries

**Rule**: EVERY attribute needs resource bundle entries for user-facing descriptions.

**Location**: `LocalDescriptions.properties` (or subsystem-specific name)

**Pattern for OBJECT Attributes**:
For each component type that uses the attribute, you need **7 entries** (1 + 5 + 1):
1. Component attribute itself (e.g., `elytron.properties-realm.brute-force-protection`)
2. Each nested field (5 entries: enabled, max-failed-attempts, etc.)
3. (Only for OBJECT attributes with 5+ fields, may need parent description)

**Example** (properties-realm, 7 entries):
```properties
# Parent OBJECT attribute
elytron.properties-realm.brute-force-protection=Configuration for brute force authentication protection.

# Nested fields (5 entries)
elytron.properties-realm.brute-force-protection.enabled=Whether brute force protection is enabled for this realm.
elytron.properties-realm.brute-force-protection.max-failed-attempts=Maximum number of failed authentication attempts before lockout. If not specified or set to -1, uses the WildFly Elytron library default (10).
elytron.properties-realm.brute-force-protection.lockout-interval=Duration in minutes to lock out an account after exceeding max failed attempts. If not specified or set to -1, uses the WildFly Elytron library default (15 minutes).
elytron.properties-realm.brute-force-protection.session-timeout=Duration in minutes before a failure session expires and attempts are forgotten. If not specified or set to -1, uses the WildFly Elytron library default (30 minutes).
elytron.properties-realm.brute-force-protection.max-cached-sessions=Maximum number of brute force sessions to cache. If not specified or set to -1, uses the WildFly Elytron library default (25000).
```

**WFCORE-7193 Count**: 10 realm types × 7 entries = **70 resource bundle entries**

### Resource Bundle Best Practices

1. **Include Library Defaults**: Tell users what -1 means
   - ✅ "If not specified or set to -1, uses the WildFly Elytron library default (10)."
   - ❌ "Maximum number of failed authentication attempts."

2. **Correct Units**: Match the actual implementation
   - ✅ "Duration in minutes"
   - ❌ "Duration in milliseconds" (when impl uses minutes)

3. **Consistent Wording**: Use same phrasing across similar components
   - All 10 realm types should have identical wording (differ only in realm name)

4. **Active Voice**: "Whether X is enabled" not "Enables or disables X"

### Error Message Strings

**Rule**: Error messages also need constants and may need resource bundle entries.

**Location Pattern**:
- Constants in `ElytronDescriptionConstants.java`
- Bundle entries in `LocalDescriptions.properties` if user-facing
- Internal exceptions can use inline strings for developer-facing messages

**Example** (if WFCORE-7193 had validation errors):
```java
// ElytronDescriptionConstants.java
String BF_INVALID_LOCKOUT = "invalid-lockout-interval";

// LocalDescriptions.properties
elytron.brute-force-protection.invalid-lockout-interval=Lockout interval must be -1 or positive value in minutes.

// Usage in code
throw new OperationFailedException(
    ElytronMessages.log.invalidValue(
        BF_LOCKOUT_INTERVAL, value, BF_INVALID_LOCKOUT));
```

## Implementation Patterns

### Pattern: ObjectTypeAttributeDefinition with Nested Fields

**Structure** (WFCORE-7193):
```java
// Step 1: Define nested field attributes (SimpleAttributeDefinition)
static final SimpleAttributeDefinition BF_ENABLED = 
    new SimpleAttributeDefinitionBuilder(
        ElytronDescriptionConstants.BF_ENABLED,
        ModelType.BOOLEAN)
        .setDefaultValue(ModelNode.FALSE)
        .setRequired(false)
        .setAllowExpression(true)  // Allow ${...} expressions
        .setRestartAllServices()
        .build();

static final SimpleAttributeDefinition BF_MAX_FAILED_ATTEMPTS = 
    new SimpleAttributeDefinitionBuilder(
        ElytronDescriptionConstants.BF_MAX_FAILED_ATTEMPTS,
        ModelType.INT)
        .setRequired(false)
        .setAllowExpression(true)
        .setValidator(new IntRangeValidator(-1, Integer.MAX_VALUE, true, true))
        .setRestartAllServices()
        .build();

static final SimpleAttributeDefinition BF_LOCKOUT_INTERVAL = 
    new SimpleAttributeDefinitionBuilder(
        ElytronDescriptionConstants.BF_LOCKOUT_INTERVAL,
        ModelType.INT)
        .setRequired(false)
        .setAllowExpression(true)
        .setMeasurementUnit(MeasurementUnit.MINUTES)  // ✅ Specify units!
        .setValidator(new IntRangeValidator(-1, Integer.MAX_VALUE, true, true))
        .setRestartAllServices()
        .build();

// ... (session-timeout, max-cached-sessions)

// Step 2: Create OBJECT attribute containing nested fields
static final ObjectTypeAttributeDefinition BRUTE_FORCE_PROTECTION = 
    new ObjectTypeAttributeDefinition.Builder(
        ElytronDescriptionConstants.BRUTE_FORCE_PROTECTION,
        BF_ENABLED,
        BF_MAX_FAILED_ATTEMPTS,
        BF_LOCKOUT_INTERVAL,
        BF_SESSION_TIMEOUT,
        BF_MAX_CACHED_SESSIONS)
        .setRequired(false)
        .setStability(Stability.COMMUNITY)  // ✅ Set stability!
        .setRestartAllServices()
        .build();
```

**Key Points**:
- Nested fields are `SimpleAttributeDefinition` (scalar types)
- Parent `ObjectTypeAttributeDefinition` contains them
- `.setAllowExpression(true)` on nested fields enables `${...}` syntax
- `.setStability()` on parent controls visibility across stability levels
- Nested fields inherit stability from parent
- `.setMeasurementUnit()` for time/size values (not just in descriptions!)

### Pattern: Apply to Multiple Component Types

**Challenge**: Add same attribute to 10 realm types without duplication.

**Solution** (WFCORE-7193):

1. **Define Once in Shared Class**:
```java
// RealmDefinitions.java (shared across all realm types)
public class RealmDefinitions {
    static final ObjectTypeAttributeDefinition BRUTE_FORCE_PROTECTION = ...;
    // Other shared definitions
}
```

2. **Add to Each Component's ATTRIBUTES Array**:
```java
// PropertiesRealmDefinition.java
import static org.wildfly.extension.elytron.RealmDefinitions.BRUTE_FORCE_PROTECTION;

static final AttributeDefinition[] ATTRIBUTES = new AttributeDefinition[] { 
    USERS_PROPERTIES, 
    GROUPS_PROPERTIES, 
    GROUPS_ATTRIBUTE, 
    HASH_ENCODING, 
    HASH_CHARSET, 
    BRUTE_FORCE_PROTECTION  // ✅ Static import
};
```

3. **Repeat for All Component Types**:
```java
// FileSystemRealmDefinition.java
import static org.wildfly.extension.elytron.RealmDefinitions.BRUTE_FORCE_PROTECTION;

static final AttributeDefinition[] ATTRIBUTES = new AttributeDefinition[] {
    PATH, RELATIVE_TO, LEVELS, ENCODED, 
    CREDENTIAL_STORE, SECRET_KEY,
    KEY_STORE, KEY_STORE_ALIAS,
    HASH_ENCODING, HASH_CHARSET,
    BRUTE_FORCE_PROTECTION  // ✅ Same attribute
};
```

**Best Practice**: Use static imports for shared attributes (FB-2)
- ✅ `import static ...BRUTE_FORCE_PROTECTION;` → cleaner code
- ❌ `RealmDefinitions.BRUTE_FORCE_PROTECTION` → verbose unless name conflict

**WFCORE-7193 Applied To**:
- 8 standard realm types (properties, filesystem, ldap, jdbc, jaas, caching, distributed, failover)
- 2 custom realm types (custom-realm, custom-modifiable-realm)

### Pattern: Parser Implementation for OBJECT Attributes

**Elytron Subsystem Pattern**: Version-chaining static helpers

**Structure**:
```java
// RealmParser.java

// Step 1: Base helper (original attributes, no brute-force-protection)
static PersistentResourceXMLBuilder propertiesRealmAttributes(
        PersistentResourceXMLBuilder builder) {
    return builder
        .addAttributes(PropertiesRealmDefinition.GROUPS_ATTRIBUTE)
        .addAttribute(PropertiesRealmDefinition.USERS_PROPERTIES, 
            AttributeParser.OBJECT_PARSER, AttributeMarshaller.ATTRIBUTE_OBJECT)
        .addAttribute(PropertiesRealmDefinition.GROUPS_PROPERTIES, 
            AttributeParser.OBJECT_PARSER, AttributeMarshaller.ATTRIBUTE_OBJECT)
        .addAttribute(PropertiesRealmDefinition.HASH_CHARSET)
        .addAttribute(PropertiesRealmDefinition.HASH_ENCODING);
}

// Step 2: Versioned helper (chains from base, adds brute-force-protection)
static PersistentResourceXMLBuilder propertiesRealmAttributes_19_0_community(
        PersistentResourceXMLBuilder builder) {
    return propertiesRealmAttributes(builder)  // ✅ Chain from base
        .addAttribute(RealmDefinitions.BRUTE_FORCE_PROTECTION, 
            AttributeParser.OBJECT_PARSER,           // ✅ OBJECT_PARSER
            AttributeMarshaller.ATTRIBUTE_OBJECT);   // ✅ ATTRIBUTE_OBJECT
}

// Step 3: Use in parser definition
private final PersistentResourceXMLDescription propertiesRealmParser_19_0_community = 
    propertiesRealmAttributes_19_0_community(
        builder(PathElement.pathElement(PROPERTIES_REALM))
    ).build();
```

**Why Version-Chaining** (FB-3):
- Single source of truth per version
- Easy to see version progression
- Future schema versions extend existing helpers
- Reduces duplication across 8+ realm types

**OBJECT Attribute Parsing**:
- Parser: `AttributeParser.OBJECT_PARSER` (built-in, handles attribute-based XML)
- Marshaller: `AttributeMarshaller.ATTRIBUTE_OBJECT` (renders as XML attributes)
- Alternative: Custom parser + marshaller for element-based rendering

**Elytron-Specific Note**: Elytron uses automatic parser routing via `since()` checks
- No explicit `ElytronSubsystemSchema.VERSION_19_0_COMMUNITY.since(version)` in parser
- Routing handled by decorator pattern in `getXMLDescription()`

**Contrast with Elytron-OIDC-Client**: Explicit parser fields per version
- See `schema-version-bump-guide.md` subsystem variations section

### Pattern: XSD Definition for OBJECT Attributes

**Attribute-Based Rendering** (WFCORE-7193):

```xml
<!-- wildfly-elytron_community_19_0.xsd -->

<!-- Step 1: Define the complex type for OBJECT attribute -->
<xs:complexType name="bruteForceProtectionType">
    <xs:attribute name="enabled" type="xs:boolean" default="false"/>
    <xs:attribute name="max-failed-attempts" type="xs:int"/>
    <xs:attribute name="lockout-interval" type="xs:int"/>
    <xs:attribute name="session-timeout" type="xs:int"/>
    <xs:attribute name="max-cached-sessions" type="xs:int"/>
</xs:complexType>

<!-- Step 2: Add to component type definitions -->
<xs:complexType name="propertiesRealmType">
    <xs:all>
        <xs:element name="users-properties" type="pathType"/>
        <xs:element name="groups-properties" type="pathType" minOccurs="0"/>
        <xs:element name="brute-force-protection" 
                    type="bruteForceProtectionType" 
                    minOccurs="0"/>  <!-- ✅ Optional -->
    </xs:all>
    <xs:attribute name="name" type="xs:string" use="required"/>
    <xs:attribute name="groups-attribute" type="xs:string"/>
    <!-- other attributes -->
</xs:complexType>
```

**XSD Patterns**:
- **xs:all**: Order-independent element container (most realm types)
- **xs:sequence**: Order-dependent element container (jdbc-realm, when `maxOccurs="unbounded"`)
- **xs:attribute**: For scalar attributes on the component itself

**Element Ordering Critical** (FB-11):
- `xs:sequence` requires elements in specific order
- ATTRIBUTES array order must match marshaller output order
- Marshaller outputs in parser registration order
- See `schema-version-bump-guide.md` for details

**Apply to All Component Types**:
- WFCORE-7193: Added `<brute-force-protection>` element to 10 realm type definitions
- Use `minOccurs="0"` since attribute is optional

## Runtime Integration

### Pattern: Reading OBJECT Attribute Values

**Access Nested Fields**:
```java
// In component's Add handler or runtime service
ModelNode bfConfig = context.readResource(PathAddress.EMPTY_ADDRESS)
    .getModel()
    .get(ElytronDescriptionConstants.BRUTE_FORCE_PROTECTION);

if (bfConfig.isDefined()) {
    boolean enabled = BF_ENABLED.resolveModelAttribute(context, bfConfig).asBoolean();
    int maxAttempts = BF_MAX_FAILED_ATTEMPTS.resolveModelAttribute(context, bfConfig).asInt();
    int lockoutInterval = BF_LOCKOUT_INTERVAL.resolveModelAttribute(context, bfConfig).asInt();
    // ... use values
}
```

**Expression Resolution**:
- `.resolveModelAttribute(context, modelNode)` resolves `${...}` expressions
- Returns resolved `ModelNode` with actual value
- Never access `.asInt()` without resolving first!

### Pattern: Backward Compatibility with System Properties

**WFCORE-7193 Special Case**: Feature already exists via system properties

**Requirement**: Model attributes take precedence, but system properties still work if attributes not set.

**Priority Logic** (3 levels):
1. **Model value** (if defined) → highest priority
2. **System property** (if model undefined) → fallback
3. **Library default** (if both undefined) → last resort

**Implementation**:
```java
// createBruteForceRealmTransformer() - 5 parameters
static Function<SecurityRealm, SecurityRealm> createBruteForceRealmTransformer(
        OperationContext context, ModelNode model,  // ✅ Model values
        String enabledProperty, String maxAttemptsProperty, 
        String lockoutProperty, String sessionTimeoutProperty, 
        String maxCachedProperty) {
    
    // Read from model (undefined if not set)
    ModelNode bfConfig = model.get(ElytronDescriptionConstants.BRUTE_FORCE_PROTECTION);
    
    return (realm) -> {
        // Priority: model → system property → default
        boolean enabled = bfConfig.hasDefined(BF_ENABLED.getName())
            ? BF_ENABLED.resolveModelAttribute(context, bfConfig).asBoolean()
            : Boolean.parseBoolean(System.getProperty(enabledProperty, "false"));
        
        int maxAttempts = bfConfig.hasDefined(BF_MAX_FAILED_ATTEMPTS.getName())
            ? BF_MAX_FAILED_ATTEMPTS.resolveModelAttribute(context, bfConfig).asInt()
            : Integer.parseInt(System.getProperty(maxAttemptsProperty, "-1"));
        
        // ... similar for other fields
        
        return BruteForceProtectedSecurityRealmBuilder.builder()
            .setDelegate(realm)
            .setEnabled(enabled)
            .setMaxFailedAttempts(maxAttempts)
            // ... other settings
            .build();
    };
}
```

**Normal Case** (no system properties):
- Simpler: Just read from model
- No priority logic needed
- Library defaults handled by builder or service

### Pattern: Transformer Integration for Backward Compatibility

**Purpose**: Allow old WildFly versions to connect to new versions

**Implementation**:
```java
// In component Add handler
@Override
public void performRuntime(OperationContext context, ModelNode operation, 
                           Resource resource) throws OperationFailedException {
    ModelNode model = resource.getModel();
    
    // Create transformer with model values
    Function<SecurityRealm, SecurityRealm> realmTransformer = 
        RealmDefinitions.createBruteForceRealmTransformer(
            context, model,
            "wildfly.elytron.realm.enabled",       // System property names
            "wildfly.elytron.realm.max.attempts",
            "wildfly.elytron.realm.lockout.interval",
            "wildfly.elytron.realm.session.timeout",
            "wildfly.elytron.realm.max.cached");
    
    // Pass transformer to service builder
    // ... create realm service with transformer
}
```

**Transformer Registration** (in `ElytronSubsystemTransformers.java`):
- See `management-model-version-bump-guide.md` for transformer mechanics
- WFCORE-7193: No transformer changes (model bump already done)

## Special Patterns

### Pattern: Custom Component Integration (FB-9)

**Challenge**: Custom components (custom-realm, custom-modifiable-realm) use anonymous subclasses in `ElytronDefinition.java`, not dedicated definition classes.

**Three-Layer Implementation Required**:

**Layer 1: Runtime - Update Transformer Interface**
```java
// CustomComponentDefinition.java

// Update interface to accept context and model
interface CustomComponentTransformer<T> {
    T prepareTransformer(OperationContext context, ModelNode model) 
        throws OperationFailedException;
}

// Update Add handler to pass context and model
class ComponentAddHandler extends AbstractAddStepHandler {
    @Override
    protected void performRuntime(OperationContext context, ModelNode operation, 
                                   Resource resource) throws OperationFailedException {
        ModelNode model = resource.getModel();
        // ... existing code
        
        T component = transformer.prepareTransformer(context, model);  // ✅ Pass both
        // ... register service
    }
}
```

**Layer 2: Parser - Add Custom Realm Parsers**
```java
// RealmParser.java

// Create versioned parsers for custom realms
static PersistentResourceXMLBuilder customRealmAttributes_19_0_community(
        PersistentResourceXMLBuilder builder) {
    return builder
        .addAttributes(CustomRealmDefinition.ATTRIBUTES)  // Base attributes
        .addAttribute(RealmDefinitions.BRUTE_FORCE_PROTECTION,
            AttributeParser.OBJECT_PARSER,
            AttributeMarshaller.ATTRIBUTE_OBJECT);  // ✅ Add new attribute
}

private final PersistentResourceXMLDescription customRealmParser_19_0_community = 
    customRealmAttributes_19_0_community(
        builder(PathElement.pathElement(CUSTOM_REALM))
    ).build();

// Update routing
PersistentResourceXMLDescription realmParser_19_0_community = decorateRealms(
    builder,
    // ... other parsers
    customRealmParser_19_0_community  // ✅ Use new parser
);
```

**Layer 3: XSD - Add to Custom Realm Types**
```xml
<!-- wildfly-elytron_community_19_0.xsd -->

<xs:complexType name="customRealmType">
    <xs:sequence>
        <xs:element name="configuration" type="configurationType" minOccurs="0"/>
        <xs:element name="brute-force-protection" 
                    type="bruteForceProtectionType" 
                    minOccurs="0"/>  <!-- ✅ Add element -->
    </xs:sequence>
    <xs:attribute name="name" type="xs:string" use="required"/>
    <xs:attribute name="module" type="xs:string" use="required"/>
    <xs:attribute name="class-name" type="xs:string" use="required"/>
</xs:complexType>
```

**All Three Layers Required**:
- Runtime only: Attribute accepted via CLI but ignored at runtime
- Parser only: XML parsed but not stored in model
- XSD only: XML validates but parser doesn't handle it

## Complete Implementation Checklist

Use this checklist to ensure complete implementation:

### Design Phase
- [ ] Version bumps complete (model + schema for target stability)
- [ ] OBJECT attribute vs child resource decision made
- [ ] XML structure (attributes vs elements) decided
- [ ] Single vs multiple component types determined
- [ ] Reviewed existing subsystem patterns for consistency

### String Handling
- [ ] All string constants added to `[Subsystem]DescriptionConstants.java`
- [ ] Count correct: 1 (parent) + N (nested fields) constants
- [ ] Resource bundle entries added to `LocalDescriptions.properties`
- [ ] Count correct: M component types × (1 parent + N fields) entries
- [ ] Resource bundles include library defaults ("uses ... default (X)")
- [ ] Resource bundles use correct units (minutes/seconds/etc)
- [ ] Wording consistent across all component types

### Model Definition
- [ ] Nested `SimpleAttributeDefinition` fields created
- [ ] `.setAllowExpression(true)` on fields that support expressions
- [ ] `.setMeasurementUnit()` on time/size fields
- [ ] `.setValidator()` for constraints (range, regex, etc)
- [ ] Parent `ObjectTypeAttributeDefinition` created
- [ ] `.setStability()` on parent (if not DEFAULT)
- [ ] Shared attributes defined in common class
- [ ] Static imports used in component definitions (unless name conflict)

### Parser Implementation
- [ ] Version-chaining static helpers created (Elytron pattern)
- [ ] Base helper for original attributes
- [ ] Versioned helper chains from base, adds new attribute
- [ ] `AttributeParser.OBJECT_PARSER` used (for attribute-based XML)
- [ ] `AttributeMarshaller.ATTRIBUTE_OBJECT` used (for attribute-based XML)
- [ ] Parser definitions use versioned helpers
- [ ] All applicable component types updated

### XSD Definition
- [ ] Complex type created for OBJECT attribute
- [ ] Uses `xs:attribute` (for attribute-based rendering)
- [ ] Added to all applicable component type definitions
- [ ] `minOccurs="0"` (if optional)
- [ ] Element ordering correct (matches ATTRIBUTES array / marshaller order)
- [ ] `xs:all` vs `xs:sequence` choice appropriate

### Runtime Integration
- [ ] Component Add handlers read model values correctly
- [ ] Expression resolution using `.resolveModelAttribute()`
- [ ] Service builders receive resolved values
- [ ] Backward compatibility handled (if replacing system properties)
- [ ] Priority logic correct: model → system property → default
- [ ] Transformer functions created (if needed)

### Custom Components (if applicable)
- [ ] Runtime: Transformer interface updated to accept context + model
- [ ] Runtime: Add handler passes context + model to transformer
- [ ] Parser: Versioned parsers created for custom components
- [ ] XSD: Custom component types updated with new elements

### Testing (see subsystem-schema-test-requirements.md)
- [ ] Test XML file updated with literal values
- [ ] Expression coverage added (for OBJECT attributes with expressions)
- [ ] All applicable component types tested
- [ ] Element ordering matches marshaller output
- [ ] Tests pass: `SubsystemParsingTestCase`
- [ ] Tests pass: `[Subsystem]MixedStabilitySubsystemParsingTestCase` (if multi-stability)

## Common Pitfalls

### Pitfall 1: Missing String Constants
**Symptom**: Using string literals instead of constants  
**Fix**: Add all strings to `[Subsystem]DescriptionConstants.java`

### Pitfall 2: Incomplete Resource Bundles
**Symptom**: Missing descriptions for some component types  
**Fix**: Count check: M components × (1 + N fields) = total entries

### Pitfall 3: Missing Measurement Units
**Symptom**: Time/size attributes without `.setMeasurementUnit()`  
**Fix**: Add `.setMeasurementUnit(MeasurementUnit.MINUTES)` etc to attribute builder

### Pitfall 4: Expression Resolution Skipped
**Symptom**: Expressions like `${...}` not resolved, treated as literals  
**Fix**: Use `.resolveModelAttribute(context, model)` before `.asInt()` etc

### Pitfall 5: Incomplete Custom Component Integration
**Symptom**: Custom components accept attribute via CLI but runtime ignores it  
**Fix**: Implement all three layers (Runtime interface + Parser + XSD)

### Pitfall 6: Element Ordering Mismatch
**Symptom**: Test XML marshalling comparison fails  
**Fix**: Ensure ATTRIBUTES array order matches marshaller output (parser registration order)

### Pitfall 7: Inconsistent Resource Bundle Wording
**Symptom**: Different descriptions for same attribute across component types  
**Fix**: Use identical wording, differ only in component type name

### Pitfall 8: Wrong Stability Level
**Symptom**: COMMUNITY feature accessible at DEFAULT stability  
**Fix**: Add `.setStability(Stability.COMMUNITY)` to parent OBJECT attribute

## Examples and References

### Complete Example: WFCORE-7193

**Feature**: Brute force protection configuration for security realms

**Classification**:
- Replacing system properties (special case)
- OBJECT attribute with 5 nested fields
- Applied to 10 realm types
- Community stability level
- Attribute-based XML rendering

**Implementation Summary**:
- 6 string constants added
- 70 resource bundle entries (10 realms × 7 entries)
- 1 OBJECT attribute + 5 nested SimpleAttributeDefinition fields
- 10 component ATTRIBUTES arrays updated
- 8 version-chaining static helpers (one per standard realm type)
- 10 XSD type definitions updated
- 10 parser definitions use versioned helpers
- Runtime: 5-parameter transformer with priority logic
- Tests: 8 realm types × 2 (literal + expression) = 16 test configurations

**Files Modified** (17 files):
1. `ElytronDescriptionConstants.java` - 6 constants
2. `RealmDefinitions.java` - OBJECT attribute, nested fields, transformer
3. `RealmParser.java` - Version-chaining helpers, parser definitions
4. `ElytronDefinition.java` - Custom realm attribute registration
5. `CustomComponentDefinition.java` - Interface update for context/model
6-13. 8 realm definition files - Static imports, ATTRIBUTES arrays
14. `wildfly-elytron_community_19_0.xsd` - 10 type definitions updated
15. `elytron-subsystem-community-19.0.xml` - Test coverage
16. `LocalDescriptions.properties` - 70 resource bundle entries
17. `ElytronSubsystemSchema.java` - (Already done in schema bump)

### Cross-Reference to Other Guides

- **Version Bumps**: See `management-model-version-bump-guide.md` and `schema-version-bump-guide.md`
- **Parser Patterns**: See `schema-version-bump-guide.md` subsystem variations
- **Testing**: See `subsystem-schema-test-requirements.md`
- **Test File Lifecycle**: See `schema-version-bump-guide.md` Step 6 (FB-5)

## Revision History

- **2026-09-12**: Initial version (v1.0) based on WFCORE-7193 implementation patterns
  - Covers OBJECT attribute patterns, string handling, runtime integration
  - Emphasizes version bump coordination via Zulip
  - Distinguishes new features vs replacing system properties
  - Custom component integration pattern (FB-9)
  - Version-chaining static helpers (FB-3)
  - Complete implementation checklist
  - Added "Check If Version Bumps Are Needed" section referencing last .Final tag checks
