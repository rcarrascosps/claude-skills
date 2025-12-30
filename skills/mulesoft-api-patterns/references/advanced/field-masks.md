# Field Masks - Advanced Partial Update Feature

## Overview

**Field Masks** provide explicit control over which fields should be updated in a partial update operation, following Google's API Design Guide standard.

**Key Difference from Standard PATCH**:
- **Standard PATCH**: Infers which fields to update from request body
- **Field Mask PATCH**: Explicitly declares which fields to update via `updateMask` parameter

## Why Use Field Masks?

### 1. Explicit Intent
The server knows exactly which fields the client wants to update, eliminating ambiguity.

### 2. Clear Null Semantics
```json
// Standard PATCH - Ambiguous
{"phone": null}  // Set to null? Or don't update?

// Field Mask PATCH - Clear
{"updateMask": "phone", "phone": null}  // Explicitly set to null
{"updateMask": "email", "email": "new@example.com"}  // Phone not updated
```

### 3. Enhanced Security
- Server validates client is allowed to update fields in mask
- Prevents accidental updates to sensitive fields
- Enables field-level permissions

### 4. Industry Standard
Used by:
- Google Cloud APIs
- Gmail API
- Google Drive API
- Microsoft Graph API

## Implementation Approaches

### Approach 1: Query Parameter (Recommended)

```http
PATCH /api/v1/resources/{id}:updateWithMask?updateMask=field1,field2
Content-Type: application/json

{
  "field1": "value1",
  "field2": "value2",
  "field3": "ignored"
}
```

**Advantages**:
- Clean separation of metadata (mask) and data (body)
- Easy to log and monitor
- RESTful and follows Google's standard

### Approach 2: Request Body

```http
PATCH /api/v1/resources/{id}:updateWithMask
Content-Type: application/json

{
  "updateMask": "field1,field2",
  "field1": "value1",
  "field2": "value2"
}
```

**Advantages**:
- Everything in one place
- No URL encoding issues
- Easier for some client libraries

**Implementation Tip**: Support both, with query parameter taking precedence.

## MuleSoft Implementation

### Endpoint Configuration

```xml
<flow name="patch-with-fieldmask-flow" doc:name="Patch with Field Mask Flow">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources/{id}:updateWithMask"
                   allowedMethods="PATCH"
                   doc:name="PATCH /api/v1/resources/{id}:updateWithMask"/>

    <!-- Implementation flows here -->
</flow>
```

**Note**: Use `:updateWithMask` suffix to distinguish from standard PATCH endpoint.

### Extract Field Mask

```xml
<ee:transform doc:name="Extract Field Mask and Updates">
    <ee:variables>
        <ee:set-variable variableName="fieldMask"><![CDATA[%dw 2.0
output application/java
---
// Check query parameter first, then body
if (attributes.queryParams.updateMask != null)
    attributes.queryParams.updateMask splitBy ","
else if (payload.updateMask != null)
    payload.updateMask splitBy ","
else if (payload.fieldMask != null)
    payload.fieldMask splitBy ","
else
    []]]></ee:set-variable>

        <ee:set-variable variableName="updateData"><![CDATA[%dw 2.0
output application/java
---
// Remove updateMask/fieldMask from payload if present
payload - "updateMask" - "fieldMask"]]></ee:set-variable>
    </ee:variables>
</ee:transform>
```

### Validate Field Mask

```xml
<!-- 1. Check field mask is provided -->
<choice doc:name="Field Mask Provided?">
    <when expression="#[sizeOf(vars.fieldMask) == 0]">
        <ee:transform doc:name="400 - Missing Field Mask">
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    "error": "MISSING_FIELD_MASK",
    "message": "Field mask is required",
    "timestamp": now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus">400</ee:set-variable>
            </ee:variables>
        </ee:transform>
    </when>
</choice>

<!-- 2. Validate field names -->
<ee:transform doc:name="Validate Field Names">
    <ee:variables>
        <ee:set-variable variableName="allowedFields"><![CDATA[%dw 2.0
output application/java
---
["email", "firstName", "lastName", "phone", "status"]]]></ee:set-variable>

        <ee:set-variable variableName="invalidFields"><![CDATA[%dw 2.0
output application/java
---
vars.fieldMask filter (not (vars.allowedFields contains $))]]></ee:set-variable>
    </ee:variables>
</ee:transform>

<choice doc:name="Invalid Fields?">
    <when expression="#[sizeOf(vars.invalidFields) > 0]">
        <ee:transform doc:name="400 - Invalid Fields">
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    "error": "INVALID_FIELD_MASK",
    "message": "Invalid field(s): " ++ (vars.invalidFields joinBy ", "),
    "allowedFields": vars.allowedFields,
    "timestamp": now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus">400</ee:set-variable>
            </ee:variables>
        </ee:transform>
    </when>
</choice>
```

