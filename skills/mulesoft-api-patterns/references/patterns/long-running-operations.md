# Long Running Operations Pattern - MuleSoft Implementation Reference

## Pattern Overview

The **Long Running Operations (LRO)** pattern enables APIs to handle operations that take significant time to complete (seconds to hours) without blocking the client. The pattern uses asynchronous processing with status polling or webhooks.

**Key Concept**: Accept the request immediately, process asynchronously, and provide a way for clients to track progress and retrieve results.

## When to Use This Pattern

✅ **Use Long Running Operations when**:
- Operation takes more than 2-3 seconds to complete
- Processing involves external API calls with unpredictable latency
- Batch processing operations (importing data, generating reports)
- File processing (video transcoding, image processing)
- Complex calculations or data transformations
- Database operations on large datasets
- Third-party service integrations with delays
- Email sending, PDF generation, or document processing

❌ **Don't use when**:
- Operations complete in under 2 seconds
- Real-time response is critical
- Simple CRUD operations
- Synchronous processing is acceptable

## Operation Lifecycle States

```
PENDING → RUNNING → COMPLETED
                  ↓
                FAILED
                  ↓
               CANCELLED
```

**State Definitions**:
- **PENDING**: Operation accepted, waiting to start
- **RUNNING**: Operation actively processing
- **COMPLETED**: Operation finished successfully
- **FAILED**: Operation encountered an error
- **CANCELLED**: Operation cancelled by user/system

## Required Endpoints

### 1. Create Long Running Operation

```
POST /api/v1/{resource}/{id}:{action}
Content-Type: application/json

{
  "param1": "value1",
  "param2": "value2"
}
```

**Response (202 Accepted)**:
```json
{
  "operationId": "op_1234567890",
  "status": "PENDING",
  "createdAt": "2024-01-15T10:00:00Z",
  "links": {
    "self": "/api/v1/operations/op_1234567890",
    "cancel": "/api/v1/operations/op_1234567890:cancel"
  }
}
```

### 2. Get Operation Status

```
GET /api/v1/operations/{operationId}
```

**Response (200 OK)**:
```json
{
  "operationId": "op_1234567890",
  "status": "RUNNING",
  "progress": 45,
  "createdAt": "2024-01-15T10:00:00Z",
  "startedAt": "2024-01-15T10:00:05Z",
  "estimatedCompletion": "2024-01-15T10:05:00Z",
  "links": {
    "self": "/api/v1/operations/op_1234567890",
    "cancel": "/api/v1/operations/op_1234567890:cancel"
  }
}
```

**Response when COMPLETED (200 OK)**:
```json
{
  "operationId": "op_1234567890",
  "status": "COMPLETED",
  "progress": 100,
  "result": {
    "processedItems": 1500,
    "downloadUrl": "/api/v1/reports/report_xyz.pdf"
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "startedAt": "2024-01-15T10:00:05Z",
  "completedAt": "2024-01-15T10:04:32Z",
  "links": {
    "self": "/api/v1/operations/op_1234567890"
  }
}
```

**Response when FAILED (200 OK)**:
```json
{
  "operationId": "op_1234567890",
  "status": "FAILED",
  "error": {
    "code": "PROCESSING_ERROR",
    "message": "Failed to process file: Invalid format",
    "details": "Expected CSV format, received JSON"
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "startedAt": "2024-01-15T10:00:05Z",
  "failedAt": "2024-01-15T10:01:15Z",
  "links": {
    "self": "/api/v1/operations/op_1234567890"
  }
}
```

### 3. Cancel Operation (Optional)

```
POST /api/v1/operations/{operationId}:cancel
```

**Response (200 OK)**:
```json
{
  "operationId": "op_1234567890",
  "status": "CANCELLED",
  "cancelledAt": "2024-01-15T10:02:00Z"
}
```

### 4. List Operations (Optional)

```
GET /api/v1/operations?status={status}&page={page}&pageSize={pageSize}
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

<!-- VM Queue for async processing -->
<vm:config name="VM_Config" doc:name="VM Config">
    <vm:queues>
        <vm:queue queueName="operationsQueue" queueType="PERSISTENT"/>
    </vm:queues>
</vm:config>
```

