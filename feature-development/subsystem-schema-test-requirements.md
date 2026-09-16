# WildFly Subsystem Schema Test Requirements

## Overview

WildFly subsystems use schema-based testing to verify that XML configuration can be correctly parsed and marshalled. Each schema version requires a corresponding test XML file that provides comprehensive coverage of all schema elements, attributes, and complex types.

## Test File Location and Naming

Test XML files are located in:
```
src/test/resources/org/wildfly/extension/{subsystem-name}/
```

### Naming Convention Based on Schema Status

Test file naming depends on whether the schema version is in the subsystem's `CURRENT` map:

**Current Schema Versions** (in `{Subsystem}Schema.CURRENT` map):
- `{subsystem-name}-{version}.xml` - For DEFAULT stability (e.g., `elytron-subsystem-19.0.xml`)
- `{subsystem-name}-{stability}-{version}.xml` - For other stabilities (e.g., `elytron-subsystem-community-19.0.xml`)

**Historical Schema Versions** (NOT in CURRENT map):
- `legacy-{subsystem-name}-{version}.xml` - For DEFAULT stability (e.g., `legacy-elytron-subsystem-18.0.xml`)
- `legacy-{subsystem-name}-{stability}-{version}.xml` - For other stabilities (e.g., `legacy-elytron-subsystem-community-18.0.xml`)

### Schema Lifecycle and Test File Management

**Critical Policy**: Schema enum entries are permanent for backward compatibility. Once added to the subsystem's schema enum (e.g., `ElytronSubsystemSchema`), entries MUST remain forever to support runtime schema version negotiation.

**Test Framework Requirements**: The test framework uses `EnumSet.allOf({Subsystem}Schema.class)`, which means every enum entry needs a corresponding test file. The framework locates test files by converting enum names to expected filenames.

**When Schema Is Promoted** (e.g., 19.0 → 20.0):

1. Add `VERSION_20_0` to schema enum (NEVER remove `VERSION_19_0`)
2. Update `CURRENT` map to point to `VERSION_20_0`
3. Rename test file: `elytron-subsystem-19.0.xml` → `legacy-elytron-subsystem-19.0.xml`
4. Create new test file: `elytron-subsystem-20.0.xml`

**Test Failure Indicator**: If you see an error like:
```
elytron-subsystem-community-18.0.xml url is null
```
This means a schema enum entry exists but the test file doesn't match the expected naming pattern (likely missing the `legacy-` prefix).

**Example from Elytron Subsystem** (WildFly 42):

```java
// ElytronSubsystemSchema.java
static final Map<Stability, ElytronSubsystemSchema> CURRENT = Feature.map(
    EnumSet.of(VERSION_19_0, VERSION_19_0_COMMUNITY));
```

Test files:
- `elytron-subsystem-19.0.xml` (DEFAULT, in CURRENT)
- `elytron-subsystem-community-19.0.xml` (COMMUNITY, in CURRENT)
- `legacy-elytron-subsystem-community-18.0.xml` (COMMUNITY, NOT in CURRENT)

## Test Requirements

### 1. Complete Schema Coverage

Every test XML file MUST include:

- **All elements** defined in the corresponding XSD schema
- **All attributes** for each element (including optional ones)
- **All complex types** with representative examples
- **Multiple instances** of repeatable elements where applicable
- **Both literal and expression forms** for every attribute that supports expression substitution (see Section 1a)

**Coverage Checklist:**
- [ ] All top-level elements (e.g., `realm`, `provider`, `secure-deployment`, `secure-server`)
- [ ] All child elements within each complex type
- [ ] All optional elements (at least one instance)
- [ ] All attributes (including optional ones like `algorithm` on credentials)
- [ ] Multiple instances of elements that can appear multiple times
- [ ] Each attribute appears at least once as a plain literal value
- [ ] Each attribute appears at least once as a `${property:default}` expression
- [ ] Each root element has at least one minimal entry (name only, no optional children)