### Merge Logic with Field Mask

```dataweave
%dw 2.0
output application/java
var currentData = payload[0]
var updates = vars.updateData
var mask = vars.fieldMask
---
{
    // Only update fields specified in the field mask
    email: if (mask contains "email")
           (updates.email default currentData.email)
           else currentData.email,

    firstName: if (mask contains "firstName")
               (if (updates.firstName != null) updates.firstName else currentData.firstName)
               else currentData.firstName,

    lastName: if (mask contains "lastName")
              (if (updates.lastName != null) updates.lastName else currentData.lastName)
              else currentData.lastName,

    phone: if (mask contains "phone")
           (if (updates.phone != null) updates.phone else currentData.phone)
           else currentData.phone,

    status: if (mask contains "status")
            (updates.status default currentData.status)
            else currentData.status
}
```

**Key Logic**:
- **Field in mask** → Use value from updates (or current if not provided in body)
- **Field NOT in mask** → Always use current value (ignore updates)

## Complete Flow Pattern

```
1. HTTP Listener receives PATCH with updateMask
   ↓
2. Extract fieldMask from query param or body
   ↓
3. Extract updateData (body without updateMask field)
   ↓
4. Validate fieldMask is provided (400 if missing)
   ↓
5. Validate all fields in mask are allowed (400 if invalid)
   ↓
6. Validate field-specific rules (e.g., email format)
   ↓
7. Fetch current resource from database
   ↓
8. Check if resource exists (404 if not)
   ↓
9. Merge using field mask logic (DataWeave)
   ↓
10. Update database
   ↓
11. Fetch updated resource
   ↓
12. Return 200 OK with updated resource
```

## Error Handling

### Missing Field Mask (400)

```json
{
  "error": "MISSING_FIELD_MASK",
  "message": "Field mask is required. Provide via '?updateMask=field1,field2' or in request body",
  "timestamp": "2025-01-15T10:00:00Z",
  "example": {
    "queryParam": "PATCH /api/v1/resources/{id}:updateWithMask?updateMask=field1,field2",
    "requestBody": {
      "updateMask": "field1,field2",
      "field1": "value1"
    }
  }
}
```

### Invalid Field Names (400)

```json
{
  "error": "INVALID_FIELD_MASK",
  "message": "Invalid field(s) in updateMask: invalidField",
  "allowedFields": ["email", "firstName", "lastName", "phone", "status"],
  "timestamp": "2025-01-15T10:00:00Z"
}
```

### Validation Errors (400)

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Invalid email format",
  "timestamp": "2025-01-15T10:00:00Z"
}
```

## Usage Examples

### Example 1: Update Single Field

**Request**:
```bash
curl -X PATCH "http://localhost:8081/api/v1/users/123?updateMask=phone" \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "+1-555-9999",
    "firstName": "Ignored"
  }'
```

**Behavior**:
- ✅ Updates phone
- ❌ Ignores firstName (not in mask)
- ✅ Preserves all other fields

### Example 2: Set Field to Null

**Request**:
```bash
curl -X PATCH "http://localhost:8081/api/v1/users/123?updateMask=phone" \
  -H "Content-Type: application/json" \
  -d '{"phone": null}'
```

**Behavior**:
- ✅ Explicitly sets phone to null
- ✅ Preserves all other fields

### Example 3: Update Multiple Fields

**Request**:
```bash
curl -X PATCH "http://localhost:8081/api/v1/users/123?updateMask=firstName,lastName,status" \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Doe",
    "status": "INACTIVE"
  }'
```

### Example 4: Using Request Body for Mask

**Request**:
```bash
curl -X PATCH "http://localhost:8081/api/v1/users/123:updateWithMask" \
  -H "Content-Type: application/json" \
  -d '{
    "updateMask": "email",
    "email": "newemail@example.com"
  }'