### Database Schema

```sql
CREATE TABLE operations (
    operation_id VARCHAR(50) PRIMARY KEY,
    resource_type VARCHAR(50) NOT NULL,
    resource_id VARCHAR(50),
    action VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    progress INT DEFAULT 0,
    input_data TEXT,
    result_data TEXT,
    error_code VARCHAR(50),
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    failed_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    estimated_completion TIMESTAMP
);

CREATE INDEX idx_operations_status ON operations(status);
CREATE INDEX idx_operations_created ON operations(created_at DESC);
```

### 1. Create Operation Flow (Synchronous Entry Point)

```xml
<flow name="create-long-running-operation" doc:name="Create LRO">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/{resource}/{id}:{action}"
                   allowedMethods="POST"
                   doc:name="POST /api/v1/{resource}/{id}:{action}"/>

    <!-- Extract variables -->
    <set-variable variableName="resourceType" value="#[attributes.uriParams.resource]"/>
    <set-variable variableName="resourceId" value="#[attributes.uriParams.id]"/>
    <set-variable variableName="action" value="#[attributes.uriParams.action]"/>
    <set-variable variableName="inputData" value="#[payload]"/>

    <!-- Generate unique operation ID -->
    <ee:transform doc:name="Generate Operation ID">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java
import * from dw::core::Strings
---
"op_" ++ uuid()]]></ee:set-payload>
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="operationId"><![CDATA[%dw 2.0
output application/java
import * from dw::core::Strings
---
"op_" ++ uuid()]]></ee:set-variable>
        </ee:variables>
    </ee:transform>

    <!-- Insert operation record (PENDING) -->
    <db:insert config-ref="Database_Config" doc:name="Create Operation Record">
        <db:sql><![CDATA[INSERT INTO operations (
    operation_id,
    resource_type,
    resource_id,
    action,
    status,
    input_data,
    created_at
) VALUES (
    :operationId,
    :resourceType,
    :resourceId,
    :action,
    'PENDING',
    :inputData,
    CURRENT_TIMESTAMP
)]]></db:sql>
        <db:input-parameters><![CDATA[#[{
            operationId: vars.operationId,
            resourceType: vars.resourceType,
            resourceId: vars.resourceId,
            action: vars.action,
            inputData: write(vars.inputData, "application/json")
        }]]]></db:input-parameters>
    </db:insert>

    <!-- Send to async processing queue -->
    <vm:publish queueName="operationsQueue" config-ref="VM_Config" doc:name="Queue for Processing">
        <vm:content><![CDATA[#[{
            operationId: vars.operationId,
            resourceType: vars.resourceType,
            resourceId: vars.resourceId,
            action: vars.action,
            inputData: vars.inputData
        }]]]></vm:content>
    </vm:publish>

    <!-- Return 202 Accepted with operation details -->
    <ee:transform doc:name="Build 202 Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
var baseUrl = "${api.base.path}"
---
{
    operationId: vars.operationId,
    status: "PENDING",
    createdAt: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"},
    links: {
        self: baseUrl ++ "/operations/" ++ vars.operationId,
        cancel: baseUrl ++ "/operations/" ++ vars.operationId ++ ":cancel"
    }
}]]></ee:set-payload>
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="httpStatus">202</ee:set-variable>
        </ee:variables>
    </ee:transform>

    <set-payload value="#[payload]" mimeType="application/json"/>
    <set-variable variableName="httpStatus" value="202"/>
</flow>
```

### 2. Async Processing Flow (Background Worker)

