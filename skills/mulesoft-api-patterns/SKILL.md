---
name: mulesoft-api-patterns
description: Generate complete MuleSoft projects implementing API design patterns from JJ Geewax's "API Design Patterns" book. Use this skill when users request to create MuleSoft integration projects, API archetypal patterns, REST API implementations, or examples based on industry-standard API patterns including Long Running Operations, Pagination, Versioning, Batch Operations, Partial Updates, Resource Expansion, List Filtering, and others. Also use when users ask to create MuleSoft project templates, API pattern examples, or integration archetypes.
---

# MuleSoft API Patterns Generator

Generate production-ready MuleSoft 4.x projects implementing proven API design patterns from JJ Geewax's "API Design Patterns" book.

## Quick Start

When a user requests a MuleSoft project based on an API pattern:

1. **Identify the pattern** from user request or ask which pattern to implement
2. **Load pattern reference**: `view references/patterns/<pattern-name>.md`
3. **Use base template**: Copy from `assets/mule-base-template/`
4. **Generate project** following pattern specifications
5. **Verify critical configurations** (see Critical Requirements below)
6. **Package and present** the complete project to user

## Critical Requirements & Common Issues

⚠️ **IMPORTANT**: All generated projects must meet these requirements to compile and run successfully:

### 1. Maven Plugin Version (CRITICAL)
- **MUST use mule-maven-plugin 4.1.1 or higher**
- Version 3.8.0 and older are incompatible with Java 17 and Maven 3.9+


### 2. Java Version
- **Use Java 17.0.11** for all new projects
- Configure in pom.xml, .classpath, and .settings/org.eclipse.jdt.core.prefs
- Set maven.compiler.release property

```xml
<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>
<maven.compiler.release>17</maven.compiler.release>
```


## 3. Supported Patterns

Load the appropriate reference file for detailed implementation guidance:

### Resource Lifecycle Patterns
- **Long Running Operations** - `references/patterns/long-running-operations.md`
- **Batch Operations** - `references/patterns/batch-operations.md`
- **Partial Updates (PATCH)** - `references/patterns/partial-updates.md`

### Data Retrieval Patterns
- **Pagination** - `references/patterns/pagination.md`
- **List Filtering & Sorting** - `references/patterns/list-filtering.md`
- **Resource Expansion** - `references/patterns/resource-expansion.md`
- **Field Selection** - `references/patterns/field-selection.md`

### API Evolution Patterns
- **Versioning** - `references/patterns/versioning.md`
- **Deprecation** - `references/patterns/deprecation.md`
- **Compatibility** - `references/patterns/compatibility.md`

### Data Consistency Patterns
- **Conditional Requests** - `references/patterns/conditional-requests.md`
- **Idempotency** - `references/patterns/idempotency.md`
- **Optimistic Locking** - `references/patterns/optimistic-locking.md`

## Project Generation Workflow

### 1. Pattern Selection

If pattern is not specified in user request, present available patterns:

```
Which API pattern would you like to implement?

Resource Lifecycle:
1. Long Running Operations - For async tasks (reports, exports, processing)
2. Batch Operations - Create/update multiple resources at once
3. Partial Updates - PATCH operations for selective field updates

Data Retrieval:
4. Pagination - Handle large result sets efficiently
5. List Filtering - Query parameters for filtering and sorting
6. Resource Expansion - Include related resources in responses

[etc.]
```

### 2. Load Pattern Reference

Read the pattern-specific reference file:

```bash
view references/patterns/<pattern-name>.md
```

This file contains:
- Pattern description and use cases
- Required endpoints and HTTP methods
- Request/response schemas
- MuleSoft-specific implementation details
- DataWeave transformations
- Error handling requirements

### 3. Generate Project Structure

Use the base template as foundation:

```bash
cp -r assets/mule-base-template/ /home/claude/<project-name>
```

The base template includes:

**Build Configuration**:
- `pom.xml` - Maven configuration with plugin 4.1.1, Java 17, Mule 4.4.0
- `mule-artifact.json` - Root location (required for plugin 4.x)

**MuleSoft Configuration**:
- `src/main/mule/global.xml` - HTTP Listener, DB, error handling (with doc namespace)
- `src/main/mule/mule-artifact.json` - Studio location (required for import)
- `src/main/resources/config.properties` - Application properties
- `src/main/resources/log4j2.xml` - Logging configuration

