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
5. **Package and present** the complete project to user

## Supported Patterns

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
- Standard `pom.xml` with common dependencies
- `global.xml` with HTTP Listener configuration
- `config.properties` template
- `log4j2.xml` for logging
- `.gitignore` for MuleSoft projects

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
- ✅ Uses only valid MuleSoft dependencies (check `references/mulesoft-dependencies.md`)
- ✅ Follows MuleSoft 4.x syntax (not Mule 3)
- ✅ Includes proper error handling for all flows
- ✅ Has complete documentation
- ✅ Compiles without dependency errors
- ✅ Includes practical usage examples

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

1. ❌ **Don't use `mule-async-module`** - `<async>` is in core
2. ❌ **Don't use Mule 3 syntax** - Ensure Mule 4 compatibility
3. ❌ **Don't hardcode values** - Use property placeholders
4. ❌ **Don't skip error handling** - Every flow needs it
5. ❌ **Don't forget documentation** - Essential for users
6. ❌ **Don't use non-existent dependencies** - Verify all exist

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

- [ ] All flows compile without errors
- [ ] No invalid dependencies in pom.xml
- [ ] Object Store configured if needed
- [ ] HTTP status codes are appropriate
- [ ] Error handling on all flows
- [ ] DataWeave transformations are correct
- [ ] README.md is complete
- [ ] API_EXAMPLES.md has all endpoints
- [ ] ARCHITECTURE.md explains design
- [ ] config.properties has all settings
- [ ] Project is zipped and in outputs folder
- [ ] Summary document is created

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
4. Generate complete, documented, working code
5. Verify quality before presenting

The goal is to provide users with immediately usable, well-documented MuleSoft projects that follow API design best practices.