```xml
<flow name="process-long-running-operation" doc:name="Process LRO">
    <vm:listener queueName="operationsQueue"
                 config-ref="VM_Config"
                 doc:name="VM Queue Listener"/>

    <set-variable variableName="operationId" value="#[payload.operationId]"/>
    <set-variable variableName="action" value="#[payload.action]"/>
    <set-variable variableName="inputData" value="#[payload.inputData]"/>

    <!-- Update status to RUNNING -->
    <db:update config-ref="Database_Config" doc:name="Update to RUNNING">
        <db:sql><![CDATA[UPDATE operations
SET status = 'RUNNING',
    started_at = CURRENT_TIMESTAMP,
    estimated_completion = CURRENT_TIMESTAMP + INTERVAL '5 minutes'
WHERE operation_id = :operationId]]></db:sql>
        <db:input-parameters><![CDATA[#[{
            operationId: vars.operationId
        }]]]></db:input-parameters>
    </db:update>

    <!-- Process based on action type -->
    <try doc:name="Try Processing">
        <choice doc:name="Route by Action">
            <when expression="#[vars.action == 'export']">
                <flow-ref name="process-export-subflow" doc:name="Process Export"/>
            </when>
            <when expression="#[vars.action == 'import']">
                <flow-ref name="process-import-subflow" doc:name="Process Import"/>
            </when>
            <when expression="#[vars.action == 'generateReport']">
                <flow-ref name="process-report-generation-subflow" doc:name="Generate Report"/>
            </when>
            <otherwise>
                <raise-error type="OPERATION:UNKNOWN_ACTION" description="Unknown action type"/>
            </otherwise>
        </choice>

        <!-- Update status to COMPLETED -->
        <db:update config-ref="Database_Config" doc:name="Update to COMPLETED">
            <db:sql><![CDATA[UPDATE operations
SET status = 'COMPLETED',
    progress = 100,
    result_data = :resultData,
    completed_at = CURRENT_TIMESTAMP
WHERE operation_id = :operationId]]></db:sql>
            <db:input-parameters><![CDATA[#[{
                operationId: vars.operationId,
                resultData: write(payload, "application/json")
            }]]]></db:input-parameters>
        </db:update>

        <error-handler>
            <on-error-propagate type="ANY">
                <!-- Update status to FAILED -->
                <db:update config-ref="Database_Config" doc:name="Update to FAILED">
                    <db:sql><![CDATA[UPDATE operations
SET status = 'FAILED',
    error_code = :errorCode,
    error_message = :errorMessage,
    failed_at = CURRENT_TIMESTAMP
WHERE operation_id = :operationId]]></db:sql>
                    <db:input-parameters><![CDATA[#[{
                        operationId: vars.operationId,
                        errorCode: error.errorType.identifier,
                        errorMessage: error.description
                    }]]]></db:input-parameters>
                </db:update>
            </on-error-propagate>
        </error-handler>
    </try>
</flow>
```

### 3. Get Operation Status Flow

```xml
<flow name="get-operation-status" doc:name="Get Operation Status">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/operations/{operationId}"
                   allowedMethods="GET"
                   doc:name="GET /api/v1/operations/{operationId}"/>

    <set-variable variableName="operationId" value="#[attributes.uriParams.operationId]"/>

    <!-- Get operation from database -->
    <db:select config-ref="Database_Config" doc:name="Get Operation">
        <db:sql><![CDATA[SELECT * FROM operations WHERE operation_id = :operationId]]></db:sql>
        <db:input-parameters><![CDATA[#[{
            operationId: vars.operationId
        }]]]></db:input-parameters>
    </db:select>

    <!-- Check if operation exists -->
    <choice doc:name="Operation Exists?">
        <when expression="#[sizeOf(payload) == 0]">
            <ee:transform doc:name="Transform to 404">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "OPERATION_NOT_FOUND",
    message: "Operation with ID " ++ vars.operationId ++ " not found",
    timestamp: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="httpStatus">404</ee:set-variable>
                </ee:variables>
            </ee:transform>
        </when>
        <otherwise>
            <!-- Transform to response format -->
            <ee:transform doc:name="Build Status Response">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
var operation = payload[0]
var baseUrl = "${api.base.path}"
var isPending = operation.status == "PENDING"
var isRunning = operation.status == "RUNNING"
var isCompleted = operation.status == "COMPLETED"
var isFailed = operation.status == "FAILED"
var isCancelled = operation.status == "CANCELLED"
---
{
    operationId: operation.operation_id,
    status: operation.status,
    (progress: operation.progress) if isRunning or isCompleted,
    (result: read(operation.result_data, "application/json")) if isCompleted,
    (error: {
        code: operation.error_code,
        message: operation.error_message
    }) if isFailed,
    createdAt: operation.created_at as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"},
    (startedAt: operation.started_at as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}) if operation.started_at != null,
    (estimatedCompletion: operation.estimated_completion as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}) if isRunning,
    (completedAt: operation.completed_at as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}) if isCompleted,
    (failedAt: operation.failed_at as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}) if isFailed,
    (cancelledAt: operation.cancelled_at as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}) if isCancelled,
    links: {
        self: baseUrl ++ "/operations/" ++ operation.operation_id,
        (cancel: baseUrl ++ "/operations/" ++ operation.operation_id ++ ":cancel") if (isPending or isRunning)
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
        </otherwise>
    </choice>
</flow>
```

