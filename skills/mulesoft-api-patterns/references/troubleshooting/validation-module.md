# Validation Module - Best Practices

## Correct Usage in Mule 4

### validation:is-number

**✅ CORRECT - With numberType:**
```xml
<validation:is-number doc:name="Validate Limit"
                      value="#[vars.limit]"
                      numberType="INTEGER"
                      message="'limit' must be a valid number"/>
```

**Valid numberType values:**
- `INTEGER` - Whole numbers
- `DECIMAL` - Decimal numbers
- `NUMBER` - Any numeric value

### validation:is-not-empty

```xml
<validation:is-not-empty doc:name="Validate Name"
                         value="#[payload.name]"
                         message="Name cannot be empty"/>
```

### validation:matches-regex

```xml
<validation:matches-regex doc:name="Validate Email"
                          value="#[payload.email]"
                          regex="^[A-Za-z0-9+_.-]+@(.+)$"
                          message="Invalid email format"/>
```

### validation:is-email

```xml
<validation:is-email doc:name="Validate Email"
                     email="#[payload.email]"
                     message="Invalid email address"/>
```

## Dependency

```xml
<dependency>
    <groupId>org.mule.modules</groupId>
    <artifactId>mule-validation-module</artifactId>
    <version>2.0.1</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

## Error Handling

```xml
<error-handler>
    <on-error-propagate type="VALIDATION:INVALID_NUMBER">
        <ee:transform>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    "error": "VALIDATION_ERROR",
    "message": error.description,
    "timestamp": now()
}]]></ee:set-payload>
        </ee:transform>
    </on-error-propagate>
</error-handler>
```