### 1a. Expression Coverage Requirements

WildFly subsystem tests verify that both literal values and expression-substituted values can be parsed and round-tripped through the management model. **For every attribute that supports expressions** (defined with `.setAllowExpression(true)` in the Java attribute builder), the test XML must include at least one literal instance and at least one `${...}` expression instance somewhere across all entries of that complex type.

**Recommended pattern**: Maintain one "full literal" entry and one "full expression" entry for each complex type, rather than spreading coverage thinly across many sparse entries. For example:
- A `google` provider entry that includes every possible provider attribute as a literal value
- A `redhat-sso-expressions` provider entry that includes every possible provider attribute as an expression

This makes it easy to verify coverage at a glance and ensures new attributes added to the schema are added to both entries.

**Resource path key attributes are exempt**: The `name` attribute on elements like `credential` and `redirect-rewrite-rule` is the WildFly management address path key, NOT a model attribute. It is registered via `PathElement.pathElement(...)`, not via `SimpleAttributeDefinitionBuilder`, and does NOT support expression substitution. These `name` keys do not need expression coverage.

**Alternative attributes**: Some attributes are defined as mutually exclusive alternatives (via `.setAlternatives(...)` in Java, e.g., `auth-server-url` and `provider-url`). Combined coverage across all entries satisfies the requirement — one entry can use `auth-server-url` and another can use `provider-url`, and each needs both a literal and expression form across those combined entries.

### 1b. Expression Coverage for OBJECT Attributes

**OBJECT attributes** (defined with `ObjectTypeAttributeDefinition`) contain multiple nested fields. Expression testing for OBJECT attributes follows a different pattern than simple attributes.

**Pattern: Embed Expression Tests in Version-Specific Test Files**

For OBJECT attributes, include both literal and expression variants directly in the same test file, placing the expression entry immediately after the corresponding literal entry:

**Elytron Subsystem Pattern** (from WFCORE-7193):

```xml
<!-- Literal entry -->
<properties-realm name="PropRealm" groups-attribute="groups">
    <brute-force-protection enabled="true" max-failed-attempts="5" 
        lockout-interval="10" session-timeout="15" max-cached-sessions="1000"/>
</properties-realm>

<!-- Expression entry - immediately after literal -->
<properties-realm name="PropRealm-expressions" groups-attribute="groups">
    <brute-force-protection 
        enabled="${exp.bf.enabled:true}" 
        max-failed-attempts="${exp.bf.max-attempts:5}"
        lockout-interval="${exp.bf.lockout:10}" 
        session-timeout="${exp.bf.timeout:15}" 
        max-cached-sessions="${exp.bf.sessions:1000}"/>
</properties-realm>
```

**Key Requirements**:
- Use standard WildFly expression format: `${exp.category.field:default}`
- Default values in expressions should match literal entry values
- Place expression entry immediately after corresponding literal entry
- Cover ALL fields in the OBJECT attribute with expressions
- Use descriptive naming suffix like `-expressions` to indicate purpose

**Why Embedded (Not Separate Files)**:
- OBJECT attributes are complex to maintain in separate expression test files
- Version-specific attributes need version-specific expression tests
- Easier to keep literal and expression tests synchronized
- Clearer relationship between literal and expression coverage

**Comparison with Simple Attributes**: Simple attributes (defined with `SimpleAttributeDefinition`) can use the "full literal" + "full expression" entry pattern described in Section 1a. OBJECT attributes require per-component embedded expression entries.

### 1c. Coverage Across All Applicable Component Types

When adding a new attribute that applies to multiple component types, you MUST add test coverage for ALL applicable component types.

**Coverage Checklist** (when adding cross-component attributes):

1. List all component types that support the new attribute
2. For each type, add a literal test entry
3. For each type, add an expression test entry (immediately after literal)
4. Verify any supporting definitions needed (e.g., `<dir-contexts>` for LDAP realms)