### 4. Cancel Operation Flow

```xml
<flow name="cancel-operation" doc:name="Cancel Operation">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/operations/{operationId}:cancel"
                   allowedMethods="POST"
                   doc:name="POST /api/v1/operations/{operationId}:cancel"/>

    <set-variable variableName="operationId" value="#[attributes.uriParams.operationId]"/>

    <!-- Update operation to CANCELLED if PENDING or RUNNING -->
    <db:update config-ref="Database_Config" doc:name="Cancel Operation">
        <db:sql><![CDATA[UPDATE operations
SET status = 'CANCELLED',
    cancelled_at = CURRENT_TIMESTAMP
WHERE operation_id = :operationId
  AND status IN ('PENDING', 'RUNNING')]]></db:sql>
        <db:input-parameters><![CDATA[#[{
            operationId: vars.operationId
        }]]]></db:input-parameters>
    </db:update>

    <!-- Check if operation was cancelled -->
    <choice doc:name="Was Cancelled?">
        <when expression="#[payload.affectedRows > 0]">
            <ee:transform doc:name="Build Cancel Response">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    operationId: vars.operationId,
    status: "CANCELLED",
    cancelledAt: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
        </when>
        <otherwise>
            <ee:transform doc:name="Transform to 400">
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "CANNOT_CANCEL",
    message: "Operation cannot be cancelled (not found or already completed/failed)",
    timestamp: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}]]></ee:set-payload>
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="httpStatus">400</ee:set-variable>
                </ee:variables>
            </ee:transform>
        </otherwise>
    </choice>
</flow>
```

### Example Processing Subflow (Report Generation)

```xml
<sub-flow name="process-report-generation-subflow" doc:name="Process Report Generation">
    <set-variable variableName="reportType" value="#[vars.inputData.reportType]"/>
    <set-variable variableName="filters" value="#[vars.inputData.filters]"/>

    <!-- Simulate progress updates -->
    <db:update config-ref="Database_Config" doc:name="Update Progress 25%">
        <db:sql><![CDATA[UPDATE operations SET progress = 25 WHERE operation_id = :operationId]]></db:sql>
        <db:input-parameters><![CDATA[#[{operationId: vars.operationId}]]]></db:input-parameters>
    </db:update>

    <!-- Fetch data for report -->
    <db:select config-ref="Database_Config" doc:name="Fetch Report Data">
        <db:sql><![CDATA[SELECT * FROM data_table WHERE category = :category]]></db:sql>
        <db:input-parameters><![CDATA[#[{category: vars.filters.category}]]]></db:input-parameters>
    </db:select>

    <db:update config-ref="Database_Config" doc:name="Update Progress 50%">
        <db:sql><![CDATA[UPDATE operations SET progress = 50 WHERE operation_id = :operationId]]></db:sql>
        <db:input-parameters><![CDATA[#[{operationId: vars.operationId}]]]></db:input-parameters>
    </db:update>

    <!-- Generate report (simulated) -->
    <ee:transform doc:name="Generate Report">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    reportId: "report_" ++ uuid(),
    totalRecords: sizeOf(payload),
    generatedAt: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"},
    downloadUrl: "${api.base.path}/reports/report_" ++ uuid() ++ ".pdf"
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <db:update config-ref="Database_Config" doc:name="Update Progress 100%">
        <db:sql><![CDATA[UPDATE operations SET progress = 100 WHERE operation_id = :operationId]]></db:sql>
        <db:input-parameters><![CDATA[#[{operationId: vars.operationId}]]]></db:input-parameters>
    </db:update>
</sub-flow>
```

