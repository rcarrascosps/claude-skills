# Pagination Pattern - MuleSoft Implementation Reference

## Pattern Overview

The **Pagination** pattern divides large result sets into manageable pages, improving performance, user experience, and reducing bandwidth usage. This pattern allows clients to fetch data in chunks rather than receiving all results at once.

**Key Concept**: Break large datasets into smaller, sequential pages that clients can request individually.

## When to Use This Pattern

✅ **Use Pagination when**:
- Returning collections with many items (>100 items)
- List operations could return large datasets
- User interfaces need to display results incrementally
- Mobile applications with limited bandwidth
- Database queries could be expensive
- Real-time data loading is needed
- Search results need to be displayed

❌ **Don't use when**:
- Collections are guaranteed to be small (<20 items)
- Full dataset is required for processing
- Export/download operations (use streaming instead)
- Real-time analytics requiring complete data

## Pagination Strategies

### 1. Offset-Based Pagination (Page Number)

**Best for**: Small to medium datasets, user-facing interfaces with page numbers

```
GET /api/v1/resources?page=2&pageSize=20
```

**Pros**:
- Simple to implement
- Users can jump to specific pages
- Familiar pattern

**Cons**:
- Performance degrades with large offsets
- Issues with concurrent data changes
- Not suitable for real-time data

### 2. Cursor-Based Pagination (Recommended)

**Best for**: Large datasets, real-time data, mobile apps, infinite scrolling

```
GET /api/v1/resources?cursor=eyJpZCI6MTIzfQ&pageSize=20
```

**Pros**:
- Consistent performance regardless of position
- Handles concurrent changes gracefully
- Efficient for large datasets
- No duplicate/missing items during pagination

**Cons**:
- Can't jump to arbitrary pages
- More complex implementation

### 3. Keyset-Based Pagination

**Best for**: Time-series data, chronological ordering

```
GET /api/v1/resources?since=2024-01-01T00:00:00Z&limit=20
```

**Pros**:
- Very efficient for time-series data
- Natural for chronological data
- Simple for clients

**Cons**:
- Limited to ordered fields (timestamps, IDs)
- Backward pagination is complex

## Required Endpoints

### Offset-Based Pagination Endpoint

```
GET /api/v1/{resources}?page={pageNumber}&pageSize={itemsPerPage}
```

**Query Parameters**:
- `page` (integer, default: 1): Page number to retrieve
- `pageSize` (integer, default: 20, max: 100): Items per page

**Example Request**:
```
GET /api/v1/users?page=2&pageSize=20
```

### Cursor-Based Pagination Endpoint

```
GET /api/v1/{resources}?cursor={opaqueCursor}&pageSize={itemsPerPage}
```

**Query Parameters**:
- `cursor` (string, optional): Opaque cursor for next page
- `pageSize` (integer, default: 20, max: 100): Items per page

**Example Request**:
```
GET /api/v1/users?cursor=eyJpZCI6NDAsImNyZWF0ZWRBdCI6IjIwMjQtMDEtMTVUMTA6MDA6MDBaIn0&pageSize=20
```

## Response Format

### Offset-Based Response

```json
{
  "data": [
    { "id": "21", "name": "User 21" },
    { "id": "22", "name": "User 22" }
  ],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": true
  },
  "links": {
    "self": "/api/v1/users?page=2&pageSize=20",
    "first": "/api/v1/users?page=1&pageSize=20",
    "prev": "/api/v1/users?page=1&pageSize=20",
    "next": "/api/v1/users?page=3&pageSize=20",
    "last": "/api/v1/users?page=8&pageSize=20"
  }
}
```

### Cursor-Based Response