**Elytron Subsystem Pattern** (WFCORE-7193 - brute-force-protection on 8 realm types):

```xml
<!-- 1. Properties Realm -->
<properties-realm name="PropRealm" groups-attribute="groups">
    <brute-force-protection enabled="true" max-failed-attempts="5"/>
</properties-realm>
<properties-realm name="PropRealm-expressions" groups-attribute="groups">
    <brute-force-protection enabled="${exp.bf.enabled:true}" 
        max-failed-attempts="${exp.bf.max-attempts:5}"/>
</properties-realm>

<!-- 2. Filesystem Realm -->
<filesystem-realm name="FileRealm" path="...">
    <brute-force-protection enabled="true" max-failed-attempts="5"/>
</filesystem-realm>
<filesystem-realm name="FileRealm-expressions" path="...">
    <brute-force-protection enabled="${exp.bf.enabled:true}" 
        max-failed-attempts="${exp.bf.max-attempts:5}"/>
</filesystem-realm>

<!-- ... repeat for all 8 realm types ... -->
```

**Parser-Supported vs Non-Parser Components**:
- **Parser-supported components** (have dedicated parser entries): Require XML test coverage
- **Non-parser components** (registered via code, like custom-realm): May not appear in parser test files

Incomplete coverage can miss parser bugs that would only surface in production. In WFCORE-7193, initial coverage had only 2 of 8 realm types; complete review found 6 missing types.

### 1d. Minimal Configuration Coverage

> **⚠ BLOCKED** — Implementation is pending a schema bug fix. See
> `notes/bug-xsd-all-minoccurs-empty-elements.md` for details. Do not add
> minimal entries to test XML files until the bug is resolved.
> This section remains in the guide for future reference.

In addition to the "full literal" and "full expression" coverage patterns, each root element in the subsystem must have at least one **minimal** entry — an entry that contains ONLY the management address `name` attribute, with NO optional child elements. This verifies:

1. That every child element is truly optional and can be omitted without error
2. That the subsystem can parse and round-trip a bare-bones configuration
3. That no default marshalling logic requires child elements to be present

**Pattern**: Use a self-closing tag with just the name attribute:
```xml
<realm name="minimal-realm"/>
<provider name="minimal-provider"/>
<secure-deployment name="minimal.war"/>
<secure-server name="minimal-server"/>   <!-- versions 2.0+ only -->
```

**Propagation**: Like other coverage requirements, minimal entries must be propagated to all subsequent schema versions. If `1.0.xml` has a `<realm name="minimal-realm"/>`, all later versions must also include it.

**Naming convention**: Use `minimal-` as a prefix (e.g., `minimal-realm`, `minimal-provider`) and `minimal.war` for deployments/servers to make their purpose immediately clear.

**Known blocker**: The XSD schemas define `realm-type`, `provider-type`, and `secure-deployment-type` using `<xs:all>` without `minOccurs="0"` on the group itself. Xerces (the XML validator used by the test framework) rejects empty elements for these types even though every individual child element has `minOccurs="0"`. The fix is to add `minOccurs="0"` to the `<xs:all>` opening tag in each affected complex type across all five XSD schema files. Until that schema fix is applied, minimal entries fail `testSchema` with `cvc-complex-type.2.4.b`.

### 2. Element Ordering Requirements

**CRITICAL**: Element ordering in test XML files MUST match the marshaller's output order, NOT the XSD schema order.

#### XSD Schema Constraints: xs:all vs xs:sequence

XSD schemas use two primary element ordering mechanisms:

**`<xs:all>`** - Elements can appear in any order during parsing:
```xml
<xs:all>
    <xs:element name="realm-public-key" type="xs:string" minOccurs="0"/>
    <xs:element name="auth-server-url" type="xs:string" minOccurs="0"/>
</xs:all>
```
- Allows any element order in XML input
- Cannot use `maxOccurs="unbounded"` on child elements
- Marshaller still outputs in a specific order (typically attribute registration order)