**Anypoint Studio Files** (for project import):
- `.project` - Eclipse project descriptor
- `.classpath` - Java classpath with JavaSE-17 configuration
- `mule-project.xml` - MuleSoft Studio metadata
- `.settings/org.eclipse.jdt.core.prefs` - Java 17 compiler settings
- `.settings/org.eclipse.m2e.core.prefs` - Maven settings

**Other**:
- `.gitignore` - MuleSoft-specific ignore patterns

### 4. Implement Pattern

Following the pattern reference, create:

**Main Flow File** (`src/main/mule/<project-name>.xml`):
- All endpoint flows (GET, POST, PUT, PATCH, DELETE as needed)
- DataWeave transformations
- Error handlers
- Pattern-specific logic (async processing, pagination, etc.)

**Configuration Files**:
- Update `pom.xml` with pattern-specific dependencies
- Configure `global.xml` with necessary connectors
- Set appropriate properties in `config.properties`

**Documentation**:
- `README.md` - Pattern overview, endpoints, usage examples
- `API_EXAMPLES.md` - cURL/Postman examples for all endpoints
- `ARCHITECTURE.md` - Implementation details and design decisions

### 5. Quality Checks

Ensure the generated project:

**Critical Configuration Checks**:
- ✅ mule-maven-plugin version is 4.1.1 or higher
- ✅ Java 17 configured in pom.xml, .classpath, and .settings
- ✅ xmlns:doc namespace present in ALL XML files
- ✅ mule-artifact.json exists in BOTH root and src/main/mule/
- ✅ validation:is-number includes numberType attribute
- ✅ db:bulk-insert uses payload pattern (not variables)
- ✅ All Studio configuration files present (.project, .classpath, etc.)

**Code Quality Checks**:
- ✅ Uses only valid MuleSoft dependencies (check `references/mulesoft-dependencies.md`)
- ✅ Follows MuleSoft 4.x syntax (not Mule 3)
- ✅ Includes proper error handling for all flows
- ✅ DataWeave transformations are correct

**Documentation Checks**:
- ✅ README.md is complete
- ✅ API_EXAMPLES.md has all endpoints
- ✅ ARCHITECTURE.md explains design
- ✅ Includes practical usage examples

**Build Verification**:
- ✅ Run `mvn clean package` to verify compilation
- ✅ No dependency errors
- ✅ No XML schema validation errors

### 6. Package and Present

Create ZIP file and present to user:

```bash
cd /home/claude
zip -r <project-name>.zip <project-name>/ -x "*.DS_Store"
cp -r <project-name> /mnt/user-data/outputs/
cp <project-name>.zip /mnt/user-data/outputs/
```

Present with summary document explaining the pattern and how to use the project.

## Common MuleSoft Patterns

### Core Components Always Available

These are **part of Mule 4 core** and require NO dependencies:
- `<async>` - Asynchronous processing
- `<try>` - Error handling
- `<choice>` - Conditional logic
- `<foreach>` - Iteration
- `<parallel-foreach>` - Parallel iteration
- `<scatter-gather>` - Parallel execution with aggregation
- `<flow-ref>` - Sub-flow calls
- `<set-variable>` - Variable assignment
- `<set-payload>` - Payload manipulation
- `<logger>` - Logging

### Connectors Requiring Dependencies

See `references/mulesoft-dependencies.md` for complete list and versions.

Common connectors:
- HTTP Connector - `mule-http-connector`
- Database Connector - `mule-db-connector`
- Object Store - `mule-objectstore-connector`
- Salesforce - `mule-salesforce-connector`
- File Connector - `mule-file-connector`
- VM Connector - `mule-vm-connector`

## Pattern-Specific Guidelines

### For Async Patterns (Long Running Operations, Batch)

**Required Components**:
- Object Store for state persistence
- Async scope for background processing
- Proper HTTP status codes (202, 200, 404, 409)

**Implementation Pattern**:
```xml
<!-- Accept request -->
<http:listener path="/operations" method="POST"/>

<!-- Generate operation ID -->
<ee:transform>
  <ee:set-payload>
    {
      operationId: uuid(),
      status: "PENDING",
      ...
    }
  </ee:set-payload>
</ee:transform>

<!-- Store in Object Store -->
<os:store key="#[payload.operationId]"/>

<!-- Start async processing -->
<async>
  <flow-ref name="process-operation"/>
</async>

<!-- Return 202 Accepted immediately -->
```

