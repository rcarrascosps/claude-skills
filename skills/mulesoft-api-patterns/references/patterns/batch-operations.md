# Batch Operations Pattern - MuleSoft Implementation Reference

## Pattern Overview

The **Batch Operations** pattern allows clients to create, update, or delete multiple resources in a single API request. This reduces network overhead, improves performance, and can provide transactional guarantees across multiple operations.

**Key Concept**: Accept an array of operations in one request and process them efficiently, returning results for each operation.

## When to Use This Pattern

✅ **Use Batch Operations when**:
- Clients need to create/update/delete multiple resources at once
- Reducing network round-trips improves performance
- Bulk data imports or exports are required
- Mobile applications with intermittent connectivity
- Synchronizing data between systems
- Initial data loading or seeding
- Mass updates to multiple resources
- Transactional consistency across operations is desired

❌ **Don't use when**:
- Only single resources are being modified
- Real-time processing of each item is critical
- Operations are unrelated and shouldn't be grouped
- Individual operation tracking is more important than batch efficiency

## Batch Operation Types

### 1. Homogeneous Batch (Same Operation Type)

All operations are the same type (all creates, all updates, or all deletes).

**Example - Batch Create**:
```
POST /api/v1/users:batchCreate
Content-Type: application/json

{
  "items": [
    {"email": "user1@example.com", "name": "User One"},
    {"email": "user2@example.com", "name": "User Two"}
  ]
}
```

### 2. Heterogeneous Batch (Mixed Operation Types)

Mix of creates, updates, and deletes in a single batch.

**Example - Mixed Batch**:
```
POST /api/v1/users:batch
Content-Type: application/json

{
  "operations": [
    {"operation": "create", "data": {"email": "new@example.com", "name": "New User"}},
    {"operation": "update", "id": "123", "data": {"name": "Updated Name"}},
    {"operation": "delete", "id": "456"}
  ]
}
```

### 3. Bulk Update with Filter

Update multiple resources matching criteria.

**Example - Bulk Update**:
```
POST /api/v1/users:bulkUpdate
Content-Type: application/json

{
  "filter": {"status": "INACTIVE"},
  "updates": {"status": "ARCHIVED"}
}
```

## Required Endpoints

### 1. Batch Create (Recommended Approach)

```
POST /api/v1/{resources}:batchCreate
Content-Type: application/json

{
  "items": [
    { /* resource 1 */ },
    { /* resource 2 */ }
  ]
}
```

**Response (200 OK or 207 Multi-Status)**:
```json
{
  "results": [
    {
      "index": 0,
      "status": "success",
      "resource": {
        "id": "user_001",
        "email": "user1@example.com",
        "name": "User One"
      }
    },
    {
      "index": 1,
      "status": "error",
      "error": {
        "code": "DUPLICATE_EMAIL",
        "message": "Email already exists"
      }
    }
  ],
  "summary": {
    "total": 2,
    "successful": 1,
    "failed": 1
  }
}
```

### 2. Batch Update

```
POST /api/v1/{resources}:batchUpdate
Content-Type: application/json

{
  "items": [
    {"id": "123", "name": "Updated Name 1"},
    {"id": "456", "name": "Updated Name 2"}
  ]
}
```

### 3. Batch Delete

```
POST /api/v1/{resources}:batchDelete
Content-Type: application/json

{
  "ids": ["123", "456", "789"]
}
```

### 4. Bulk Update with Filter

```
POST /api/v1/{resources}:bulkUpdate
Content-Type: application/json

{
  "filter": {
    "status": "PENDING",
    "createdBefore": "2024-01-01T00:00:00Z"
  },
  "updates": {
    "status": "EXPIRED"
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

### Batch Create Flow

```xml
<flow name="batch-create-resources" doc:name="Batch Create Resources">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources:batchCreate"
                   allowedMethods="POST"
                   doc:name="POST /api/v1/resources:batchCreate"/>

    <!-- Validate request -->
    <set-variable variableName="items" value="#[payload.items]" doc:name="Set Items"/>

    <validation:is-not-empty-collection value="#[vars.items]"
                                        message="Items array cannot be empty"
                                        doc:name="Validate Items Not Empty"/>

    <!-- Validate batch size limit -->
    <choice doc:name="Check Batch Size">
        <when expression="#[sizeOf(vars.items) &gt; 100]">
            <ee:transform doc:name="Transform to 400">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "BATCH_TOO_LARGE",
    message: "Batch size exceeds maximum of 100 items",
    timestamp: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="httpStatus">400</ee:set-variable>
                </ee:variables>
            </ee:transform>
        </when>
        <otherwise>
            <!-- Process batch -->
            <flow-ref name="process-batch-create-subflow" doc:name="Process Batch Create"/>
        </otherwise>
    </choice>