**`<xs:sequence>`** - Elements MUST appear in the exact order specified:
```xml
<xs:sequence>
    <xs:element name="principal-query" type="authenticationQueryType" maxOccurs="unbounded"/>
    <xs:element name="brute-force-protection" type="bruteForceProtectionType" minOccurs="0"/>
</xs:sequence>
```
- Enforces strict ordering (required for `maxOccurs="unbounded"`)
- Parser rejects XML with wrong element order
- Marshaller outputs in the order defined in the ATTRIBUTES array

#### Marshaller Behavior and ATTRIBUTES Array

The marshaller outputs elements in the order they appear in the resource definition's `ATTRIBUTES` array:

**Example** (JdbcRealmDefinition.java):
```java
static final AttributeDefinition[] ATTRIBUTES = new AttributeDefinition[] {
    // ... other attributes ...
    PRINCIPAL_QUERIES,        // Must come before BRUTE_FORCE_PROTECTION
    BRUTE_FORCE_PROTECTION    // Must come after PRINCIPAL_QUERIES
};
```

**Test XML must match**:
```xml
<jdbc-realm name="JdbcRealmModular" ...>
    <principal-query .../>      <!-- First -->
    <principal-query .../>      <!-- Can have multiple -->
    <brute-force-protection .../> <!-- Must be last -->
</jdbc-realm>
```

**Impact**:
- For `xs:sequence` schemas: Wrong order fails XML validation
- For `xs:all` schemas: Wrong order fails test comparison (marshaller vs input)
- Marshaller always outputs in ATTRIBUTES array order

#### How to Determine Correct Ordering

1. Run the test with your initial XML file
2. If the test fails with a comparison error, examine the test output in:
   ```
   target/surefire-reports/{SubsystemName}SubsystemTestCase.txt
   ```
3. The "expected" section shows the marshaller's actual output order
4. Reorder elements in your test XML to match the marshaller's order
5. Re-run tests to verify

**Common Ordering Patterns:**

For the elytron-oidc-client subsystem (as an example):
- Basic configuration elements come first (e.g., `realm-public-key`, `auth-server-url`)
- Security/SSL elements follow (e.g., `ssl-required`, `truststore`)
- CORS elements are grouped together
- Timeout/connection elements are grouped
- New feature elements often come after core elements
- Request object elements follow authentication format elements

### 3. Cumulative Schema Evolution

Schema versions are cumulative - each new version builds upon the previous one:

- **Version 1.0**: Base elements
- **Version 2.0**: Base + new elements from 2.0
- **Preview 2.0**: Base + 2.0 + preview-specific elements (e.g., `scope`)
- **Preview 3.0**: Base + 2.0 + preview-2.0 + preview-3.0 elements (e.g., request object support)
- **Preview 4.0**: Base + 2.0 + preview-2.0 + preview-3.0 + preview-4.0 elements (e.g., OIDC logout)

**Best Practice**: When creating a new version's test file, start by copying the previous version's test file, update the namespace, then add only the new elements introduced in that version.

**Coverage propagation is mandatory**: When a coverage gap is discovered and fixed in an earlier version's test file (e.g., adding a missing literal to `1.0.xml`), that fix MUST be propagated to ALL subsequent versions. Each file should be logically identical to its predecessor except for:
1. The namespace URI (e.g., `urn:wildfly:elytron-oidc-client:1.0` → `urn:wildfly:elytron-oidc-client:2.0`)
2. New attributes/elements added by that schema version (added to the "full literal" and "full expression" entries)

Failing to cascade coverage fixes forward means later versions will silently have the same gap.

### 4. Test Values

Use realistic but clearly test-oriented values:

- **Keys/Certificates**: Use valid-format but obviously test keys
- **URLs**: Use localhost or example.com domains
- **Ports**: Use standard ports (8080, 8180, etc.)
- **Credentials**: Use UUIDs or clearly fake values
- **Timeouts**: Use reasonable values (not extreme)

