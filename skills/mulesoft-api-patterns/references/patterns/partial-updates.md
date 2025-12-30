# Partial Update (PATCH) Pattern - MuleSoft Implementation Reference

## Pattern Overview

The **Partial Update** pattern allows clients to update specific fields of a resource without sending the entire resource representation. This is implemented using the HTTP PATCH method.

**Key Concept**: Only modify fields provided in the request, preserving all other fields.

## When to Use This Pattern

✅ **Use Partial Updates when**:
- Resources have many fields
- Clients need to update only specific fields
- Full resource replacement (PUT) is inefficient
- Bandwidth optimization is important
- Mobile applications with limited connectivity
- Incremental form completion

❌ **Don't use when**:
- Entire resource needs replacement → Use PUT
- Creating new resources → Use POST
- All-or-nothing atomic updates → Consider PUT

## Required Endpoints

### 1. Standard PATCH (Implicit Field Selection)

```
PATCH /api/v1/{resource}/{id}
Content-Type: application/json

{
  "field1": "newValue1",
  "field2": "newValue2"
}
```

**Behavior**: Updates only fields present in request body.

### 2. PATCH with Field Mask (Explicit Field Selection) - ADVANCED

```
PATCH /api/v1/{resource}/{id}:updateWithMask?updateMask=field1,field2
Content-Type: application/json

{
  "field1": "newValue1",
  "field2": "newValue2",
  "field3": "ignored"
}
```

**Behavior**: Updates only fields specified in `updateMask`, ignores other fields in body.

See `references/advanced/field-masks.md` for complete Field Mask documentation.

## Resource Schema Example

```json
{
  "id": "UUID (auto-generated)",
  "email": "string (required, unique)",
  "firstName": "string (optional)",
  "lastName": "string (optional)",
  "phone": "string (optional)",
  "status": "string (default: ACTIVE)",
  "createdAt": "timestamp (auto-generated)",
  "updatedAt": "timestamp (auto-updated on each update)"
}
```

## MuleSoft Implementation

### Database Configuration

```xml
<!-- global.xml -->
<db:config name="Database_Config" doc:name="Database Config">
    <db:generic-connection url="${db.url}"
                          driverClassName="${db.driver}"
                          user="${db.user}"
                          password="${db.password}"/>
</db:config>
```

### PATCH Flow Implementation

```xml
<flow name="patch-resource-flow" doc:name="Patch Resource Flow">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources/{id}"
                   allowedMethods="PATCH"
                   doc:name="PATCH /api/v1/resources/{id}"/>

    <!-- Extract variables -->
    <set-variable variableName="resourceId" value="#[attributes.uriParams.id]"/>
    <set-variable variableName="updateFields" value="#[payload]"/>

    <!-- Get current resource -->
    <flow-ref name="get-resource-by-id-subflow"/>

    <!-- Check if resource exists -->
    <choice doc:name="Resource Exists?">
        <when expression="#[sizeOf(payload) == 0]">
            <!-- Return 404 Not Found -->
        </when>
        <otherwise>
            <!-- Merge and update -->
            <flow-ref name="merge-and-update-subflow"/>
        </otherwise>
    </choice>
</flow>
```

### Critical: Merge Logic (DataWeave)

The heart of the pattern is the merge transformation:

```dataweave
%dw 2.0
output application/java
var currentData = payload[0]
var updates = vars.updateFields
---
{
    // Only update fields that are provided in the request
    field1: updates.field1 default currentData.field1,
    field2: if (updates.field2 != null)
            updates.field2
            else currentData.field2,
    field3: updates.field3 default currentData.field3,
    // ... repeat for all fields
}
```

**Key Points**:
- Use `default` for fields that should use current value if not provided
- Use `if/else` for fields that can be explicitly set to null
- Always preserve fields not in the update request

### Database Update

```xml
<db:update config-ref="Database_Config" doc:name="Update Resource">
    <db:sql><![CDATA[UPDATE resources
SET field1 = :field1,
    field2 = :field2,
    field3 = :field3,
    updatedAt = CURRENT_TIMESTAMP
WHERE id = :id]]></db:sql>
    <db:input-parameters><![CDATA[#[payload ++ {id: vars.resourceId}]]]></db:input-parameters>
</db:update>
```

## HTTP Status Codes

- **200 OK**: Resource successfully updated
- **400 Bad Request**: Invalid input (validation errors)
- **404 Not Found**: Resource does not exist
- **409 Conflict**: Duplicate unique field (e.g., email)
- **500 Internal Server Error**: Unexpected server error