</flow>

<sub-flow name="process-batch-create-subflow" doc:name="Process Batch Create">
    <!-- Initialize results array -->
    <set-variable variableName="results" value="#[[]]" doc:name="Initialize Results"/>
    <set-variable variableName="successCount" value="#[0]" doc:name="Initialize Success Count"/>
    <set-variable variableName="failureCount" value="#[0]" doc:name="Initialize Failure Count"/>

    <!-- Process each item -->
    <foreach collection="#[vars.items]" doc:name="For Each Item">
        <set-variable variableName="currentIndex" value="#[payload.index]" doc:name="Set Index"/>
        <set-variable variableName="currentItem" value="#[payload]" doc:name="Set Current Item"/>

        <try doc:name="Try Create">
            <!-- Validate individual item -->
            <flow-ref name="validate-resource-subflow" doc:name="Validate Resource"/>

            <!-- Generate ID for new resource -->
            <ee:transform doc:name="Generate Resource ID">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/java
import * from dw::core::Strings
---
vars.currentItem ++ {
    id: "user_" ++ uuid()[0 to 7],
    createdAt: now(),
    updatedAt: now(),
    status: vars.currentItem.status default "ACTIVE"
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>

            <!-- Insert into database -->
            <db:insert config-ref="Database_Config" doc:name="Insert Resource">
                <db:sql><![CDATA[INSERT INTO resources (
    id, email, name, status, created_at, updated_at
) VALUES (
    :id, :email, :name, :status, :createdAt, :updatedAt
)]]></db:sql>
                <db:input-parameters><![CDATA[#[payload]]]></db:input-parameters>
            </db:insert>

            <!-- Add success result -->
            <ee:transform doc:name="Add Success Result">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
payload]]></ee:set-payload>
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="results"><![CDATA[%dw 2.0
output application/java
---
vars.results << {
    index: vars.currentIndex,
    status: "success",
    resource: payload
}]]></ee:set-variable>
                    <ee:set-variable variableName="successCount"><![CDATA[%dw 2.0
output application/java
---
vars.successCount + 1]]></ee:set-variable>
                </ee:variables>
            </ee:transform>

            <error-handler>
                <on-error-continue type="ANY">
                    <!-- Add error result -->
                    <ee:transform doc:name="Add Error Result">
                        <ee:message>
                            <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
payload]]></ee:set-payload>
                        </ee:message>
                        <ee:variables>
                            <ee:set-variable variableName="results"><![CDATA[%dw 2.0
output application/java
---
vars.results << {
    index: vars.currentIndex,
    status: "error",
    error: {
        code: error.errorType.identifier,
        message: error.description
    }
}]]></ee:set-variable>
                            <ee:set-variable variableName="failureCount"><![CDATA[%dw 2.0
output application/java
---
vars.failureCount + 1]]></ee:set-variable>
                        </ee:variables>
                    </ee:transform>
                </on-error-continue>
            </error-handler>
        </try>
    </foreach>

    <!-- Build final response -->
    <ee:transform doc:name="Build Batch Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    results: vars.results,
    summary: {
        total: sizeOf(vars.items),
        successful: vars.successCount,
        failed: vars.failureCount
    }
}]]></ee:set-payload>
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="httpStatus"><![CDATA[%dw 2.0
output application/java
---
if (vars.failureCount > 0) 207 else 200]]></ee:set-variable>
        </ee:variables>
    </ee:transform>
</sub-flow>
```

### Batch Create with db:bulk-insert (Optimized)

```xml
<flow name="batch-create-resources-optimized" doc:name="Batch Create (Optimized)">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources:batchCreate"
                   allowedMethods="POST"
                   doc:name="POST /api/v1/resources:batchCreate"/>

    <set-variable variableName="items" value="#[payload.items]"/>

    <!-- Validate all items first -->
    <flow-ref name="validate-all-items-subflow" doc:name="Validate All Items"/>

    <!-- Prepare data for bulk insert -->
    <ee:transform doc:name="Prepare Bulk Insert Data">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java