### For Pagination Patterns

**Required Parameters**:
- `limit` - Number of items per page
- `offset` or `page` - Pagination position
- Optional: `cursor` for cursor-based pagination

**Response Format**:
```json
{
  "data": [...],
  "pagination": {
    "total": 1000,
    "limit": 20,
    "offset": 0,
    "hasMore": true,
    "nextUrl": "..."
  }
}
```

### For Versioning Patterns

**Common Approaches**:
1. URI versioning: `/v1/resource`, `/v2/resource`
2. Header versioning: `Accept: application/vnd.api.v2+json`
3. Query parameter: `/resource?version=2`

**MuleSoft Implementation**:
- Separate flows per version
- Shared sub-flows for common logic
- Version selection via HTTP Listener path or choice router

## Documentation Standards

Every generated project must include:

### README.md
- Pattern name and description
- Use cases and when to apply
- Endpoints list with HTTP methods
- Quick start instructions
- Configuration options
- Sample requests/responses

### API_EXAMPLES.md
- cURL examples for each endpoint
- Expected responses for different scenarios
- Error response examples
- Complete workflow examples

### ARCHITECTURE.md
- Component diagram
- Flow descriptions
- State management approach
- Scalability considerations
- Design decisions and trade-offs

## Error Handling Requirements

All flows must handle:

1. **Validation Errors** (400 Bad Request)
   - Missing required fields
   - Invalid data formats
   - Business rule violations

2. **Resource Not Found** (404)
   - Invalid resource IDs
   - Deleted or non-existent resources

3. **Conflict Errors** (409)
   - Duplicate resources
   - State conflicts (e.g., operation not ready)

4. **Server Errors** (500)
   - Unexpected exceptions
   - External system failures

**Standard Error Response Format**:
```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable description",
  "timestamp": "ISO-8601 datetime",
  "details": { /* optional */ }
}
```

## Best Practices

### DataWeave Transformations
- Use meaningful variable names
- Comment complex transformations
- Validate data before processing
- Handle null values gracefully

### Flow Design
- Keep flows focused and single-purpose
- Use sub-flows for reusable logic
- Add descriptive `doc:name` attributes
- Log important operations

### Configuration Management
- Use property placeholders for all configuration
- Never hardcode URLs, ports, or credentials
- Provide sensible defaults in `config.properties`
- Document all properties

### Performance
- Use async processing for long operations
- Implement pagination for large datasets
- Configure appropriate Object Store TTL
- Consider caching strategies

## Testing Recommendations

Include in documentation:

1. **Unit Testing** - Individual flow testing
2. **Integration Testing** - End-to-end scenarios
3. **Performance Testing** - Load and stress tests
4. **Example Test Cases** - For each endpoint

## Common Pitfalls to Avoid

### Critical Build/Runtime Issues

1. ❌ **Don't use mule-maven-plugin 3.8.0 or older**
   - Incompatible with Java 17 and Maven 3.9+
   - Use 4.1.1 or higher
   - Error: `BasicRepositoryConnectorFactory missing`

2. ❌ **Don't forget xmlns:doc namespace**
   - Required in ALL XML configuration files
   - Error: `The prefix "doc" for attribute "doc:name" is not bound`
   - Add: `xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"`

3. ❌ **Don't use variables in db:bulk-insert**
   - Bulk-insert consumes PAYLOAD, not variables
   - Don't use ee:variables inside bulk-insert
   - Don't use bulkMode attribute (doesn't exist)
   - Don't use db:input-parameters with bulk-insert

4. ❌ **Don't forget numberType in validation:is-number**
   - Required in validation module 2.0.1+
   - Valid values: INTEGER, DECIMAL, NUMBER
   - Error: `Attribute 'numberType' must appear`

5. ❌ **Don't create only one mule-artifact.json**
   - Required in TWO locations: root AND src/main/mule/
   - Plugin 4.x needs root location
   - Studio needs src/main/mule/ location

6. ❌ **Don't skip Anypoint Studio configuration files**
   - Required for Studio import: .project, .classpath, mule-project.xml
   - Required for Java 17: .settings/org.eclipse.jdt.core.prefs
   - Configure JavaSE-17 in .classpath

### Code Quality Issues