## HTTP Status Codes

- **202 Accepted**: Operation created and queued for processing
- **200 OK**: Successfully retrieved operation status
- **400 Bad Request**: Cannot cancel operation (already completed/failed)
- **404 Not Found**: Operation does not exist
- **500 Internal Server Error**: Unexpected server error

## Client Polling Strategy

### Simple Polling

```javascript
async function pollOperation(operationId) {
    const maxAttempts = 60;  // 5 minutes with 5-second intervals
    let attempts = 0;

    while (attempts < maxAttempts) {
        const response = await fetch(`/api/v1/operations/${operationId}`);
        const operation = await response.json();

        if (operation.status === 'COMPLETED') {
            return operation.result;
        }

        if (operation.status === 'FAILED') {
            throw new Error(operation.error.message);
        }

        // Wait 5 seconds before next poll
        await new Promise(resolve => setTimeout(resolve, 5000));
        attempts++;
    }

    throw new Error('Operation timeout');
}
```

### Exponential Backoff Polling

```javascript
async function pollWithBackoff(operationId) {
    let delay = 1000;  // Start with 1 second
    const maxDelay = 30000;  // Max 30 seconds
    const maxAttempts = 20;
    let attempts = 0;

    while (attempts < maxAttempts) {
        const response = await fetch(`/api/v1/operations/${operationId}`);
        const operation = await response.json();

        if (operation.status === 'COMPLETED' || operation.status === 'FAILED') {
            return operation;
        }

        await new Promise(resolve => setTimeout(resolve, delay));
        delay = Math.min(delay * 2, maxDelay);  // Exponential backoff
        attempts++;
    }

    throw new Error('Operation timeout');
}
```

## Complete Flow Pattern

```
1. Client sends POST to create operation
   ↓
2. Generate unique operation ID
   ↓
3. Store operation in database (PENDING)
   ↓
4. Queue operation for async processing (VM)
   ↓
5. Return 202 Accepted with operation ID
   ↓
6. Async worker picks up operation from queue
   ↓
7. Update status to RUNNING
   ↓
8. Process operation (with progress updates)
   ↓
9. Update status to COMPLETED/FAILED
   ↓
10. Client polls for status until completed
```

## Webhook Alternative (Advanced)

Instead of polling, notify clients via webhooks when operation completes:

```xml
<flow name="notify-operation-complete" doc:name="Notify via Webhook">
    <!-- After operation completes -->
    <http:request method="POST"
                  url="#[vars.webhookUrl]"
                  config-ref="HTTP_Request_config"
                  doc:name="Send Webhook Notification">
        <http:body><![CDATA[#[{
            operationId: vars.operationId,
            status: "COMPLETED",
            result: payload
        }]]]></http:body>
    </http:request>
</flow>
```

## Testing Checklist

- [ ] Create operation returns 202 Accepted
- [ ] Operation ID is unique and valid
- [ ] Status transitions: PENDING → RUNNING → COMPLETED
- [ ] Status transitions: PENDING → RUNNING → FAILED
- [ ] Get status for non-existent operation (404)
- [ ] Get status for PENDING operation
- [ ] Get status for RUNNING operation (with progress)
- [ ] Get status for COMPLETED operation (with result)
- [ ] Get status for FAILED operation (with error)
- [ ] Cancel PENDING operation (success)
- [ ] Cancel RUNNING operation (success)
- [ ] Cancel COMPLETED operation (failure)
- [ ] Progress updates during processing
- [ ] Error handling in async processing
- [ ] Database transaction integrity
- [ ] VM queue persistence