## Error Response Format

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable description",
  "timestamp": "ISO-8601 datetime"
}
```

## Validation Requirements

### Input Validations

1. **Field Format Validation**:
   ```xml
   <validation:is-email email="#[payload.email]"
                       message="Invalid email format"
                       doc:name="Validate Email Format"/>
   ```

2. **Required Field Validation** (for updates that require certain fields):
   ```xml
   <validation:is-not-empty-string value="#[payload.requiredField]"
                                   message="Field is required"/>
   ```

3. **Custom Business Logic Validation**:
   - Check field value ranges
   - Validate relationships
   - Verify permissions

### Resource Existence Validation

Always verify the resource exists before updating:

```dataweave
<choice doc:name="Resource Exists?">
    <when expression="#[sizeOf(payload) == 0]">
        <ee:transform doc:name="Transform to 404">
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    "error": "RESOURCE_NOT_FOUND",
    "message": "Resource with ID " ++ vars.resourceId ++ " not found",
    "timestamp": now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus">404</ee:set-variable>
            </ee:variables>
        </ee:transform>
    </when>
</choice>
```

## Complete Flow Pattern

```
1. HTTP Listener receives PATCH request
   ↓
2. Extract resource ID and update fields
   ↓
3. Validate input (email format, etc.)
   ↓
4. Fetch current resource from database
   ↓
5. Check if resource exists (404 if not)
   ↓
6. Merge current data with updates (DataWeave)
   ↓
7. Update database with merged data
   ↓
8. Fetch updated resource
   ↓
9. Return 200 OK with updated resource
```

## Advanced Feature: Field Masks

For explicit control over which fields to update, implement Field Masks:

```
PATCH /api/v1/resources/{id}:updateWithMask?updateMask=field1,field2
```

**Benefits**:
- Explicit field selection
- Clear null semantics
- Enhanced security
- Industry standard (Google APIs)

See `references/advanced/field-masks.md` for complete implementation guide.

## Example Use Cases

### 1. User Profile Update
```bash
PATCH /api/v1/users/123
{
  "phone": "+1-555-9999"
}
```
Updates only phone, preserves all other fields.

### 2. Status Change
```bash
PATCH /api/v1/users/123
{
  "status": "INACTIVE"
}
```
Updates only status field.

### 3. Multiple Field Update
```bash
PATCH /api/v1/users/123
{
  "firstName": "John",
  "lastName": "Doe",
  "phone": "+1-555-1234"
}
```
Updates three fields, preserves email and status.

### 4. Set Field to Null
```bash
PATCH /api/v1/users/123
{
  "phone": null
}
```
Explicitly clears the phone field.

## Testing Checklist

- [ ] Update single field
- [ ] Update multiple fields
- [ ] Update with null value
- [ ] Update non-existent resource (404)
- [ ] Update with invalid email format (400)
- [ ] Update with duplicate unique field (409)
- [ ] Fields not in request are preserved
- [ ] updatedAt timestamp is updated
- [ ] createdAt timestamp is NOT updated

## Common Pitfalls

❌ **Don't**: Update all fields to null if not provided
```dataweave
// WRONG
{
    email: updates.email,  // Will be null if not provided!
    firstName: updates.firstName
}
```

✅ **Do**: Preserve current values
```dataweave
// CORRECT
{
    email: updates.email default currentData.email,
    firstName: updates.firstName default currentData.firstName
}
```

❌ **Don't**: Forget to update the updatedAt timestamp

✅ **Do**: Always update timestamp on modifications

❌ **Don't**: Skip resource existence check

✅ **Do**: Always verify resource exists before updating

## Dependencies Required

```xml
<!-- Database Connector -->
<dependency>
    <groupId>org.mule.connectors</groupId>
    <artifactId>mule-db-connector</artifactId>
    <version>1.13.4</version>
    <classifier>mule-plugin</classifier>
</dependency>

<!-- Validation Module -->
<dependency>
    <groupId>org.mule.modules</groupId>
    <artifactId>mule-validation-module</artifactId>
    <version>2.0.1</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

## Performance Considerations

- **Database Queries**: 3 queries per PATCH (SELECT + UPDATE + SELECT)
- **Optimization**: Remove final SELECT if only HTTP 200 (no body) is acceptable
- **Caching**: Consider caching frequently accessed resources
- **Connection Pooling**: Configure database connection pool appropriately

## Security Considerations

1. **Field-Level Permissions**: Validate user can update requested fields
2. **Sensitive Fields**: Prevent updates to sensitive fields (e.g., id, createdAt)
3. **Input Sanitization**: Validate and sanitize all input
4. **Audit Logging**: Log all update operations for security audit

## Related Patterns

- **PUT (Full Update)**: Complete resource replacement
- **Field Masks**: Explicit field selection (advanced)
- **Conditional Updates**: Using ETags for optimistic locking
- **Batch Updates**: Updating multiple resources in one request

## References

- [API Design Patterns by JJ Geewax](https://www.manning.com/books/api-design-patterns)
- [RFC 5789 - HTTP PATCH Method](https://tools.ietf.org/html/rfc5789)
- [Google API Design Guide - Update](https://cloud.google.com/apis/design/standard_methods#update)

---

**Implementation Template**: Use `assets/mule-base-template/` as starting point
**Advanced Features**: See `references/advanced/field-masks.md`
**Troubleshooting**: See `references/troubleshooting/` for common issues
