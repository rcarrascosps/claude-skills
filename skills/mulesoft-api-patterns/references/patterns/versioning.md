# API Versioning Pattern - MuleSoft Implementation Reference

## Pattern Overview

The **API Versioning** pattern enables multiple versions of an API to coexist, allowing backward-incompatible changes while supporting existing clients. This pattern is essential for evolving APIs without breaking existing integrations.

**Key Concept**: Provide a mechanism for clients to specify which API version they want to use, allowing old and new versions to run simultaneously.

## When to Use This Pattern

✅ **Use API Versioning when**:
- Making backward-incompatible changes to the API
- Changing request/response schemas significantly
- Removing or renaming endpoints
- Changing endpoint behavior fundamentally
- Multiple client versions need support simultaneously
- Gradual migration of clients is required
- Long-term API evolution is planned

❌ **Don't use when**:
- Changes are backward-compatible (add optional fields instead)
- Only one client exists and can be updated immediately
- API is internal and all clients can be coordinated
- Over-engineering for premature versioning

## What Requires a New Version

### Breaking Changes (Require New Version)

- ❌ Removing fields from responses
- ❌ Removing endpoints
- ❌ Renaming fields or endpoints
- ❌ Changing field data types
- ❌ Making optional fields required
- ❌ Changing endpoint behavior significantly
- ❌ Changing error response formats

### Non-Breaking Changes (No Version Needed)

- ✅ Adding new optional fields to requests
- ✅ Adding new fields to responses
- ✅ Adding new endpoints
- ✅ Making required fields optional
- ✅ Expanding enum values
- ✅ Bug fixes that don't change behavior

## Versioning Strategies

### 1. URI Versioning (Recommended)

**Version in URL path** - Most common and visible approach.

```
GET /api/v1/users
GET /api/v2/users
```

**Pros**:
- Explicit and visible
- Easy to route in API gateway
- Browser-friendly
- Easy to test with curl/Postman
- Clear documentation

**Cons**:
- Appears to create "different" resources
- URL changes with versions

### 2. Header Versioning

**Version in custom header**:

```
GET /api/users
X-API-Version: 2
```

Or **Accept header** (content negotiation):

```
GET /api/users
Accept: application/vnd.myapi.v2+json
```

**Pros**:
- Clean URLs
- RESTful resource naming
- Supports multiple versions per resource

**Cons**:
- Less visible
- Harder to test in browser
- Requires header support in all clients

### 3. Query Parameter Versioning

**Version in query parameter**:

```
GET /api/users?version=2
```

**Pros**:
- Simple to implement
- Easy to override
- URL-visible

**Cons**:
- Clutters query parameters
- Can be accidentally cached
- Not recommended by REST best practices

## MuleSoft Implementation

### URI Versioning Implementation

#### Approach 1: Separate Flows per Version

```xml
<!-- Version 1 Flow -->
<flow name="get-users-v1" doc:name="Get Users V1">
    <http:listener config-ref="HTTP_Listener_config"
                   path="/api/v1/users"
                   allowedMethods="GET"
                   doc:name="GET /api/v1/users"/>

    <flow-ref name="get-users-v1-logic" doc:name="V1 Logic"/>

    <ee:transform doc:name="Transform to V1 Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
payload map (user) -> {
    id: user.id,
    name: user.fullName,  // V1 field name
    email: user.email,
    status: user.status
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>

<!-- Version 2 Flow -->
<flow name="get-users-v2" doc:name="Get Users V2">
    <http:listener config-ref="HTTP_Listener_config"
                   path="/api/v2/users"
                   allowedMethods="GET"
                   doc:name="GET /api/v2/users"/>

    <flow-ref name="get-users-v2-logic" doc:name="V2 Logic"/>

    <ee:transform doc:name="Transform to V2 Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
payload map (user) -> {
    id: user.id,
    firstName: user.firstName,  // V2 split name
    lastName: user.lastName,
    email: user.email,
    status: user.status,
    createdAt: user.createdAt,  // V2 added field
    metadata: {                  // V2 nested object
        lastLogin: user.lastLogin,
        loginCount: user.loginCount
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

#### Approach 2: Shared Logic with Version Transformations

```xml
<flow name="get-users-versioned" doc:name="Get Users (Versioned)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="/api/{version}/users"
                   allowedMethods="GET"
                   doc:name="GET /api/{version}/users"/>

    <!-- Extract version -->
    <set-variable variableName="apiVersion" value="#[attributes.uriParams.version]"/>

    <!-- Validate version -->
    <choice doc:name="Valid Version?">
        <when expression="#[!(['v1', 'v2'] contains vars.apiVersion)]">
            <ee:transform doc:name="Transform to 400">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "INVALID_VERSION",
    message: "Supported versions: v1, v2",
    timestamp: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="httpStatus">400</ee:set-variable>
                </ee:variables>
            </ee:transform>
        </when>
        <otherwise>
            <!-- Shared business logic -->
            <flow-ref name="get-users-core-logic" doc:name="Core Logic"/>

            <!-- Version-specific transformation -->
            <choice doc:name="Route by Version">
                <when expression="#[vars.apiVersion == 'v1']">
                    <flow-ref name="transform-users-v1" doc:name="Transform V1"/>
                </when>
                <when expression="#[vars.apiVersion == 'v2']">
                    <flow-ref name="transform-users-v2" doc:name="Transform V2"/>
                </when>
            </choice>
        </otherwise>
    </choice>
