# List Filtering & Sorting Pattern - MuleSoft Implementation Reference

## Pattern Overview

The **List Filtering & Sorting** pattern allows clients to refine collection results using query parameters. Clients can filter by field values, search text, apply date ranges, and sort results in various orders.

**Key Concept**: Provide flexible query parameters that translate into database queries, allowing clients to retrieve exactly the data they need.

## When to Use This Pattern

✅ **Use List Filtering & Sorting when**:
- Collections can contain many items
- Users need to find specific subsets of data
- Search functionality is required
- Different sorting orders are needed
- Date range queries are common
- Status or category filtering is necessary
- Combining multiple filter criteria makes sense
- Building dashboards or reports

❌ **Don't use when**:
- Collections are always small and complete
- No filtering criteria make sense for the resource
- Complex search requires full-text search engine
- Advanced query language (GraphQL) is more appropriate

## Filter Types

### 1. Exact Match Filters

Match exact field values.

```
GET /api/v1/users?status=ACTIVE
GET /api/v1/orders?customerId=123
```

### 2. Comparison Filters

Greater than, less than, between values.

```
GET /api/v1/orders?totalGreaterThan=100
GET /api/v1/users?createdAfter=2024-01-01
GET /api/v1/products?priceBetween=10,50
```

### 3. Text Search Filters

Partial text matching.

```
GET /api/v1/users?search=john
GET /api/v1/products?nameContains=laptop
```

### 4. Multi-Value Filters

Match any of several values.

```
GET /api/v1/users?status=ACTIVE,PENDING
GET /api/v1/orders?id=123,456,789
```

### 5. Combination Filters

Multiple filters combined with AND logic.

```
GET /api/v1/orders?status=SHIPPED&createdAfter=2024-01-01&totalGreaterThan=100
```

## Sorting Patterns

### 1. Single Field Sort

```
GET /api/v1/users?sort=createdAt
GET /api/v1/users?sort=-createdAt  (descending)
```

### 2. Multi-Field Sort

```
GET /api/v1/orders?sort=status,-createdAt
```

### 3. Default Sort

Always provide a default sort order for consistent results.

```
Default: sort=createdAt (descending)
```

## Required Endpoints

### Basic List with Filtering

```
GET /api/v1/{resources}?{filters}&sort={fields}&page={page}&pageSize={pageSize}
```

**Common Query Parameters**:
- Exact match: `status=ACTIVE`, `customerId=123`
- Comparison: `createdAfter=2024-01-01`, `totalGreaterThan=100`
- Search: `search=keyword`, `nameContains=text`
- Sort: `sort=field` or `sort=-field` (descending)
- Pagination: `page=1&pageSize=20`

**Example Request**:
```
GET /api/v1/orders?status=SHIPPED&createdAfter=2024-01-01&totalGreaterThan=100&sort=-createdAt&page=1&pageSize=20
```

**Response**:
```json
{
  "data": [
    {
      "id": "order_001",
      "customerId": "cust_123",
      "status": "SHIPPED",
      "total": 150.00,
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 45,
    "totalPages": 3
  },
  "filters": {
    "status": "SHIPPED",
    "createdAfter": "2024-01-01",
    "totalGreaterThan": 100
  },
  "sort": "-createdAt"
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

### List with Filtering Flow

```xml
<flow name="get-filtered-resources" doc:name="Get Filtered Resources">
    <http:listener config-ref="HTTP_Listener_config"
                   path="${api.base.path}/resources"
                   allowedMethods="GET"
                   doc:name="GET /api/v1/resources"/>

    <!-- Extract query parameters -->
    <ee:transform doc:name="Extract Filters">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
payload]]></ee:set-payload>
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="filters"><![CDATA[%dw 2.0
output application/java
var params = attributes.queryParams
---
{
    status: params.status,
    search: params.search,
    createdAfter: params.createdAfter,
    createdBefore: params.createdBefore,
    customerId: params.customerId,
    minTotal: params.minTotal,
    maxTotal: params.maxTotal
}]]></ee:set-variable>
            <ee:set-variable variableName="sort"><![CDATA[%dw 2.0
output application/java
---
attributes.queryParams.sort default "-createdAt"]]></ee:set-variable>
            <ee:set-variable variableName="page"><![CDATA[%dw 2.0
output application/java
---
attributes.queryParams.page default 1]]></ee:set-variable>
            <ee:set-variable variableName="pageSize"><![CDATA[%dw 2.0
output application/java
---
attributes.queryParams.pageSize default 20]]></ee:set-variable>
        </ee:variables>
    </ee:transform>

    <!-- Build dynamic WHERE clause -->
    <ee:transform doc:name="Build WHERE Clause">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