```json
{
  "data": [
    { "id": "21", "name": "User 21", "createdAt": "2024-01-15T10:00:00Z" },
    { "id": "22", "name": "User 22", "createdAt": "2024-01-15T10:05:00Z" }
  ],
  "pagination": {
    "pageSize": 20,
    "hasNext": true,
    "nextCursor": "eyJpZCI6NDAsImNyZWF0ZWRBdCI6IjIwMjQtMDEtMTVUMTA6MzA6MDBaIn0"
  },
  "links": {
    "self": "/api/v1/users?cursor=eyJpZCI6MjAsImNyZWF0ZWRBdCI6IjIwMjQtMDEtMTVUMDk6MzA6MDBaIn0&pageSize=20",
    "next": "/api/v1/users?cursor=eyJpZCI6NDAsImNyZWF0ZWRBdCI6IjIwMjQtMDEtMTVUMTA6MzA6MDBaIn0&pageSize=20"
  }
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

### Offset-Based Pagination Flow

```xml
<flow name="get-resources-offset-pagination" doc:name="Get Resources (Offset)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources"
                   allowedMethods="GET"
                   doc:name="GET /api/v1/resources"/>

    <!-- Extract and validate query parameters -->
    <set-variable variableName="page"
                  value="#[attributes.queryParams.page default 1]"
                  doc:name="Set Page"/>
    <set-variable variableName="pageSize"
                  value="#[attributes.queryParams.pageSize default 20]"
                  doc:name="Set Page Size"/>

    <!-- Validate pageSize limits -->
    <choice doc:name="Validate Page Size">
        <when expression="#[vars.pageSize &gt; 100]">
            <set-variable variableName="pageSize" value="#[100]"/>
        </when>
        <when expression="#[vars.pageSize &lt; 1]">
            <set-variable variableName="pageSize" value="#[20]"/>
        </when>
    </choice>

    <!-- Calculate offset -->
    <set-variable variableName="offset"
                  value="#[(vars.page - 1) * vars.pageSize]"
                  doc:name="Calculate Offset"/>

    <!-- Get total count -->
    <flow-ref name="get-total-count-subflow" doc:name="Get Total Count"/>
    <set-variable variableName="totalItems" value="#[payload[0].total]"/>

    <!-- Get paginated data -->
    <db:select config-ref="Database_Config" doc:name="Get Paginated Resources">
        <db:sql><![CDATA[SELECT id, name, email, createdAt, updatedAt
FROM resources
ORDER BY createdAt DESC
LIMIT :limit OFFSET :offset]]></db:sql>
        <db:input-parameters><![CDATA[#[{
            limit: vars.pageSize,
            offset: vars.offset
        }]]]></db:input-parameters>
    </db:select>

    <!-- Transform to response format -->
    <ee:transform doc:name="Build Paginated Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
var totalPages = ceil(vars.totalItems / vars.pageSize)
var hasNext = vars.page < totalPages
var hasPrevious = vars.page > 1
var baseUrl = "${api.base.path}/resources"
---
{
    data: payload,
    pagination: {
        page: vars.page,
        pageSize: vars.pageSize,
        totalItems: vars.totalItems,
        totalPages: totalPages,
        hasNext: hasNext,
        hasPrevious: hasPrevious
    },
    links: {
        self: baseUrl ++ "?page=" ++ vars.page ++ "&pageSize=" ++ vars.pageSize,
        first: baseUrl ++ "?page=1&pageSize=" ++ vars.pageSize,
        (prev: baseUrl ++ "?page=" ++ (vars.page - 1) ++ "&pageSize=" ++ vars.pageSize) if hasPrevious,
        (next: baseUrl ++ "?page=" ++ (vars.page + 1) ++ "&pageSize=" ++ vars.pageSize) if hasNext,
        last: baseUrl ++ "?page=" ++ totalPages ++ "&pageSize=" ++ vars.pageSize
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>

<sub-flow name="get-total-count-subflow" doc:name="Get Total Count">
    <db:select config-ref="Database_Config" doc:name="Count Total Resources">
        <db:sql><![CDATA[SELECT COUNT(*) as total FROM resources]]></db:sql>
    </db:select>
</sub-flow>
```

### Cursor-Based Pagination Flow

```xml
<flow name="get-resources-cursor-pagination" doc:name="Get Resources (Cursor)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources"
                   allowedMethods="GET"
                   doc:name="GET /api/v1/resources"/>

    <!-- Extract query parameters -->
    <set-variable variableName="cursor"
                  value="#[attributes.queryParams.cursor]"
                  doc:name="Set Cursor"/>
    <set-variable variableName="pageSize"
                  value="#[attributes.queryParams.pageSize default 20]"
                  doc:name="Set Page Size"/>

    <!-- Decode cursor if present -->
    <choice doc:name="Has Cursor?">
        <when expression="#[vars.cursor != null]">
            <ee:transform doc:name="Decode Cursor">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
import * from dw::core::Binaries
output application/json
---
fromBase64(vars.cursor) as String]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="decodedCursor" value="#[payload]"/>
        </when>
        <otherwise>
            <set-variable variableName="decodedCursor" value="#[null]"/>
        </otherwise>
    </choice>

    <!-- Get paginated data with cursor -->
    <choice doc:name="First Page or Next Page?">
        <when expression="#[vars.decodedCursor == null]">
            <!-- First page -->
            <db:select config-ref="Database_Config" doc:name="Get First Page">
                <db:sql><![CDATA[SELECT id, name, email, createdAt, updatedAt
FROM resources
ORDER BY createdAt DESC, id DESC
LIMIT :limit]]></db:sql>
                <db:input-parameters><![CDATA[#[{
                    limit: vars.pageSize + 1
                }]]]></db:input-parameters>
            </db:select>
        </when>
        <otherwise>
            <!-- Next page using cursor -->
            <db:select config-ref="Database_Config" doc:name="Get Next Page">
                <db:sql><![CDATA[SELECT id, name, email, createdAt, updatedAt
FROM resources
WHERE (createdAt < :cursorDate)
   OR (createdAt = :cursorDate AND id < :cursorId)
ORDER BY createdAt DESC, id DESC
LIMIT :limit]]></db:sql>
                <db:input-parameters><![CDATA[#[{
                    cursorDate: vars.decodedCursor.createdAt,
                    cursorId: vars.decodedCursor.id,
                    limit: vars.pageSize + 1
                }]]]></db:input-parameters>
            </db:select>
        </otherwise>
    </choice>

    <!-- Transform to response format -->
    <ee:transform doc:name="Build Cursor Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
