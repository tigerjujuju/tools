```
---
type: specs.openrewrite.org/v1beta/recipe
name: com.company.analysis.FindCompanyTypes
displayName: Find Company Type References
description: Find all references to com.company classes

recipeList:
  - org.openrewrite.java.search.FindTypes:
      fullyQualifiedTypeName: com.company..*
      checkAssignability: false
```

```
mvn org.openrewrite.maven:rewrite-maven-plugin:6.50.0:dryRun \
  -Drewrite.activeRecipes=com.company.analysis.FindCompanyTypes \
  -Drewrite.configLocation=rewrite.yml \
  -Drewrite.exportDatatables=true
```