</flow>

<sub-flow name="get-users-core-logic" doc:name="Get Users Core Logic">
    <!-- Shared database query and business logic -->
    <db:select config-ref="Database_Config" doc:name="Get Users">
        <db:sql><![CDATA[SELECT * FROM users ORDER BY created_at DESC]]></db:sql>
    </db:select>
</sub-flow>

<sub-flow name="transform-users-v1" doc:name="Transform Users V1">
    <ee:transform doc:name="V1 Response Format">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
payload map (user) -> {
    id: user.id,
    name: user.first_name ++ " " ++ user.last_name,
    email: user.email,
    status: user.status
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</sub-flow>

<sub-flow name="transform-users-v2" doc:name="Transform Users V2">
    <ee:transform doc:name="V2 Response Format">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
payload map (user) -> {
    id: user.id,
    firstName: user.first_name,
    lastName: user.last_name,
    email: user.email,
    status: user.status,
    createdAt: user.created_at as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"},
    metadata: {
        lastLogin: user.last_login,
        loginCount: user.login_count
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</sub-flow>
```

### Header Versioning Implementation

```xml
<flow name="get-users-header-versioning" doc:name="Get Users (Header Versioning)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="/api/users"
                   allowedMethods="GET"
                   doc:name="GET /api/users"/>

    <!-- Extract version from header -->
    <set-variable variableName="apiVersion"
                  value="#[attributes.headers['x-api-version'] default 'v1']"
                  doc:name="Extract API Version"/>

    <!-- Validate version -->
    <validation:is-true expression="#[['v1', 'v2'] contains vars.apiVersion]"
                        message="Invalid API version. Supported: v1, v2"/>

    <!-- Shared business logic -->
    <flow-ref name="get-users-core-logic" doc:name="Core Logic"/>

    <!-- Version-specific transformation -->
    <choice doc:name="Route by Version">
        <when expression="#[vars.apiVersion == 'v1']">
            <flow-ref name="transform-users-v1" doc:name="Transform V1"/>
        </when>
        <when expression="#[vars.apiVersion == 'v2']">
            <flow-ref name="transform-users-v2" doc:name="Transform V2"/>
        </when>
    </choice>
</flow>
```

### Accept Header Versioning (Content Negotiation)

```xml
<flow name="get-users-accept-header" doc:name="Get Users (Accept Header)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="/api/users"
                   allowedMethods="GET"
                   doc:name="GET /api/users"/>

    <!-- Extract version from Accept header -->
    <ee:transform doc:name="Parse Accept Header">
        <ee:variables>
            <ee:set-variable variableName="apiVersion"><![CDATA[%dw 2.0
output application/java
var accept = attributes.headers.accept default "application/vnd.myapi.v1+json"
---
if (accept contains "v2")
    "v2"
else
    "v1"]]></ee:set-variable>
        </ee:variables>
    </ee:transform>

    <!-- Core logic and version routing -->
    <flow-ref name="get-users-core-logic" doc:name="Core Logic"/>

    <choice doc:name="Route by Version">
        <when expression="#[vars.apiVersion == 'v1']">
            <flow-ref name="transform-users-v1" doc:name="Transform V1"/>
        </when>
        <when expression="#[vars.apiVersion == 'v2']">
            <flow-ref name="transform-users-v2" doc:name="Transform V2"/>
        </when>
    </choice>
</flow>
```

## Version-Specific Request/Response Examples

### V1 API

**Request**:
```bash
GET /api/v1/users/123
```

**Response**:
```json
{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "status": "ACTIVE"
}
```

### V2 API (Breaking Changes)

**Request**:
```bash
GET /api/v2/users/123
```

**Response**:
```json
{
  "id": "123",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "status": "ACTIVE",
  "createdAt": "2024-01-15T10:00:00Z",
  "metadata": {
    "lastLogin": "2024-03-20T14:30:00Z",
    "loginCount": 45
  }
}
```

**Breaking Changes from V1 to V2**:
- Split `name` into `firstName` and `lastName`
- Added `createdAt` field
- Added `metadata` nested object
- Changed date format to ISO-8601

## HTTP Status Codes

- **200 OK**: Successful request on valid version
- **400 Bad Request**: Invalid version specified
- **404 Not Found**: Version not supported or deprecated
- **410 Gone**: Version deprecated and removed
- **500 Internal Server Error**: Unexpected server error

## Version Deprecation Strategy

### 1. Announce Deprecation

Add deprecation headers and documentation:

```xml
<flow name="get-users-v1-deprecated" doc:name="Get Users V1 (Deprecated)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="/api/v1/users"
                   allowedMethods="GET"/>

    <!-- Add deprecation headers -->
    <ee:transform doc:name="Add Deprecation Headers">
        <ee:variables>
            <ee:set-variable variableName="outboundHeaders"><![CDATA[%dw 2.0
output application/java
---
{
    "Deprecation": "true",
    "Sunset": "2024-12-31T23:59:59Z",
    "Link": "</api/v2/users>; rel=\"successor-version\""
}]]></ee:set-variable>
        </ee:variables>
    </ee:transform>

    <!-- Include deprecation notice in response -->
    <flow-ref name="get-users-v1-logic"/>

    <ee:transform doc:name="Add Deprecation Notice">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
payload ++ {
    _deprecation: {
        deprecated: true,
        sunset: "2024-12-31T23:59:59Z",
        message: "This API version is deprecated. Please migrate to v2",
        migrationGuide: "https://docs.example.com/api/migration-v1-to-v2"
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <foreach collection="#[vars.outboundHeaders pluck $$]" doc:name="Set Headers">
        <set-variable variableName="headerName" value="#[payload]"/>
        <set-variable variableName="headerValue" value="#[vars.outboundHeaders[vars.headerName]]"/>
        <set-property propertyName="#[vars.headerName]" value="#[vars.headerValue]"/>
    </foreach>
</flow>
```

### 2. Grace Period

Maintain deprecated version for a defined period (e.g., 6-12 months).

### 3. Monitor Usage

Log and monitor deprecated version usage:

```xml
<logger level="WARN"
        message="Deprecated API v1 used by client: #[attributes.headers.'user-agent']"
        doc:name="Log Deprecation Usage"/>
```

### 4. Remove Version

After grace period, return 410 Gone:

```xml
<flow name="get-users-v1-removed" doc:name="Get Users V1 (Removed)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="/api/v1/users"
                   allowedMethods="GET"/>

    <ee:transform doc:name="Transform to 410 Gone">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "VERSION_REMOVED",
    message: "API v1 has been removed. Please use v2",
    sunset: "2024-12-31T23:59:59Z",
    migrationGuide: "https://docs.example.com/api/migration-v1-to-v2"
}]]></ee:set-payload>
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="httpStatus">410</ee:set-variable>
        </ee:variables>
    </ee:transform>
</flow>
```

## Version Management Best Practices

### 1. Semantic Versioning for APIs

- **Major version** (v1, v2, v3): Breaking changes
- Keep it simple: Just use major versions in API URLs
- Minor/patch versions are internal implementation details

### 2. Default Version

Always provide a default version:

```xml
<set-variable variableName="apiVersion"
              value="#[attributes.headers['x-api-version'] default 'v2']"/>
```

### 3. Shared Business Logic

Extract common logic to sub-flows:

```xml
<sub-flow name="core-business-logic">
    <!-- Database queries, validations, business rules -->
</sub-flow>

<sub-flow name="transform-v1">
    <!-- V1-specific response transformation -->
</sub-flow>

<sub-flow name="transform-v2">
    <!-- V2-specific response transformation -->
</sub-flow>
```

### 4. Version in Response

Include version in response metadata:

```json
{
  "version": "v2",
  "data": { ... }
}
```

### 5. Documentation per Version

Maintain separate API documentation for each version:
- `/docs/v1/` - V1 documentation
- `/docs/v2/` - V2 documentation

## Complete Flow Pattern

```
1. HTTP Listener receives request
   ↓
2. Extract version (URI, header, or query param)
   ↓
3. Validate version is supported
   ↓
4. Execute shared core business logic
   ↓
5. Route to version-specific transformation
   ↓
6. Apply version-specific response format
   ↓
7. Add version metadata to response
   ↓
8. Return response with correct status code
```

## Testing Checklist

- [ ] V1 endpoint returns V1 response format
- [ ] V2 endpoint returns V2 response format
- [ ] Invalid version returns 400 Bad Request
- [ ] Default version works when not specified
- [ ] Deprecated version includes deprecation headers
- [ ] Removed version returns 410 Gone
- [ ] Shared business logic works for all versions
- [ ] Version-specific validations work correctly
- [ ] Migration guide is accessible
- [ ] Monitoring logs version usage

## Common Pitfalls

❌ **Don't**: Version too frequently (every small change)

✅ **Do**: Version only for breaking changes

❌ **Don't**: Support unlimited old versions forever

✅ **Do**: Deprecate and remove old versions with grace period

❌ **Don't**: Duplicate all business logic per version

✅ **Do**: Share core logic, version only transformations

❌ **Don't**: Change version behavior after release

✅ **Do**: Keep versions immutable after release

❌ **Don't**: Forget to document breaking changes

✅ **Do**: Maintain clear migration guides

## Performance Considerations

- **Shared Logic**: Reuse database queries and business logic across versions
- **Caching**: Cache version-specific transformations if possible
- **Routing Overhead**: Minimal overhead for version routing
- **Monitoring**: Track usage per version to identify migration progress

## Security Considerations

1. **Version Validation**: Validate version input to prevent injection
2. **Authorization**: Apply same security to all versions
3. **Audit Logging**: Log which version clients use
4. **Deprecation Communication**: Notify clients of security-critical upgrades

## Advanced Features

### Version Negotiation

```xml
<!-- Support multiple version formats -->
<ee:transform doc:name="Detect Version">
    <ee:variables>
        <ee:set-variable variableName="apiVersion"><![CDATA[%dw 2.0
output application/java
---
// Priority: URI > Header > Query > Default
attributes.uriParams.version default
attributes.headers.'x-api-version' default
attributes.queryParams.version default
"v2"]]></ee:set-variable>
    </ee:variables>
</ee:transform>
```

### Per-Endpoint Versioning

Different endpoints can have different version lifecycles:

```
/api/v1/users      (deprecated)
/api/v2/users      (current)
/api/v1/orders     (still current)
```

## Dependencies Required

```xml
<!-- Validation Module -->
<dependency>
    <groupId>org.mule.modules</groupId>
    <artifactId>mule-validation-module</artifactId>
    <version>2.0.1</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

## Related Patterns

- **Deprecation**: Strategy for retiring old versions
- **Backward Compatibility**: Techniques for avoiding versioning
- **Feature Flags**: Alternative to versioning for gradual rollouts
- **API Gateway**: Centralized version routing

## References

- [API Design Patterns by JJ Geewax](https://www.manning.com/books/api-design-patterns) - Chapter 15
- [Semantic Versioning](https://semver.org/)
- [RFC 8594 - Sunset HTTP Header](https://tools.ietf.org/html/rfc8594)
- [Microsoft API Versioning](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design#versioning-a-restful-web-api)
- [Google API Versioning](https://cloud.google.com/apis/design/versioning)

---

**Implementation Template**: Use `assets/mule-base-template/` as starting point
**Advanced Features**: See `references/advanced/` for version negotiation strategies
**Troubleshooting**: See `references/troubleshooting/` for common versioning issues