import * from dw::core::Binaries
output application/json
var hasNext = sizeOf(payload) > vars.pageSize
var items = if (hasNext) payload[0 to (vars.pageSize - 1)] else payload
var lastItem = items[-1]
var nextCursor = if (hasNext)
    toBase64({
        id: lastItem.id,
        createdAt: lastItem.createdAt
    } as String)
    else null
var baseUrl = "${api.base.path}/resources"
var currentCursor = vars.cursor default ""
---
{
    data: items,
    pagination: {
        pageSize: vars.pageSize,
        hasNext: hasNext,
        (nextCursor: nextCursor) if hasNext
    },
    links: {
        self: baseUrl ++ "?cursor=" ++ currentCursor ++ "&pageSize=" ++ vars.pageSize,
        (next: baseUrl ++ "?cursor=" ++ nextCursor ++ "&pageSize=" ++ vars.pageSize) if hasNext
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

## HTTP Status Codes

- **200 OK**: Successfully retrieved paginated results
- **400 Bad Request**: Invalid pagination parameters
- **500 Internal Server Error**: Unexpected server error

## Validation Requirements

### Query Parameter Validation

```xml
<validation:is-true expression="#[vars.page &gt;= 1]"
                    message="Page number must be >= 1"/>

<validation:is-true expression="#[vars.pageSize &gt;= 1 and vars.pageSize &lt;= 100]"
                    message="Page size must be between 1 and 100"/>
```

### Cursor Validation

```dataweave
%dw 2.0
output application/json
import * from dw::core::Binaries
---
try {
    fromBase64(vars.cursor) as String
} catch (error) {
    error("Invalid cursor format")
}
```

## Complete Flow Pattern

### Offset-Based Flow
```
1. HTTP Listener receives GET request
   ↓
2. Extract page and pageSize query parameters
   ↓
3. Validate and enforce limits (max pageSize)
   ↓
4. Calculate offset: (page - 1) * pageSize
   ↓
5. Query total count (for metadata)
   ↓
6. Query paginated data using LIMIT/OFFSET
   ↓
7. Calculate metadata (totalPages, hasNext, etc.)
   ↓
8. Build response with data, pagination, and links
   ↓
9. Return 200 OK with paginated response
```

### Cursor-Based Flow
```
1. HTTP Listener receives GET request
   ↓
2. Extract cursor and pageSize query parameters
   ↓
3. Decode cursor (Base64) if present
   ↓
4. Query data: First page OR Next page using cursor
   ↓
5. Fetch pageSize + 1 items (to detect hasNext)
   ↓
6. Build next cursor from last item
   ↓
7. Build response with data, pagination, and links
   ↓
8. Return 200 OK with paginated response
```

## Cursor Encoding/Decoding

### Encoding (DataWeave)

```dataweave
%dw 2.0
import * from dw::core::Binaries
output application/json
var cursorData = {
    id: payload[-1].id,
    createdAt: payload[-1].createdAt
}
---
toBase64(write(cursorData, "application/json"))
```

### Decoding (DataWeave)

```dataweave
%dw 2.0
import * from dw::core::Binaries
output application/json
---
read(fromBase64(vars.cursor), "application/json")
```

## Example Use Cases

### 1. User List with Page Numbers

```bash
GET /api/v1/users?page=1&pageSize=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8
  }
}
```

### 2. Infinite Scrolling (Cursor-Based)

```bash
# First request
GET /api/v1/posts?pageSize=20

# Next request using cursor from previous response
GET /api/v1/posts?cursor=eyJpZCI6NDAsImNyZWF0ZWRBdCI6IjIwMjQtMDEtMTVUMTA6MzA6MDBaIn0&pageSize=20
```

### 3. Time-Based Pagination

```bash
GET /api/v1/events?since=2024-01-01T00:00:00Z&limit=50
```

## Testing Checklist

- [ ] First page retrieval (no cursor/page param)
- [ ] Next page retrieval (with cursor/page param)
- [ ] Last page detection (hasNext = false)
- [ ] Empty result set handling
- [ ] Invalid page number handling (page < 1)
- [ ] Invalid pageSize handling (pageSize > max)
- [ ] Invalid cursor handling (malformed Base64)
- [ ] Page size limit enforcement (max 100)
- [ ] Total count accuracy (offset-based)
- [ ] No duplicate items across pages
- [ ] No missing items across pages
- [ ] Concurrent data changes (cursor-based resilience)

## Common Pitfalls

❌ **Don't**: Use offset pagination for large datasets
```sql
-- WRONG for large datasets (slow with high offsets)
SELECT * FROM resources LIMIT 20 OFFSET 10000
```

✅ **Do**: Use cursor-based pagination for large datasets
```sql
-- CORRECT for large datasets
SELECT * FROM resources
WHERE createdAt < :cursorDate
ORDER BY createdAt DESC
LIMIT 20
```

❌ **Don't**: Forget to limit maximum page size

✅ **Do**: Enforce maximum page size (e.g., 100)

❌ **Don't**: Return total count for cursor-based pagination (expensive)

✅ **Do**: Only include hasNext indicator

❌ **Don't**: Use simple IDs as cursors (predictable, insecure)

✅ **Do**: Use opaque, Base64-encoded cursors

❌ **Don't**: Query exact total count on every request (expensive)

✅ **Do**: Use approximate counts or cache total count

## Performance Considerations

### Offset-Based Optimization
- **Database Indexes**: Ensure ORDER BY columns are indexed
- **Count Query Caching**: Cache total count for frequently accessed lists
- **Materialized Views**: Consider for complex queries
- **Avoid COUNT(*)**: Use approximate counts for large tables

### Cursor-Based Optimization
- **Composite Indexes**: Create indexes on (timestamp, id) for cursor queries
- **Avoid SELECT ***: Only fetch required columns
- **Connection Pooling**: Configure database connection pool
- **pageSize + 1 Trick**: Fetch one extra item to detect hasNext efficiently

### Database Index Examples

```sql
-- For offset-based pagination
CREATE INDEX idx_resources_created_at ON resources(createdAt DESC);

-- For cursor-based pagination (composite index)
CREATE INDEX idx_resources_created_id ON resources(createdAt DESC, id DESC);

-- For keyset pagination (timestamp-based)
CREATE INDEX idx_events_timestamp ON events(timestamp DESC);
```

## Security Considerations

1. **Cursor Tampering**: Validate decoded cursor data
2. **Page Bombing**: Enforce maximum page size limits
3. **Resource Exhaustion**: Implement rate limiting
4. **Data Leakage**: Don't expose internal IDs in cursors (use UUIDs)
5. **Authorization**: Verify user permissions for accessed data

## Advanced Features

### Bi-directional Cursor Pagination

Support both next and previous page navigation:

```
GET /api/v1/resources?cursor={cursor}&direction=next&pageSize=20
GET /api/v1/resources?cursor={cursor}&direction=prev&pageSize=20
```

### Filtering with Pagination

Combine with list filtering pattern:

```
GET /api/v1/resources?status=ACTIVE&page=1&pageSize=20
GET /api/v1/resources?createdAfter=2024-01-01&cursor={cursor}&pageSize=20
```

### Stable Pagination

Ensure consistent results during pagination despite concurrent changes:
- Use cursor-based pagination
- Include version/timestamp in cursor
- Snapshot isolation for database queries

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

## Related Patterns

- **List Filtering**: Combine with pagination for filtered results
- **Sorting**: Allow clients to specify sort order
- **Field Selection**: Let clients choose which fields to return
- **Resource Expansion**: Expand related resources in paginated lists

## References

- [API Design Patterns by JJ Geewax](https://www.manning.com/books/api-design-patterns) - Chapter 8
- [Google API Design Guide - List Pagination](https://cloud.google.com/apis/design/design_patterns#list_pagination)
- [Slack API Pagination](https://api.slack.com/docs/pagination)
- [Stripe API Cursor-Based Pagination](https://stripe.com/docs/api/pagination)

---

**Implementation Template**: Use `assets/mule-base-template/` as starting point
**Advanced Features**: See `references/advanced/` for filtering and sorting
**Troubleshooting**: See `references/troubleshooting/` for common issues