import * from dw::core::Strings
---
vars.items map (item, index) -> {
    id: "user_" ++ uuid()[0 to 7],
    email: item.email,
    name: item.name,
    status: item.status default "ACTIVE",
    createdAt: now(),
    updatedAt: now()
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <!-- Store for response -->
    <set-variable variableName="createdResources" value="#[payload]" doc:name="Store Resources"/>

    <!-- Bulk insert into database -->
    <db:bulk-insert config-ref="Database_Config" doc:name="Bulk Insert Resources">
        <db:sql><![CDATA[INSERT INTO resources (
    id, email, name, status, created_at, updated_at
) VALUES (
    :id, :email, :name, :status, :createdAt, :updatedAt
)]]></db:sql>
    </db:bulk-insert>

    <!-- Build response -->
    <ee:transform doc:name="Build Success Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    results: vars.createdResources map (resource, index) -> {
        index: index,
        status: "success",
        resource: resource
    },
    summary: {
        total: sizeOf(vars.createdResources),
        successful: sizeOf(vars.createdResources),
        failed: 0
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <error-handler>
        <on-error-propagate type="ANY">
            <ee:transform doc:name="Transform to Error Response">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: error.errorType.identifier,
    message: error.description,
    timestamp: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="httpStatus">500</ee:set-variable>
                </ee:variables>
            </ee:transform>
        </on-error-propagate>
    </error-handler>
</flow>
```

### Batch Update Flow

```xml
<flow name="batch-update-resources" doc:name="Batch Update Resources">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources:batchUpdate"
                   allowedMethods="POST"
                   doc:name="POST /api/v1/resources:batchUpdate"/>

    <set-variable variableName="items" value="#[payload.items]"/>
    <set-variable variableName="results" value="#[[]]"/>
    <set-variable variableName="successCount" value="#[0]"/>
    <set-variable variableName="failureCount" value="#[0]"/>

    <!-- Process each update -->
    <foreach collection="#[vars.items]" doc:name="For Each Update">
        <try doc:name="Try Update">
            <set-variable variableName="resourceId" value="#[payload.id]"/>
            <set-variable variableName="updates" value="#[payload]"/>

            <!-- Check if resource exists -->
            <db:select config-ref="Database_Config" doc:name="Get Current Resource">
                <db:sql><![CDATA[SELECT * FROM resources WHERE id = :id]]></db:sql>
                <db:input-parameters><![CDATA[#[{id: vars.resourceId}]]]></db:input-parameters>
            </db:select>

            <choice doc:name="Resource Exists?">
                <when expression="#[sizeOf(payload) == 0]">
                    <raise-error type="RESOURCE:NOT_FOUND"
                                 description="#['Resource with ID ' ++ vars.resourceId ++ ' not found']"/>
                </when>
                <otherwise>
                    <!-- Merge and update -->
                    <ee:transform doc:name="Merge Updates">
                        <ee:message>
                            <ee:set-payload><![CDATA[%dw 2.0
output application/java
var current = payload[0]
var updates = vars.updates
---
{
    id: current.id,
    email: updates.email default current.email,
    name: updates.name default current.name,
    status: updates.status default current.status,
    updatedAt: now()
}]]></ee:set-payload>
                        </ee:message>
                    </ee:transform>

                    <!-- Update database -->
                    <db:update config-ref="Database_Config" doc:name="Update Resource">
                        <db:sql><![CDATA[UPDATE resources
SET email = :email,
    name = :name,
    status = :status,
    updated_at = :updatedAt
WHERE id = :id]]></db:sql>
                        <db:input-parameters><![CDATA[#[payload]]]></db:input-parameters>
                    </db:update>

                    <!-- Add success result -->
                    <ee:transform doc:name="Add Success Result">
                        <ee:variables>
                            <ee:set-variable variableName="results"><![CDATA[%dw 2.0
output application/java
---
vars.results << {
    id: vars.resourceId,
    status: "success",
    resource: payload
}]]></ee:set-variable>
                            <ee:set-variable variableName="successCount"><![CDATA[%dw 2.0
output application/java
---
vars.successCount + 1]]></ee:set-variable>
                        </ee:variables>
                    </ee:transform>
                </otherwise>
            </choice>

            <error-handler>
                <on-error-continue type="ANY">
                    <!-- Add error result -->
                    <ee:transform doc:name="Add Error Result">
                        <ee:variables>
                            <ee:set-variable variableName="results"><![CDATA[%dw 2.0
output application/java
---
vars.results << {
    id: vars.resourceId,
    status: "error",
    error: {
        code: error.errorType.identifier,
        message: error.description
    }
}]]></ee:set-variable>
                            <ee:set-variable variableName="failureCount"><![CDATA[%dw 2.0
output application/java
---
vars.failureCount + 1]]></ee:set-variable>
                        </ee:variables>
                    </ee:transform>
                </on-error-continue>
            </error-handler>
        </try>
    </foreach>

    <!-- Build response -->
    <ee:transform doc:name="Build Batch Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    results: vars.results,
    summary: {
        total: sizeOf(vars.items),
        successful: vars.successCount,
        failed: vars.failureCount
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

### Batch Delete Flow

```xml
<flow name="batch-delete-resources" doc:name="Batch Delete Resources">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources:batchDelete"
                   allowedMethods="POST"
                   doc:name="POST /api/v1/resources:batchDelete"/>

    <set-variable variableName="ids" value="#[payload.ids]"/>

    <!-- Delete resources -->
    <ee:transform doc:name="Prepare IN Clause">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
{
    ids: vars.ids joinBy ","
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <db:delete config-ref="Database_Config" doc:name="Delete Resources">
        <db:sql><![CDATA[DELETE FROM resources WHERE id IN (:ids)]]></db:sql>
        <db:input-parameters><![CDATA[#[payload]]]></db:input-parameters>
    </db:delete>

    <!-- Build response -->
    <ee:transform doc:name="Build Delete Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    deletedCount: payload.affectedRows,
    ids: vars.ids
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

## HTTP Status Codes

- **200 OK**: All operations succeeded
- **207 Multi-Status**: Partial success (some succeeded, some failed)
- **400 Bad Request**: Invalid batch request (empty, too large, malformed)
- **413 Payload Too Large**: Batch exceeds maximum size
- **500 Internal Server Error**: Unexpected server error

## Processing Strategies

### 1. All-or-Nothing (Transactional)

All operations succeed or all fail (rolled back).

**Use when**: Data consistency is critical
**Implementation**: Use database transactions, rollback on any failure

```xml
<try transactionalAction="ALWAYS_BEGIN" doc:name="Transaction">
    <db:bulk-insert config-ref="Database_Config">
        <!-- bulk insert -->
    </db:bulk-insert>

    <error-handler>
        <on-error-propagate type="ANY">
            <!-- Transaction automatically rolls back -->
        </on-error-propagate>
    </error-handler>
</try>
```

### 2. Best Effort (Partial Success)

Process all items, report individual successes/failures.

**Use when**: Independent operations, partial success is acceptable
**Implementation**: foreach with try/catch, collect results

### 3. Stop-on-First-Error

Process until first error, then stop.

**Use when**: Errors indicate fundamental issues
**Implementation**: foreach without error handling, propagate first error

## Complete Flow Pattern

```
1. HTTP Listener receives batch request
   ↓
2. Validate batch (not empty, size within limits)
   ↓
3. Choose processing strategy:
   ├─ Transactional: Use db:bulk-insert with transaction
   ├─ Best Effort: foreach with error handling
   └─ Stop-on-Error: foreach without error handling
   ↓
4. Process each item in batch
   ↓
5. Collect results (success/error for each item)
   ↓
6. Build summary (total, successful, failed)
   ↓
7. Return 200 (all success) or 207 (partial)
```

## Example Use Cases

### 1. Bulk User Import

```bash
POST /api/v1/users:batchCreate
{
  "items": [
    {"email": "alice@example.com", "name": "Alice"},
    {"email": "bob@example.com", "name": "Bob"},
    {"email": "charlie@example.com", "name": "Charlie"}
  ]
}

Response (200 OK):
{
  "results": [
    {"index": 0, "status": "success", "resource": {"id": "user_001", ...}},
    {"index": 1, "status": "success", "resource": {"id": "user_002", ...}},
    {"index": 2, "status": "error", "error": {"code": "DUPLICATE_EMAIL", ...}}
  ],
  "summary": {
    "total": 3,
    "successful": 2,
    "failed": 1
  }
}
```

### 2. Bulk Status Update

```bash
POST /api/v1/users:batchUpdate
{
  "items": [
    {"id": "user_001", "status": "INACTIVE"},
    {"id": "user_002", "status": "INACTIVE"}
  ]
}
```

### 3. Bulk Delete

```bash
POST /api/v1/users:batchDelete
{
  "ids": ["user_001", "user_002", "user_003"]
}

Response:
{
  "deletedCount": 3,
  "ids": ["user_001", "user_002", "user_003"]
}
```

## Testing Checklist

- [ ] Batch create with all valid items (200)
- [ ] Batch create with some invalid items (207)
- [ ] Empty batch request (400)
- [ ] Batch exceeding size limit (400 or 413)
- [ ] Batch update with non-existent IDs (207)
- [ ] Batch delete with valid IDs (200)
- [ ] Transactional batch rollback on error
- [ ] Performance test with maximum batch size
- [ ] Concurrent batch requests
- [ ] Duplicate items in single batch

## Common Pitfalls

❌ **Don't**: Process unlimited batch sizes

✅ **Do**: Enforce maximum batch size (e.g., 100-1000 items)

❌ **Don't**: Use foreach for large batches when bulk operations exist

✅ **Do**: Use db:bulk-insert for better performance

❌ **Don't**: Return 200 when some operations failed

✅ **Do**: Return 207 Multi-Status for partial success

❌ **Don't**: Fail entire batch on single validation error (unless transactional)

✅ **Do**: Report individual failures, continue processing

❌ **Don't**: Use variables inside db:bulk-insert

✅ **Do**: Transform to payload before bulk-insert

## Performance Considerations

- **Batch Size Limits**: Enforce 100-1000 items per batch
- **Database Bulk Operations**: Use db:bulk-insert instead of foreach + insert
- **Parallel Processing**: Use parallel-foreach for independent operations
- **Memory Management**: Stream large batches if possible
- **Transaction Overhead**: Only use transactions when necessary
- **Index Optimization**: Ensure database indexes for bulk queries

### Performance Comparison

```
Single inserts (foreach):    10 items  = ~500ms
Bulk insert (db:bulk-insert): 10 items  = ~50ms
Bulk insert:                  100 items = ~200ms
Bulk insert:                  1000 items = ~1500ms
```

## Security Considerations

1. **Batch Size Limits**: Prevent resource exhaustion
2. **Rate Limiting**: Limit batch operations per user/time
3. **Authorization**: Verify permissions for all items in batch
4. **Input Validation**: Validate each item in batch
5. **Audit Logging**: Log batch operations with user context
6. **Idempotency**: Support idempotency keys for batch operations

## Advanced Features

### Async Batch Processing

For very large batches, combine with Long Running Operations:

```
POST /api/v1/users:batchCreate?async=true

Response (202 Accepted):
{
  "operationId": "op_12345",
  "status": "PENDING"
}
```

### Batch with Validation

```xml
<foreach collection="#[vars.items]">
    <validation:all doc:name="Validate Item">
        <validation:is-not-empty-string value="#[payload.email]"/>
        <validation:is-email email="#[payload.email]"/>
        <validation:is-not-empty-string value="#[payload.name]"/>
    </validation:all>
</foreach>
```

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

- **Long Running Operations**: For async batch processing
- **Idempotency**: Prevent duplicate batch processing
- **Pagination**: Return large batch results in pages
- **Partial Updates**: Combine with batch for bulk partial updates

## References

- [API Design Patterns by JJ Geewax](https://www.manning.com/books/api-design-patterns) - Chapter 11
- [Google Cloud Batch Operations](https://cloud.google.com/apis/design/standard_methods#batch_operations)
- [JSON Batch (JSON-RPC)](https://www.jsonrpc.org/specification#batch)
- [Microsoft Graph Batch Requests](https://docs.microsoft.com/en-us/graph/json-batching)

---

**Implementation Template**: Use `assets/mule-base-template/` as starting point
**Advanced Features**: See `references/advanced/` for async batch processing
**Troubleshooting**: See `references/troubleshooting/bulk-operations.md`