payload]]></ee:set-payload>
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="whereConditions"><![CDATA[%dw 2.0
output application/java
var filters = vars.filters
var conditions = []
    ++ (if (!isEmpty(filters.status)) ["status = :status"] else [])
    ++ (if (!isEmpty(filters.customerId)) ["customer_id = :customerId"] else [])
    ++ (if (!isEmpty(filters.createdAfter)) ["created_at >= :createdAfter"] else [])
    ++ (if (!isEmpty(filters.createdBefore)) ["created_at <= :createdBefore"] else [])
    ++ (if (!isEmpty(filters.minTotal)) ["total >= :minTotal"] else [])
    ++ (if (!isEmpty(filters.maxTotal)) ["total <= :maxTotal"] else [])
    ++ (if (!isEmpty(filters.search)) ["(name LIKE :search OR email LIKE :search)"] else [])
---
if (sizeOf(conditions) > 0)
    "WHERE " ++ (conditions joinBy " AND ")
else
    ""]]></ee:set-variable>
            <ee:set-variable variableName="orderByClause"><![CDATA[%dw 2.0
output application/java
var sortField = if (vars.sort startsWith "-")
    vars.sort[1 to -1]
else
    vars.sort
var sortDirection = if (vars.sort startsWith "-") "DESC" else "ASC"
---
"ORDER BY " ++ sortField ++ " " ++ sortDirection]]></ee:set-variable>
        </ee:variables>
    </ee:transform>

    <!-- Calculate offset for pagination -->
    <set-variable variableName="offset"
                  value="#[(vars.page - 1) * vars.pageSize]"
                  doc:name="Calculate Offset"/>

    <!-- Get total count with filters -->
    <ee:transform doc:name="Build Count Query">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
"SELECT COUNT(*) as total FROM resources " ++ vars.whereConditions]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <db:select config-ref="Database_Config" doc:name="Get Total Count">
        <db:sql>#[payload]</db:sql>
        <db:input-parameters><![CDATA[#[{
            status: vars.filters.status,
            customerId: vars.filters.customerId,
            createdAfter: vars.filters.createdAfter,
            createdBefore: vars.filters.createdBefore,
            minTotal: vars.filters.minTotal,
            maxTotal: vars.filters.maxTotal,
            search: if (!isEmpty(vars.filters.search)) "%" ++ vars.filters.search ++ "%" else null
        }]]]></db:input-parameters>
    </db:select>

    <set-variable variableName="totalItems" value="#[payload[0].total]"/>

    <!-- Get filtered data -->
    <ee:transform doc:name="Build Data Query">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
"SELECT * FROM resources " ++
vars.whereConditions ++ " " ++
vars.orderByClause ++ " " ++
"LIMIT :limit OFFSET :offset"]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <db:select config-ref="Database_Config" doc:name="Get Filtered Resources">
        <db:sql>#[payload]</db:sql>
        <db:input-parameters><![CDATA[#[{
            status: vars.filters.status,
            customerId: vars.filters.customerId,
            createdAfter: vars.filters.createdAfter,
            createdBefore: vars.filters.createdBefore,
            minTotal: vars.filters.minTotal,
            maxTotal: vars.filters.maxTotal,
            search: if (!isEmpty(vars.filters.search)) "%" ++ vars.filters.search ++ "%" else null,
            limit: vars.pageSize,
            offset: vars.offset
        }]]]></db:input-parameters>
    </db:select>

    <!-- Build response -->
    <ee:transform doc:name="Build Response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