### 5. Multiple Instances

Include multiple instances of repeatable elements to verify:
- Different configuration patterns
- Different credential types (e.g., `secret`, `jwt`, `secret-jwt`)
- Different deployment scenarios (e.g., with realm, with provider, bearer-only)

## Verification Process

### Running Tests

```bash
cd wildfly/elytron-oidc-client
mvn clean test -Dtest=ElytronOidcClientSubsystemTestCase
```

### Test Failure Analysis

When a test fails:

1. **Check the error type**:
   - `ComparisonFailure`: Element ordering issue or missing/extra elements
   - `ParseException`: Invalid XML or schema violation
   - Other errors: Logic or configuration issues

2. **For ComparisonFailure**:
   - Open `target/surefire-reports/ElytronOidcClientSubsystemTestCase.txt`
   - Compare "expected" (marshaller output) vs "actual" (your XML)
   - Look for differences in element order or content
   - Update your XML to match the expected output

3. **For ParseException**:
   - Verify XML is well-formed
   - Check element names match schema exactly
   - Verify required attributes are present
   - Ensure values match expected types (boolean, integer, string, etc.)

## Review Checklist for AI Agents

When reviewing or creating subsystem test files:

- [ ] Test file exists for each schema enum version (including legacy versions)
- [ ] Test file naming follows CURRENT map convention (legacy- prefix for non-current)
- [ ] Namespace in test XML matches schema version
- [ ] All schema elements are represented at least once as a literal value
- [ ] All schema elements are represented at least once as an expression (`${...}` form)
- [ ] All attributes (including optional) are included in both literal and expression form
- [ ] For OBJECT attributes: literal and expression entries are embedded in the same file
- [ ] For cross-component attributes: coverage exists for ALL applicable component types
- [ ] Element ordering matches marshaller output and XSD constraints (xs:sequence vs xs:all)
- [ ] Multiple instances of repeatable elements are included
- [ ] Test values are realistic but clearly for testing
- [ ] New version test files build upon previous versions
- [ ] Coverage fixes in any version have been propagated to all later versions
- [ ] Each root element has at least one minimal entry (blocked pending XSD bug fix)
- [ ] Tests pass without comparison failures

**Coverage verification strategy**: The most reliable way to verify expression coverage is to identify the "full literal" entry for each complex type and check it contains every attribute, then do the same for the "full expression" entry. For OBJECT attributes, verify literal and expression variants exist as paired entries. Cross-file agent verification is prone to false positives — always confirm reported gaps by directly reading the XML before making changes.

## Common Pitfalls

1. **Ordering Assumption**: Don't assume XSD order = marshaller order
2. **Missing Optional Elements**: Include optional elements to ensure they can be parsed
3. **Incomplete Coverage**: Missing even one element can leave gaps in validation
4. **Copy-Paste Errors**: When copying from previous versions, update all version-specific references
5. **Namespace Mismatches**: Ensure namespace exactly matches the schema version
6. **Expression Coverage Gaps**: Forgetting that each attribute needs BOTH a literal and expression form, not just one
7. **Not Cascading Fixes**: Fixing a gap in one version but forgetting to apply the same fix to all later versions
8. **Confusing Path Keys with Model Attributes**: The `name` attribute on credentials and redirect-rewrite-rules is a management resource address key — it doesn't support expressions and doesn't need expression coverage
9. **Agent False Positives in Coverage Checks**: Automated agents scanning XML for coverage can miss entries that exist in different parts of the file. Always verify a reported gap by manually reading the XML before acting on it. Confirmed gaps will also exist identically in all versions derived from the one with the gap.
10. **Missing Minimal Configuration**: Only testing full configurations and forgetting to verify that entries with only the required name attribute (and no optional children) can be parsed and marshalled (currently blocked - see Section 1d)
11. **Wrong Test File Naming for Historical Schemas**: Forgetting to rename test files with `legacy-` prefix when a schema version is no longer in the CURRENT map. Results in "url is null" test failures.
12. **Incomplete Component Type Coverage**: When adding a new attribute that applies to multiple component types (e.g., brute-force-protection on 8 realm types), only testing a subset of component types and missing parser bugs in the others
13. **Separate Expression Files for OBJECT Attributes**: Creating separate expression test files for OBJECT attributes instead of embedding literal and expression variants in the same file. This makes synchronization difficult and version-specific attributes harder to maintain.
14. **Violating xs:sequence Order**: For schemas using `xs:sequence`, placing elements in wrong order causes XML validation failures. Always check if XSD uses `xs:sequence` (required for unbounded elements) vs `xs:all` (flexible ordering).

