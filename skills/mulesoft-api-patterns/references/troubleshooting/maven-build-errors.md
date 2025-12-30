# Maven Build Errors - Troubleshooting Guide

## Common Build Errors and Solutions

### Error 1: BasicRepositoryConnectorFactory Missing

**Error Message:**
```
A required class was missing: org/eclipse/aether/connector/basic/BasicRepositoryConnectorFactory
```

**Cause:** Using mule-maven-plugin 3.8.0 or older with Maven 3.9+ or Java 17

**Solution:**
Update `pom.xml`:
```xml
<mule.maven.plugin.version>4.1.1</mule.maven.plugin.version>
```

---

### Error 2: mule-artifact.json Not Found

**Error Message:**
```
Fail to read mule application file from mule-artifact.json: File does not exist
```

**Cause:** Plugin 4.x requires mule-artifact.json in project root

**Solution:**
Create `mule-artifact.json` in TWO locations:
1. Project root (for Maven 4.x)
2. `src/main/mule/` (for Anypoint Studio)

---

### Error 3: Namespace "doc" Not Bound

**Error Message:**
```
The prefix "doc" for attribute "doc:name" is not bound
```

**Cause:** Missing doc namespace declaration

**Solution:**
Add to XML files:
```xml
xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
```

---

### Error 4: validation:is-number Missing Attribute

**Error Message:**
```
Attribute 'numberType' must appear on element 'validation:is-number'
```

**Cause:** Validation module 2.0.1 requires numberType attribute

**Solution:**
```xml
<validation:is-number value="#[vars.param]"
                      numberType="INTEGER"
                      message="Must be a number"/>
```

Valid numberType values: INTEGER, DECIMAL, NUMBER

---

### Error 5: Invalid db:bulk-insert Content

**Error Message:**
```
Element 'db:bulk-insert' cannot have character [children]
Invalid content was found starting with element 'ee:variables'
```

**Cause:** Incorrect Mule 4 syntax for bulk-insert

**Solution:**
```xml
<!-- ✅ CORRECT -->
<ee:transform doc:name="Prepare Data">
    <ee:message>
        <ee:set-payload><![CDATA[%dw 2.0
output application/java
---
[...array of objects...]
        ]]></ee:set-payload>
    </ee:message>
</ee:transform>

<db:bulk-insert doc:name="Insert" config-ref="Database_Config">
    <db:sql><![CDATA[INSERT INTO table (col1, col2)
        VALUES (:field1, :field2)]]></db:sql>
</db:bulk-insert>
```

---

## Build Command Reference

```bash
# Clean and compile
mvn clean package

# Skip tests
mvn clean package -DskipTests

# Force update dependencies
mvn clean package -U

# Debug mode
mvn clean package -X

# Offline mode
mvn clean package -o
```

---

## Requirements

- **Java:** 17.0.11+
- **Maven:** 3.8.x or 3.9.x
- **Mule Maven Plugin:** 4.1.1+
- **Mule Runtime:** 4.4.0+