var totalPages = ceil(vars.totalItems / vars.pageSize)
var activeFilters = vars.filters filterObject ((value, key) -> !isEmpty(value))
---
{
    data: payload,
    pagination: {
        page: vars.page,
        pageSize: vars.pageSize,
        totalItems: vars.totalItems,
        totalPages: totalPages,
        hasNext: vars.page < totalPages,
        hasPrevious: vars.page > 1
    },
    (filters: activeFilters) if (!isEmpty(activeFilters)),
    sort: vars.sort
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

### Alternative: Parameterized Query Builder

```xml
<sub-flow name="build-filter-query-subflow" doc:name="Build Filter Query">
    <ee:transform doc:name="Build Query Components">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/java

fun buildWhereClause(filters) = do {
    var conditions = filters
        filterObject ((value, key) -> !isEmpty(value))
        pluck ((value, key) ->
            key match {
                case "status" -> "status = :status"
                case "customerId" -> "customer_id = :customerId"
                case "createdAfter" -> "created_at >= :createdAfter"
                case "createdBefore" -> "created_at <= :createdBefore"
                case "minTotal" -> "total >= :minTotal"
                case "maxTotal" -> "total <= :maxTotal"
                case "search" -> "(name LIKE :search OR email LIKE :search)"
                else -> null
            }
        ) filter ((item) -> item != null)
    ---
    if (sizeOf(conditions) > 0)
        "WHERE " ++ (conditions joinBy " AND ")
    else
        ""
}

fun buildOrderByClause(sort) = do {
    var sortField = if (sort startsWith "-") sort[1 to -1] else sort
    var direction = if (sort startsWith "-") "DESC" else "ASC"
    ---
    "ORDER BY " ++ sortField ++ " " ++ direction
}

---
{
    whereClause: buildWhereClause(vars.filters),
    orderByClause: buildOrderByClause(vars.sort)
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>

    <set-variable variableName="whereClause" value="#[payload.whereClause]"/>
    <set-variable variableName="orderByClause" value="#[payload.orderByClause]"/>
</sub-flow>
```

## HTTP Status Codes

- **200 OK**: Successfully retrieved filtered results (even if empty)
- **400 Bad Request**: Invalid filter parameters or values
- **500 Internal Server Error**: Unexpected server error

## Common Filter Patterns

### Date Range Filters

```xml
<!-- DataWeave date parsing -->
<ee:set-variable variableName="createdAfter"><![CDATA[%dw 2.0
output application/java
---
attributes.queryParams.createdAfter as DateTime {format: "yyyy-MM-dd"}]]></ee:set-variable>
```

**SQL**:
```sql
WHERE created_at >= :createdAfter AND created_at <= :createdBefore
```

### Enum/Status Filters

```xml
<!-- Validate against allowed values -->
<validation:is-true expression="#[['ACTIVE', 'INACTIVE', 'PENDING'] contains vars.filters.status]"
                    message="Invalid status value"/>
```

**SQL**:
```sql
WHERE status = :status
```

### Search/LIKE Filters

```xml
<!-- Add wildcards for LIKE -->
<ee:set-variable variableName="searchPattern"><![CDATA[%dw 2.0
output application/java
---
"%" ++ vars.filters.search ++ "%"]]></ee:set-variable>
```

**SQL**:
```sql
WHERE name LIKE :searchPattern OR email LIKE :searchPattern
```

### Multi-Value Filters (IN clause)

```xml
<!-- Parse comma-separated values -->
<ee:set-variable variableName="statusList"><![CDATA[%dw 2.0
output application/java
---
vars.filters.status splitBy ","]]></ee:set-variable>
```

**SQL**:
```sql
WHERE status IN (:statusList)
```

### Range Filters

```xml
<!-- Parse min/max values -->
<ee:set-variable variableName="priceRange"><![CDATA[%dw 2.0
output application/java
---
{
    min: attributes.queryParams.minPrice as Number,
    max: attributes.queryParams.maxPrice as Number
}]]></ee:set-variable>
```

**SQL**:
```sql
WHERE price >= :minPrice AND price <= :maxPrice
```

## Complete Flow Pattern

```
1. HTTP Listener receives GET request with query params
   ↓
2. Extract and parse filter parameters
   ↓
3. Validate filter values (enums, dates, numbers)
   ↓
4. Build dynamic WHERE clause from filters
   ↓
5. Build ORDER BY clause from sort parameter
   ↓
6. Query total count with filters applied
   ↓
7. Query data with filters, sorting, and pagination
   ↓
8. Build response with data, pagination, and active filters
   ↓
9. Return 200 OK with results
```

## Example Use Cases

### 1. Filter by Status

```bash
GET /api/v1/orders?status=SHIPPED

Response:
{
  "data": [...],
  "filters": {"status": "SHIPPED"},
  "pagination": {...}
}
```

### 2. Date Range Filter

```bash
GET /api/v1/orders?createdAfter=2024-01-01&createdBefore=2024-01-31

Response includes orders created in January 2024
```

### 3. Search Filter

```bash
GET /api/v1/users?search=john

Response includes users with "john" in name or email
```

### 4. Multi-Filter with Sort

```bash
GET /api/v1/orders?status=PENDING&minTotal=100&sort=-createdAt

Response:
{
  "data": [...pending orders with total >= 100, sorted by date desc...],
  "filters": {
    "status": "PENDING",
    "minTotal": 100
  },
  "sort": "-createdAt",
  "pagination": {...}
}
```

### 5. Combined Filters and Pagination

```bash
GET /api/v1/products?category=Electronics&minPrice=100&maxPrice=500&sort=price&page=2&pageSize=20

Response: Page 2 of electronics products between $100-$500, sorted by price
```

## Testing Checklist

- [ ] No filters (return all results)
- [ ] Single filter (status, customerId, etc.)
- [ ] Multiple combined filters (AND logic)
- [ ] Date range filters
- [ ] Text search filters
- [ ] Sort ascending and descending
- [ ] Multi-field sort
- [ ] Filters with pagination
- [ ] Invalid filter values (400)
- [ ] Empty result set with valid filters (200)
- [ ] Special characters in search text
- [ ] SQL injection attempts (must be prevented)

## Common Pitfalls

❌ **Don't**: Build SQL queries with string concatenation

✅ **Do**: Use parameterized queries to prevent SQL injection

❌ **Don't**: Allow arbitrary field names in filters

✅ **Do**: Whitelist allowed filter fields

❌ **Don't**: Ignore invalid filter values silently

✅ **Do**: Validate and return 400 for invalid values

❌ **Don't**: Forget default sort order

✅ **Do**: Always have predictable default sorting

❌ **Don't**: Allow unbounded result sets

✅ **Do**: Combine with pagination for large collections

## Performance Considerations

### Database Indexes

Critical for filter performance:

```sql
-- Index on commonly filtered fields
CREATE INDEX idx_resources_status ON resources(status);
CREATE INDEX idx_resources_created_at ON resources(created_at);
CREATE INDEX idx_resources_customer_id ON resources(customer_id);

-- Composite index for common filter combinations
CREATE INDEX idx_resources_status_created ON resources(status, created_at DESC);

-- Full-text index for search
CREATE FULLTEXT INDEX idx_resources_search ON resources(name, description);
```

### Query Optimization

- **Selective Filters First**: Place most selective filters first
- **Avoid LIKE with Leading Wildcard**: `LIKE '%text'` can't use indexes
- **Use Covering Indexes**: Include all queried fields in index
- **Limit Result Sets**: Always use pagination with filters
- **Cache Common Queries**: Cache frequently used filter combinations

## Security Considerations

1. **SQL Injection Prevention**: Always use parameterized queries
2. **Input Validation**: Validate all filter values
3. **Authorization**: Filter results by user permissions
4. **Sensitive Data**: Don't allow filtering on sensitive fields
5. **Rate Limiting**: Prevent filter query abuse
6. **Audit Logging**: Log complex filter queries

### SQL Injection Prevention

```xml
<!-- ❌ WRONG - Vulnerable to SQL injection -->
<db:select>
    <db:sql>#["SELECT * FROM users WHERE name = '" ++ vars.filters.name ++ "'"]</db:sql>
</db:select>

<!-- ✅ CORRECT - Parameterized query -->
<db:select>
    <db:sql>SELECT * FROM users WHERE name = :name</db:sql>
    <db:input-parameters>#[{name: vars.filters.name}]</db:input-parameters>
</db:select>
```

## Advanced Features

### Custom Filter Operators

Support advanced operators in query params:

```
GET /api/v1/products?price[gte]=100&price[lte]=500
GET /api/v1/users?name[contains]=john
GET /api/v1/orders?createdAt[between]=2024-01-01,2024-01-31
```

### Saved Filters

Allow users to save filter combinations:

```
POST /api/v1/filters
{
  "name": "High Value Shipped Orders",
  "filters": {
    "status": "SHIPPED",
    "minTotal": 1000
  }
}

GET /api/v1/orders?savedFilter=filter_123
```

### Filter Presets

Provide common filter presets:

```
GET /api/v1/orders?preset=recentHighValue

Expands to:
?createdAfter=<7 days ago>&minTotal=1000&sort=-createdAt
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

- **Pagination**: Essential companion for filtered lists
- **Field Selection**: Let clients choose which fields to return
- **Resource Expansion**: Include related resources in filtered results
- **Search**: Advanced full-text search capabilities

## References

- [API Design Patterns by JJ Geewax](https://www.manning.com/books/api-design-patterns) - Chapter 9
- [Google API Design Guide - List Filtering](https://cloud.google.com/apis/design/design_patterns#list_filtering)
- [JSON API Filtering](https://jsonapi.org/format/#fetching-filtering)
- [OData Query Options](https://www.odata.org/getting-started/basic-tutorial/#queryData)

---

**Implementation Template**: Use `assets/mule-base-template/` as starting point
**Advanced Features**: See `references/advanced/` for full-text search
**Troubleshooting**: See `references/troubleshooting/` for query optimization tips