## Example: Adding a New Element

When a new element is added to a schema:

1. Identify which complex type(s) it belongs to
2. Add the element to ALL test files for versions that include it
3. Place it in the correct position (run tests to verify)
4. Use a representative test value
5. Run tests and adjust ordering if needed
6. Verify all tests pass

## Stability-Level-Specific Testing (OidcTestCase Pattern)

Some subsystems use a separate test class (e.g., `OidcTestCase`) that validates runtime behavior by parsing configurations and checking loaded values. When a subsystem supports multiple stability levels (DEFAULT, COMMUNITY, PREVIEW), this test should be parameterized to test each CURRENT stability level independently.

### Implementation Pattern

Use JUnit parameterized tests to run the same test suite against each stability level:

```java
@RunWith(Parameterized.class)
public class OidcTestCase extends AbstractSubsystemSchemaTest<ElytronOidcSubsystemSchema> {

    @Parameters(name = "{0}")
    public static Collection<ElytronOidcSubsystemSchema> parameters() {
        return ElytronOidcSubsystemSchema.CURRENT.values();
    }

    public OidcTestCase(ElytronOidcSubsystemSchema schema) {
        super(ElytronOidcExtension.SUBSYSTEM_NAME, new ElytronOidcExtension(),
              schema, ElytronOidcSubsystemSchema.CURRENT.get(schema.getStability()));
    }

    @Override
    protected String getSubsystemXml() throws IOException {
        return readResource(String.format("oidc-%s.xml",
            this.getSubsystemSchema().getStability().toString().toLowerCase()));
    }
}
```

### Test XML Files for Each Stability Level

Create separate test XML files for each stability level:

- `oidc-default.xml` - Uses DEFAULT stability namespace (e.g., `urn:wildfly:elytron-oidc-client:2.0`)
- `oidc-community.xml` - Uses COMMUNITY stability namespace (e.g., `urn:wildfly:elytron-oidc-client:community:2.0`)
- `oidc-preview.xml` - Uses PREVIEW stability namespace (e.g., `urn:wildfly:elytron-oidc-client:preview:4.0`)

**Key Requirements:**
- Each file must use the correct namespace for its stability level
- Each file must include only features available at that stability level
- Files should NOT include XML declarations (`<?xml version="1.0"?>`)
- Element ordering must match marshaller output (alphabetical for `xs:all` schemas)

### Conditional Test Execution

Use `assumeTrue()` to skip tests for features not available at certain stability levels:

```java
@Test
public void testPreviewOnlyFeature() throws Exception {
    assumeTrue("Feature X is PREVIEW-only",
        this.getSubsystemSchema().getStability() == Stability.PREVIEW);
    // Test code here
}
```

### Adding New Features

When adding a new feature to the subsystem:

1. **Determine the initial stability level** (usually PREVIEW)
2. **Add the feature to the appropriate test XML file(s)**:
   - If PREVIEW: Add to `oidc-preview.xml` only
   - If COMMUNITY: Add to `oidc-community.xml` and `oidc-preview.xml`
   - If DEFAULT: Add to all three files
3. **Add tests for the new feature** with appropriate `assumeTrue()` guards
4. **Update test XML files** to include both literal and expression forms of new attributes