7. ❌ **Don't use `mule-async-module`** - `<async>` is in core
8. ❌ **Don't use Mule 3 syntax** - Ensure Mule 4 compatibility
9. ❌ **Don't hardcode values** - Use property placeholders
10. ❌ **Don't skip error handling** - Every flow needs it
11. ❌ **Don't forget documentation** - Essential for users
12. ❌ **Don't use non-existent dependencies** - Verify all exist

## When to Use Which Pattern

**Use Long Running Operations when**:
- Operations take >2 seconds
- Processing is computationally expensive
- User shouldn't wait for completion
- Examples: Report generation, data exports, batch imports

**Use Pagination when**:
- Returning lists of resources
- Dataset size is unbounded or large
- Client needs control over result size
- Examples: Search results, user lists, transaction history

**Use Partial Updates when**:
- Resources have many fields
- Clients need to update specific fields only
- Full replacement is inefficient
- Examples: User profiles, settings, large documents

**Use Batch Operations when**:
- Multiple resources need same operation
- Reducing API calls improves performance
- Atomicity is desired across operations
- Examples: Bulk imports, mass updates, multi-delete

**Use Versioning when**:
- API changes break backward compatibility
- Multiple API versions must coexist
- Gradual migration is required
- Examples: Schema changes, endpoint restructuring

## Advanced Features

For complex patterns, consult:
- `references/advanced-dataweave.md` - Complex transformations
- `references/security-patterns.md` - Authentication/authorization
- `references/caching-strategies.md` - Performance optimization
- `references/monitoring-logging.md` - Observability

## Output Checklist

Before presenting the project:

**Critical Configuration**:
- [ ] mule-maven-plugin version is 4.1.1+
- [ ] Java 17 in pom.xml, .classpath, .settings
- [ ] xmlns:doc in ALL XML files (global.xml, main flow)
- [ ] mule-artifact.json in root directory
- [ ] mule-artifact.json in src/main/mule/
- [ ] Studio files present (.project, .classpath, mule-project.xml, .settings/)
- [ ] validation:is-number has numberType attribute
- [ ] db:bulk-insert uses payload pattern (if applicable)

**Code Quality**:
- [ ] All flows compile without errors (run mvn clean package)
- [ ] No invalid dependencies in pom.xml
- [ ] No XML schema validation errors
- [ ] Object Store configured if needed
- [ ] HTTP status codes are appropriate
- [ ] Error handling on all flows
- [ ] DataWeave transformations are correct

**Documentation**:
- [ ] README.md is complete
- [ ] API_EXAMPLES.md has all endpoints
- [ ] ARCHITECTURE.md explains design
- [ ] config.properties has all settings

**Delivery**:
- [ ] Project is zipped and in outputs folder
- [ ] Summary document is created
- [ ] Build verification completed successfully

## Pattern Combination

Multiple patterns can be combined:

**Example: Paginated List with Filtering**
- Pagination for large results
- List Filtering for query parameters
- Combine both patterns in single endpoint

**Example: Async Batch Operations**
- Batch Operations for bulk processing
- Long Running Operations for async handling
- Status endpoint to check batch progress

When combining patterns, reference multiple pattern files and merge their implementations logically.

## Summary

This skill enables rapid generation of production-ready MuleSoft projects implementing industry-standard API patterns. Always:

1. Understand the user's use case
2. Select the appropriate pattern(s)
3. Load relevant reference documentation
4. **Use the corrected base template** from `assets/mule-base-template/`
5. Generate complete, documented, working code
6. **Verify ALL critical requirements** (plugin version, Java 17, doc namespace, etc.)
7. **Test with `mvn clean package`** to ensure compilation
8. Present to user with documentation

The goal is to provide users with immediately usable, well-documented MuleSoft projects that follow API design best practices and compile successfully on first try.

## Version History & Updates

**Latest Update**: Critical corrections applied based on real-world troubleshooting:
- ✅ Updated base template to plugin 4.1.1 and Java 17
- ✅ Added xmlns:doc namespace to all XML templates
- ✅ Corrected db:bulk-insert pattern (payload-based)
- ✅ Added dual mule-artifact.json locations
- ✅ Added complete Anypoint Studio configuration files
- ✅ Added validation:is-number numberType requirement
- ✅ Created troubleshooting guides for common issues

**Troubleshooting Guides Available**:
- Maven build errors and solutions
- Validation module best practices
- Bulk operations patterns

All generated projects now include these corrections by default.