## Common Pitfalls

❌ **Don't**: Return 200 OK when creating operation

✅ **Do**: Return 202 Accepted to indicate async processing

❌ **Don't**: Block the HTTP request while processing

✅ **Do**: Queue operation and return immediately

❌ **Don't**: Use GET to create operations

✅ **Do**: Use POST with custom action suffix (e.g., :export)

❌ **Don't**: Forget to handle operation failures

✅ **Do**: Update status to FAILED with error details

❌ **Don't**: Store large results in database

✅ **Do**: Store results in file storage and provide download URL

❌ **Don't**: Keep operation records forever

✅ **Do**: Implement cleanup job for old operations

## Performance Considerations

- **Queue Configuration**: Use persistent queues for reliability
- **Worker Scaling**: Scale async workers based on queue depth
- **Database Cleanup**: Purge old operation records (e.g., >30 days)
- **Result Storage**: Store large results in object storage (S3, blob storage)
- **Progress Updates**: Update progress at meaningful intervals (not too frequent)
- **Timeout Handling**: Implement operation timeouts to prevent hanging
- **Connection Pooling**: Configure database and HTTP connection pools

## Security Considerations

1. **Operation ID Security**: Use UUIDs, not sequential IDs
2. **Authorization**: Verify user owns the operation before showing status
3. **Rate Limiting**: Limit operation creation rate per user
4. **Result Access Control**: Secure download URLs with signed tokens
5. **Sensitive Data**: Don't expose sensitive input data in status responses
6. **Audit Logging**: Log operation creation, completion, and cancellation

## Advanced Features

### Priority Queues

Route high-priority operations to dedicated queues:

```xml
<vm:publish queueName="highPriorityQueue" config-ref="VM_Config">
    <!-- High priority operations -->
</vm:publish>
```

### Operation Expiration

Auto-cancel operations that exceed time limits:

```sql
UPDATE operations
SET status = 'FAILED',
    error_code = 'TIMEOUT',
    error_message = 'Operation exceeded maximum execution time'
WHERE status IN ('PENDING', 'RUNNING')
  AND created_at < NOW() - INTERVAL '1 hour';
```

### Batch Status Query

Get status of multiple operations in one request:

```
POST /api/v1/operations:batchGet
{
  "operationIds": ["op_123", "op_456", "op_789"]
}
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

<!-- VM Connector (for async queues) -->
<dependency>
    <groupId>org.mule.connectors</groupId>
    <artifactId>mule-vm-connector</artifactId>
    <version>2.0.0</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

## Alternative: Anypoint MQ

For production environments, consider Anypoint MQ instead of VM connector:

```xml
<anypoint-mq:config name="Anypoint_MQ_Config">
    <anypoint-mq:connection url="${mq.url}" clientId="${mq.clientId}" clientSecret="${mq.clientSecret}"/>
</anypoint-mq:config>

<anypoint-mq:publish config-ref="Anypoint_MQ_Config" destination="operations-queue"/>
<anypoint-mq:listener config-ref="Anypoint_MQ_Config" destination="operations-queue"/>
```

## Related Patterns

- **Pagination**: List operations with pagination
- **Webhooks**: Alternative to polling for status updates
- **Batch Operations**: Process multiple items in single LRO
- **Resource Expansion**: Include related resource details in operation result

## References

- [API Design Patterns by JJ Geewax](https://www.manning.com/books/api-design-patterns) - Chapter 13
- [Google API Design Guide - Long Running Operations](https://cloud.google.com/apis/design/design_patterns#long_running_operations)
- [Azure Long Running Operations](https://docs.microsoft.com/en-us/azure/architecture/patterns/async-request-reply)
- [AWS Asynchronous Request-Response Pattern](https://aws.amazon.com/blogs/compute/managing-backend-requests-and-frontend-notifications-in-serverless-web-apps/)

---

**Implementation Template**: Use `assets/mule-base-template/` as starting point
**Advanced Features**: See `references/advanced/` for webhooks and priority queues
**Troubleshooting**: See `references/troubleshooting/` for common issues