### Promoting Features Between Stability Levels

When a feature is promoted from one stability level to another (e.g., PREVIEW → COMMUNITY):

1. **Update the schema** to reflect the new stability level
2. **Move feature entries between test XML files**:
   - Remove from higher stability file (e.g., `oidc-preview.xml`)
   - Add to lower stability file (e.g., `oidc-community.xml`)
   - Ensure all lower stability files also get the feature
3. **Update `assumeTrue()` conditions** in tests:
   ```java
   // Before (PREVIEW-only):
   assumeTrue("Feature X is PREVIEW-only",
       this.getSubsystemSchema().getStability() == Stability.PREVIEW);

   // After (COMMUNITY and above):
   assumeTrue("Feature X requires COMMUNITY or higher",
       this.getSubsystemSchema().getStability().ordinal() >= Stability.COMMUNITY.ordinal());
   ```
4. **Verify all tests pass** for all stability levels

### Test Execution Results

When tests run correctly:
- Total tests = (number of test methods) × (number of stability levels)
- Skipped tests = (stability-restricted tests) × (number of stability levels where feature is unavailable)
- Example: 30 test methods × 2 stability levels = 60 total tests
  - If 6 tests are PREVIEW-only: 6 × 1 (DEFAULT run) = 6 skipped tests
  - Result: 60 tests run, 54 passed, 6 skipped

### Common Issues

1. **XML Declaration in Test Files**: Test XML files should NOT include `<?xml version="1.0"?>` declarations. The test framework reads them as strings and the declaration causes parsing errors.

2. **Element Ordering**: Marshalled XML outputs elements in a specific order (often alphabetical for `xs:all` schemas). Test XML files must match this order exactly.

3. **Namespace Mismatches**: Each test XML must use the correct namespace for its stability level. Check `ElytronOidcSubsystemSchema` enum for the correct namespace URNs.

4. **Missing Feature Removal**: When promoting a feature, don't forget to remove it from the higher stability test XML file. Leaving it in both files can cause confusion.

5. **Incorrect assumeTrue() Logic**: When promoting features, update the stability check to use ordinal comparison (`>=`) rather than equality (`==`) if the feature should be available at multiple levels.

## References

- XSD Schemas: `src/main/resources/schema/`
- Test Files: `src/test/resources/org/wildfly/extension/{subsystem}/`
- Test Class: `src/test/java/org/wildfly/extension/{subsystem}/{Subsystem}SubsystemTestCase.java`
- Test Framework: `org.jboss.as.subsystem.test.AbstractSubsystemSchemaTest`
- Stability Levels: `org.jboss.as.version.Stability` enum

## Version History

### 2026-09-11 - WFCORE-7193 Lessons Integration

Updated based on lessons learned from WFCORE-7193 (brute-force-protection implementation):

**Major Additions:**
- **Section 2 (Test File Naming)**: Added comprehensive test file lifecycle management, legacy- prefix convention, CURRENT map relationship, and schema promotion process
- **Section 1b (Expression Coverage for OBJECT Attributes)**: New section on embedded expression testing pattern for OBJECT attributes
- **Section 1c (Coverage Across Component Types)**: New section on ensuring complete coverage when attributes apply to multiple component types
- **Section 2 (Element Ordering)**: Expanded with xs:sequence vs xs:all constraints, ATTRIBUTES array ordering, and marshaller behavior
- **Review Checklist**: Updated with new requirements for test file naming, OBJECT attributes, component type coverage, and XSD constraints
- **Common Pitfalls**: Added 4 new pitfalls (#11-14) covering historical schema naming, component type coverage, OBJECT attribute expression patterns, and xs:sequence violations

**Key Patterns Documented:**
- Elytron subsystem pattern for OBJECT attribute expression testing
- Test file naming conventions based on schema CURRENT map status
- Complete component type coverage checklist (parser-supported vs non-parser)
- XSD sequence constraint impact on element ordering