```

## Comparison: Standard PATCH vs Field Mask

| Scenario | Standard PATCH | Field Mask PATCH |
|----------|---------------|------------------|
| **Update phone only** | `{"phone": "+1-555-9999"}` | `updateMask=phone` + `{"phone": "+1-555-9999"}` |
| **Set phone to null** | `{"phone": null}` (ambiguous) | `updateMask=phone` + `{"phone": null}` (clear) |
| **Extra fields in body** | All processed | Only mask fields processed |
| **Null handling** | Ambiguous | Explicit and clear |
| **Security** | Process all sent fields | Process only mask fields |

## Best Practices

### 1. Support Both Query Param and Body
```dataweave
if (attributes.queryParams.updateMask != null)
    attributes.queryParams.updateMask
else
    payload.updateMask
```

### 2. Validate Allowed Fields
Maintain a whitelist of updatable fields:
```dataweave
var allowedFields = ["email", "firstName", "lastName", "phone", "status"]
```

### 3. Field-Level Permissions
```dataweave
// Example: Admin can update status, regular users cannot
var allowedFields = if (vars.userRole == "ADMIN")
    ["email", "firstName", "lastName", "phone", "status"]
else
    ["email", "firstName", "lastName", "phone"]
```

### 4. Log Field Masks
```xml
<logger level="INFO" message="#['Field Mask: ' ++ (vars.fieldMask joinBy ', ')]"/>
```

### 5. Clear Error Messages
Provide helpful error messages with examples when validation fails.

## Advanced: Nested Field Masks

For complex resources with nested objects:

```
updateMask=address.city,address.zipCode,contact.phone
```

**Implementation**:
```dataweave
{
    address: {
        city: if (mask contains "address.city") updates.address.city else current.address.city,
        zipCode: if (mask contains "address.zipCode") updates.address.zipCode else current.address.zipCode,
        street: current.address.street  // Not in mask, preserve
    }
}
```

## Testing Checklist

- [ ] Update with query parameter updateMask
- [ ] Update with request body updateMask
- [ ] Update with request body fieldMask (alternative name)
- [ ] Query parameter takes precedence over body
- [ ] Missing field mask returns 400
- [ ] Invalid field name in mask returns 400
- [ ] Fields in mask are updated
- [ ] Fields NOT in mask are preserved
- [ ] Set field to null with mask
- [ ] Extra fields in body are ignored
- [ ] Valid email format when email in mask
- [ ] Invalid email format returns 400

## Performance Impact

**Additional Processing**:
- Field mask parsing: ~1-2ms
- Field validation: ~1-5ms
- Conditional merge logic: ~2-5ms

**Total Overhead**: ~5-12ms per request (negligible)

## Security Considerations

1. **Whitelist Validation**: Always validate against allowed fields
2. **Field-Level Permissions**: Check user permissions for each field in mask
3. **Sensitive Fields**: Never allow sensitive fields in mask (e.g., id, password)
4. **Audit Logging**: Log which fields were updated for security audit
5. **Rate Limiting**: Same as standard PATCH

## When to Use

**Use Field Masks when**:
- Need explicit control over field updates
- Null value semantics must be clear
- Field-level security/permissions required
- Following Google API standards
- Complex clients need precise control

**Use Standard PATCH when**:
- Simple updates with clear intent
- Null ambiguity is not a concern
- No field-level security needed
- Simpler client implementation preferred

## Integration with Standard PATCH

**Best Practice**: Offer both endpoints

```
# Standard PATCH (implicit)
PATCH /api/v1/users/{id}

# Field Mask PATCH (explicit)
PATCH /api/v1/users/{id}:updateWithMask?updateMask=...
```

Clients choose based on their needs:
- Simple updates → Standard PATCH
- Precise control → Field Mask PATCH

## References

- [Google API Design Guide - Update](https://cloud.google.com/apis/design/standard_methods#update)
- [Field Masks in Protocol Buffers](https://protobuf.dev/reference/protobuf/google.protobuf/#field-mask)
- [API Design Patterns by JJ Geewax](https://www.manning.com/books/api-design-patterns)
- [RFC 5789 - HTTP PATCH Method](https://tools.ietf.org/html/rfc5789)

---

**Pattern Reference**: See `references/patterns/partial-updates.md`
**Troubleshooting**: See `references/troubleshooting/`
