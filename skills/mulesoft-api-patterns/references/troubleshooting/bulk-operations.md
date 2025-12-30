# Bulk Operations - Best Practices

## db:bulk-insert in Mule 4

### ✅ CORRECT Pattern

```xml
<!-- Step 1: Transform data to payload -->
<ee:transform doc:name="Prepare Bulk Data">
    <ee:message>
        <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
(1 to 100) map {
    name: "Item " ++ $,
    description: "Description " ++ $,
    price: $ * 10.0,
    category: "Category"
}]]></ee:set-payload>
    </ee:message>
</ee:transform>

<!-- Step 2: Bulk insert from payload -->
<db:bulk-insert doc:name="Bulk Insert Records" config-ref="Database_Config">
    <db:sql><![CDATA[INSERT INTO products (name, description, price, category)
        VALUES (:name, :description, :price, :category)]]></db:sql>
</db:bulk-insert>
```

### ❌ INCORRECT Patterns

**Don't do this:**
```xml
<!-- ❌ Variables inside bulk-insert -->
<db:bulk-insert>
    <ee:variables>
        <ee:set-variable>...</ee:set-variable>
    </ee:variables>
    #[vars.data]
</db:bulk-insert>

<!-- ❌ Using bulkMode attribute (doesn't exist) -->
<db:bulk-insert bulkMode="true">
    ...
</db:bulk-insert>

<!-- ❌ Using db:input-parameters (wrong element) -->
<db:bulk-insert>
    <db:input-parameters>...</db:input-parameters>
</db:bulk-insert>
```

## Key Points

1. **Data goes in payload**, not variables
2. **Use ee:transform** before bulk-insert
3. **Bulk-insert only needs db:sql**
4. **Mule auto-maps** payload fields to SQL parameters
5. **Parameter names** must match payload field names

## Example: CSV to Database

```xml
<!-- Read CSV -->
<file:read path="data.csv" outputMimeType="application/csv"/>

<!-- Transform CSV to Java objects -->
<ee:transform>
    <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
payload map {
    productName: $.name,
    productPrice: $.price as Number,
    productStock: $.stock as Number
}]]></ee:set-payload>
</ee:transform>

<!-- Bulk insert -->
<db:bulk-insert config-ref="Database_Config">
    <db:sql><![CDATA[INSERT INTO products (name, price, stock)
        VALUES (:productName, :productPrice, :productStock)]]></db:sql>
</db:bulk-insert>
```

## Performance Tips

1. **Batch size:** 100-1000 records optimal
2. **Use transactions** for atomicity
3. **Disable indexes** temporarily for large imports
4. **Monitor connection pool** size
5. **Consider parallel processing** for very large datasets
