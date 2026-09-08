# OpenRewrite — 패키지·클래스 이동과 클래스 인벤토리 추출

> **Part 1** 패키지 단위 / 클래스 단위로 클래스를 이동하는 방법
> **Part 2** 프로젝트 내 클래스 목록(프로젝트명 · 패키지명 · 클래스명 · 상위 클래스 · 인터페이스) 추출

---

## 목차

**Part 1 — 이동**

1. [두 레시피의 차이](#1-두-레시피의-차이)
2. [패키지 단위 이동 — ChangePackage](#2-패키지-단위-이동--changepackage)
3. [클래스 단위 이동 — ChangeType](#3-클래스-단위-이동--changetype)
4. [대량 매핑 관리 — CSV → rewrite.yml](#4-대량-매핑-관리--csv--rewriteyml)
5. [실행과 튜닝](#5-실행과-튜닝)
6. [검증](#6-검증)
7. [자바 밖의 잔재 처리](#7-자바-밖의-잔재-처리)

**Part 2 — 인벤토리 추출**

8. [내장 레시피로 먼저 뽑기](#8-내장-레시피로-먼저-뽑기)
9. [커스텀 레시피 — 프로젝트명 포함 전체 인벤토리](#9-커스텀-레시피--프로젝트명-포함-전체-인벤토리)
10. [레시피 모듈 빌드와 실행](#10-레시피-모듈-빌드와-실행)
11. [결과 활용](#11-결과-활용)
12. [컴파일 불가 코드베이스 대응](#12-컴파일-불가-코드베이스-대응)
13. [참고 링크](#13-참고-링크)

---

# Part 1 — 이동

## 1. 두 레시피의 차이

| | `ChangePackage` | `ChangeType` |
|---|---|---|
| 대상 | 패키지 (하위 포함 가능) | 클래스 하나 |
| `package` 선언 갱신 | ✅ | ✅ |
| `import` · FQ 참조 갱신 | ✅ | ✅ |
| **소스 파일 물리적 이동** | ✅ | ✅ (조건부, 아래 참조) |
| 클래스 이름 변경 | ❌ | ✅ |
| 적합한 규모 | 수백 건 이하의 규칙으로 수만 클래스 처리 | 개별 예외 처리 |

두 레시피 모두 **파일을 실제로 옮깁니다.** 문서에는 명시되어 있지 않지만 구현부에서 `withSourcePath()`로 경로를 갱신합니다.

`ChangePackage` (`ChangePackage.java`, `postVisit`):

```java
sf = ((SourceFile) sf).withSourcePath(Paths.get(path.replaceFirst(
        changingFrom.replace('.', '/'),
        changingTo.replace('.', '/')
)));
```

`ChangeType` (`ChangeType.java`, `ChangeClassDefinition`):

```java
Path newPath = Paths.get(oldPath.replaceFirst(oldFqn, newFqn));
if (updatePath(cu, oldPath, newPath.toString())) {
    cu = cu.withSourcePath(newPath);
}
```

---

## 2. 패키지 단위 이동 — ChangePackage

### 옵션

| 옵션 | 필수 | 설명 |
|---|:---:|---|
| `oldPackageName` | ✅ | 바꿀 패키지명 |
| `newPackageName` | ✅ | 새 패키지명 |
| `recursive` | | `true` 면 하위 패키지까지. **기본값 `false`** |

`recursive` 기본값이 `false`라는 점을 놓치기 쉽습니다. 하위 패키지를 통째로 옮기려면 반드시 명시하세요.

### 예시

```yaml
type: specs.openrewrite.org/v1beta/recipe
name: com.mycompany.PackageRelocation
displayName: 패키지 대량 이동
recipeList:
  - org.openrewrite.java.ChangePackage:
      oldPackageName: com.oldcorp.legacy.biz
      newPackageName: com.newcorp.domain.business
      recursive: true
  - org.openrewrite.java.ChangePackage:
      oldPackageName: com.oldcorp.legacy.dao
      newPackageName: com.newcorp.infra.persistence
      recursive: true
```

결과:

```
src/main/java/com/oldcorp/legacy/biz/OrderService.java
  → src/main/java/com/newcorp/domain/business/OrderService.java
```

### 순서 주의

레시피는 위에서 아래로 **순차 적용**되므로, 앞 규칙의 결과가 뒤 규칙에 다시 걸릴 수 있습니다.

- 접두어가 겹치는 매핑(`com.a` 와 `com.a.b`)은 **긴 것을 먼저** 배치
- 이동 결과가 다른 규칙의 `oldPackageName`과 겹치지 않는지 확인

### 현재 패키지 분포 파악

매핑 설계 전에 실제 분포를 먼저 보세요. 상위 몇 개 접두어가 대부분을 덮는다면 `recursive: true` 한 줄로 끝납니다.

```bash
find . -name "*.java" -path "*/src/main/java/*" \
  | sed 's|.*/src/main/java/||; s|/[^/]*\.java$||; s|/|.|g' \
  | sort | uniq -c | sort -rn | head -50
```

---

## 3. 클래스 단위 이동 — ChangeType

### 옵션

| 옵션 | 필수 | 설명 |
|---|:---:|---|
| `oldFullyQualifiedTypeName` | ✅ | 원래 FQCN |
| `newFullyQualifiedTypeName` | ✅ | 새 FQCN (패키지 + 이름 동시 변경 가능) |
| `ignoreDefinition` | | `true` 면 정의는 그대로 두고 **사용처만** 변경 |

### 파일이 이동하는 조건

`updatePath()`가 참을 반환할 때만 파일이 옮겨집니다.

| 상황 | 참조 갱신 | 파일 이동 |
|---|:---:|:---:|
| `Foo.java` 의 top-level `Foo` 이동 | ✅ | ✅ |
| 중첩 클래스 `Foo.Bar` (FQCN에 `$` 포함) | ✅ | ❌ |
| 파일명 ≠ public 클래스명 | ✅ | ❌ |
| `ignoreDefinition: true` | ✅ | ❌ |

레거시 코드에는 **파일명과 public 클래스명이 어긋난 파일**이 종종 있습니다. 이런 파일은 참조만 바뀌고 파일은 제자리에 남아 컴파일이 깨지므로 사전에 걸러 두세요.

```bash
# 파일명 ≠ public class 명 인 파일 찾기
find . -name "*.java" -path "*/src/*" | while read f; do
  base=$(basename "$f" .java)
  grep -qE "^[[:space:]]*public[[:space:]]+(final[[:space:]]+|abstract[[:space:]]+)?(class|interface|enum|record)[[:space:]]+$base\b" "$f" \
    || echo "MISMATCH: $f"
done
```

### 예시 — 이동과 개명을 동시에

```yaml
recipeList:
  - org.openrewrite.java.ChangeType:
      oldFullyQualifiedTypeName: com.oldcorp.legacy.biz.OrderDao
      newFullyQualifiedTypeName: com.newcorp.infra.order.OrderRepository
```

---

## 4. 대량 매핑 관리 — CSV → rewrite.yml

매핑은 CSV로 관리하고 YAML은 생성하세요. 리뷰는 CSV에서, 실행은 YAML로.

### 4-1. 패키지 매핑

```csv
# package-map.csv   (old,new)
com.oldcorp.legacy.biz,com.newcorp.domain.business
com.oldcorp.legacy.dao,com.newcorp.infra.persistence
com.oldcorp.common.util,com.newcorp.shared.util
```

```bash
#!/bin/bash
# gen-package-yml.sh
{
  echo "type: specs.openrewrite.org/v1beta/recipe"
  echo "name: com.mycompany.PackageRelocation"
  echo "displayName: 패키지 대량 이동"
  echo "recipeList:"
  awk -F',' '!/^#/ && NF==2 {
    gsub(/[ \t\r]/,"",$1); gsub(/[ \t\r]/,"",$2)
    printf "  - org.openrewrite.java.ChangePackage:\n"
    printf "      oldPackageName: %s\n", $1
    printf "      newPackageName: %s\n", $2
    printf "      recursive: true\n"
  }' "$1"
} > rewrite.yml
```

### 4-2. 클래스 매핑

```csv
# class-map.csv   (oldFqcn,newFqcn)
com.oldcorp.legacy.biz.OrderDao,com.newcorp.infra.order.OrderRepository
com.oldcorp.util.StringHelper,com.newcorp.shared.text.StringUtils
```

```bash
#!/bin/bash
# gen-class-yml.sh
{
  echo "type: specs.openrewrite.org/v1beta/recipe"
  echo "name: com.mycompany.ClassRelocation"
  echo "displayName: 클래스 단위 이동"
  echo "recipeList:"
  awk -F',' '!/^#/ && NF==2 {
    gsub(/[ \t\r]/,"",$1); gsub(/[ \t\r]/,"",$2)
    printf "  - org.openrewrite.java.ChangeType:\n"
    printf "      oldFullyQualifiedTypeName: %s\n", $1
    printf "      newFullyQualifiedTypeName: %s\n", $2
  }' "$1"
} > rewrite.yml
```

### 4-3. 섞어 쓸 때 — 접을 수 있는 건 접는다

`ChangeType` 하나하나가 독립 방문자이고 각각 전체 LST를 순회합니다. **레시피 수가 곧 순회 횟수**이므로, 3만 개를 나열하면 실행 시간이 폭발합니다.

이름이 그대로이고 패키지만 옮기는 항목은 `ChangePackage`로 접으세요.

```bash
# 이름 불변 + 패키지만 이동 → 패키지 쌍으로 집계
awk -F',' '!/^#/ && NF==2 {
  n1=$1; sub(/.*\./,"",n1)
  n2=$2; sub(/.*\./,"",n2)
  if (n1 == n2) {
    p1=$1; sub(/\.[^.]*$/,"",p1)
    p2=$2; sub(/\.[^.]*$/,"",p2)
    print p1 "," p2
  }
}' class-map.csv | sort | uniq -c | sort -rn | head -30
```

실제 재구성 작업에서는 매핑의 70~90%가 이 케이스입니다. 접고 나면 레시피 수가 수만에서 수백으로 줄어듭니다.

```yaml
recipeList:
  # 1) 개별 예외 먼저 — 개명·특수 이동
  - org.openrewrite.java.ChangeType:
      oldFullyQualifiedTypeName: com.oldcorp.legacy.biz.OrderDao
      newFullyQualifiedTypeName: com.newcorp.infra.order.OrderRepository
  # 2) 일괄 이동 나중
  - org.openrewrite.java.ChangePackage:
      oldPackageName: com.oldcorp.legacy.biz
      newPackageName: com.newcorp.domain.business
      recursive: true
```

> **반드시 예외를 먼저.** `ChangePackage`가 먼저 돌면 `ChangeType`의 `oldFullyQualifiedTypeName`이 이미 바뀐 뒤라 매칭에 실패합니다.

---

## 5. 실행과 튜닝

```bash
export MAVEN_OPTS="-Xmx16g -Xss8m -XX:+UseG1GC -XX:MaxGCPauseMillis=500"

mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.activeRecipes=com.mycompany.PackageRelocation \
  -Drewrite.exclusions='**/target/**,**/generated/**,**/node_modules/**' \
  -Drewrite.pomCacheEnabled=true \
  -Drewrite.failOnInvalidActiveRecipes=true \
  -o
```

| 옵션 | 이유 |
|---|---|
| `-Xmx16g` | 수만 클래스 LST를 메모리에 올립니다. 8g 미만이면 대부분 OOM |
| `-Xss8m` | 깊은 상속 구조에서 방문자 재귀가 스택을 넘길 수 있음 |
| `-XX:+UseG1GC` | 대용량 힙의 full GC 정지 완화 |
| `exclusions` | 생성 코드·빌드 산출물 파싱은 순수 낭비. 체감이 가장 큰 옵션 |
| `pomCacheEnabled` | 멀티모듈 POM 해석 캐싱 |
| `failOnInvalidActiveRecipes` | 수천 줄 YAML의 오타가 조용히 무시되는 것 방지 |
| `-o` | 오프라인. 의존성을 미리 받아 뒀다면 네트워크 왕복 제거 |

### LST를 한 번만 만드는 것이 전부

레시피를 나눠 여러 번 실행하면 **매번 전체 재파싱**합니다. 수만 클래스에서 파싱은 수십 분이므로, 이동 규칙은 하나의 `rewrite.yml`에 모아 **단일 패스**로 돌리세요.

### 규모별 기대치

| 클래스 수 | dry-run 예상 | 권장 힙 |
|---|---|---|
| ~5,000 | 5~15분 | 6g |
| ~20,000 | 20~60분 | 12g |
| 50,000+ | 1~3시간 | 16g+ |

### 배치로 쪼개기 (접기가 불가능할 때)

```bash
split -l 500 class-map.csv batch-

for b in batch-*; do
  echo "=== $b ==="
  ./gen-class-yml.sh "$b"
  mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:run \
    -Drewrite.activeRecipes=com.mycompany.ClassRelocation \
    -Drewrite.exclusions='**/target/**,**/generated/**' -o
  git add -A && git commit -m "refactor: relocate classes ($b)"
done
```

배치 커밋이 남는 게 부수 효과로 좋습니다. 깨졌을 때 어느 배치인지 즉시 특정됩니다. 한 배치가 10분을 넘기면 크기를 줄이세요.

---

## 6. 검증

`ChangeType`은 **매칭이 안 돼도 에러 없이 조용히 넘어갑니다.** 사후 검증이 필수입니다.

```bash
# 0) 사전 — 반드시 clean 상태의 전용 브랜치
git switch -c refactor/package-relocation
git status

# 1) 파일이 실제로 이동했는지 — rename 으로 인식되어야 정상
git add -A
git status --find-renames=40% | grep -c "renamed:"
git diff --cached --stat | tail -5

# 2) 옛 FQCN 잔재 — 0 이어야 함
cut -d',' -f1 class-map.csv | grep -v '^#' | while read fqcn; do
  hits=$(grep -rl --include='*.java' --include='*.xml' --include='*.jsp' \
           --include='*.properties' -F "$fqcn" . 2>/dev/null | wc -l)
  [ "$hits" -gt 0 ] && echo "LEFTOVER: $fqcn ($hits files)"
done

# 3) 옛 경로에 남은 파일
cut -d',' -f1 class-map.csv | grep -v '^#' | while read fqcn; do
  p="src/main/java/$(echo "$fqcn" | tr '.' '/').java"
  [ -f "$p" ] && echo "NOT MOVED: $p"
done

# 4) 컴파일
mvn clean compile
```

**1번이 핵심입니다.** Git이 이동을 `renamed:`로 인식하면 실제 diff는 경로 변경 + import 몇 줄뿐이라 PR 리뷰가 가능합니다. `deleted` + `new file`로 잡힌다면 내용도 크게 바뀐 것이니 원인을 확인하세요.

---

## 7. 자바 밖의 잔재 처리

`ChangePackage` · `ChangeType`은 자바 코드만 다룹니다. 다음은 **대상 밖**입니다.

- Spring 빈 정의 XML의 `class=` 속성
- MyBatis / iBATIS 매퍼의 `type`, `resultType`, `parameterType`
- `web.xml` 의 서블릿·필터 클래스명
- `Class.forName("...")`, 리플렉션 문자열
- `.properties`, JSP 스크립틀릿

```yaml
  - org.openrewrite.text.FindAndReplace:
      find: com.oldcorp.legacy.biz.OrderService
      replace: com.newcorp.domain.order.OrderService
      filePattern: '**/*.xml'
```

CSV에서 같은 방식으로 생성하되, **단순 문자열 치환이라 접두어가 겹치면 오작동**합니다. 긴 FQCN이 먼저 오도록 정렬하세요.

```bash
sort -r -t',' -k1 class-map.csv > class-map-sorted.csv
```

---

# Part 2 — 클래스 인벤토리 추출

## 8. 내장 레시피로 먼저 뽑기

커스텀 레시피를 만들기 전에, 내장 레시피로 얼마나 커버되는지 확인하세요.

### 8-1. FindClassHierarchy

```
recipe:     org.openrewrite.java.search.FindClassHierarchy
data table: org.openrewrite.java.table.ClassHierarchy
```

| 컬럼 | 내용 |
|---|---|
| Source Path | 소스 파일 경로 |
| Class name | 클래스 / 인터페이스 |
| Superclass | 상속한 클래스 |
| Interfaces | 구현 인터페이스 (쉼표 구분) |

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.activeRecipes=org.openrewrite.java.search.FindClassHierarchy \
  -Drewrite.exportDatatables=true \
  -Drewrite.exclusions='**/target/**,**/generated/**'
```

CSV는 `target/rewrite/datatables/` 아래에 떨어집니다. **`run`이 아니라 `dryRun`** 이므로 소스는 전혀 바뀌지 않습니다.

> **한계: 프로젝트명 컬럼이 없습니다.** `Source Path`에서 유추할 수는 있지만 멀티모듈에서 경로 규칙이 일정하지 않으면 부정확합니다. 프로젝트명이 필요하면 9장의 커스텀 레시피를 쓰세요.

### 8-2. 함께 돌리면 좋은 것

| 레시피 | 데이터 테이블 | 용도 |
|---|---|---|
| `org.openrewrite.java.search.FindCompileErrors` | `…java.table.CompileErrors` | **가장 먼저.** 타입 정보를 신뢰할 수 있는지 확인 |
| `org.openrewrite.FindSourceFiles` | `org.openrewrite.table.SourcesFiles` | 전체 파일 목록·모듈 분포 |
| `org.openrewrite.java.search.FindTypes` | `…java.table.TypeUses` | 특정 타입의 사용처 |
| `org.openrewrite.java.search.FindMethods` | `…java.table.MethodCalls` | 특정 메서드 호출처 |

`FindCompileErrors` 결과가 비어 있어야 이후 분석을 신뢰할 수 있습니다.

---

## 9. 커스텀 레시피 — 프로젝트명 포함 전체 인벤토리

프로젝트(모듈) 이름은 LST 루트에 붙은 **`JavaProject` 마커**에서 읽습니다.

```
org.openrewrite.java.marker.JavaProject
  ├─ String projectName
  └─ Publication publication   (groupId / artifactId / version)

org.openrewrite.java.marker.JavaSourceSet
  └─ String name               ("main" 또는 "test")
```

### 9-1. 데이터 테이블

`src/main/java/com/mycompany/rewrite/ClassInventory.java`

```java
package com.mycompany.rewrite;

import lombok.Value;
import org.openrewrite.Column;
import org.openrewrite.DataTable;
import org.openrewrite.Recipe;

public class ClassInventory extends DataTable<ClassInventory.Row> {

    public ClassInventory(Recipe recipe) {
        super(recipe,
              "Class inventory",
              "프로젝트 · 패키지 · 클래스 · 상위 클래스 · 인터페이스 목록");
    }

    @Value
    public static class Row {

        @Column(displayName = "Project", description = "모듈(프로젝트) 이름")
        String project;

        @Column(displayName = "Artifact id", description = "모듈 artifactId")
        String artifactId;

        @Column(displayName = "Source set", description = "main 또는 test")
        String sourceSet;

        @Column(displayName = "Package", description = "패키지명")
        String packageName;

        @Column(displayName = "Class name", description = "단순 클래스명")
        String className;

        @Column(displayName = "FQCN", description = "정규화된 클래스명")
        String fqcn;

        @Column(displayName = "Kind", description = "class / interface / enum / annotation / record")
        String kind;

        @Column(displayName = "Modifiers", description = "public, abstract, final 등")
        String modifiers;

        @Column(displayName = "Superclass", description = "상위 클래스")
        String superclass;

        @Column(displayName = "Interfaces", description = "구현 인터페이스, 쉼표 구분")
        String interfaces;

        @Column(displayName = "Nested", description = "중첩 클래스 여부 Y/N")
        String nested;

        @Column(displayName = "Type attributed", description = "타입 정보 유무 Y/N")
        String typeAttributed;

        @Column(displayName = "Source path", description = "소스 파일 경로")
        String sourcePath;
    }
}
```

### 9-2. 레시피

`src/main/java/com/mycompany/rewrite/ExtractClassInventory.java`

```java
package com.mycompany.rewrite;

import lombok.EqualsAndHashCode;
import lombok.Value;
import org.openrewrite.ExecutionContext;
import org.openrewrite.Recipe;
import org.openrewrite.SourceFile;
import org.openrewrite.TreeVisitor;
import org.openrewrite.java.JavaIsoVisitor;
import org.openrewrite.java.marker.JavaProject;
import org.openrewrite.java.marker.JavaSourceSet;
import org.openrewrite.java.tree.J;
import org.openrewrite.java.tree.JavaType;
import org.openrewrite.java.tree.TypeTree;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

@Value
@EqualsAndHashCode(callSuper = false)
public class ExtractClassInventory extends Recipe {

    transient ClassInventory inventory = new ClassInventory(this);

    @Override
    public String getDisplayName() {
        return "클래스 인벤토리 추출";
    }

    @Override
    public String getDescription() {
        return "프로젝트명, 패키지명, 클래스명, 상위 클래스, 인터페이스를 Data Table 로 기록한다.";
    }

    @Override
    public TreeVisitor<?, ExecutionContext> getVisitor() {
        return new JavaIsoVisitor<ExecutionContext>() {

            @Override
            public J.ClassDeclaration visitClassDeclaration(J.ClassDeclaration classDecl,
                                                            ExecutionContext ctx) {
                J.ClassDeclaration cd = super.visitClassDeclaration(classDecl, ctx);

                SourceFile sf = getCursor().firstEnclosing(SourceFile.class);
                J.CompilationUnit cu = getCursor().firstEnclosing(J.CompilationUnit.class);
                if (sf == null) {
                    return cd;
                }

                // ---- 프로젝트 / 소스셋 (LST 루트 마커) ----
                Optional<JavaProject> project = sf.getMarkers().findFirst(JavaProject.class);

                String projectName = project.map(JavaProject::getProjectName).orElse("");
                String artifactId = project
                        .map(JavaProject::getPublication)
                        .map(JavaProject.Publication::getArtifactId)
                        .orElse("");
                String sourceSet = sf.getMarkers().findFirst(JavaSourceSet.class)
                        .map(JavaSourceSet::getName)
                        .orElse("");

                // ---- 타입 정보 (있으면 우선) ----
                JavaType.FullyQualified type = cd.getType();
                boolean attributed = type != null;

                String fqcn;
                String packageName;
                String superclass;
                String interfaces;

                if (attributed) {
                    fqcn = type.getFullyQualifiedName();
                    packageName = type.getPackageName();

                    JavaType.FullyQualified sup = type.getSupertype();
                    superclass = (sup == null || "java.lang.Object".equals(sup.getFullyQualifiedName()))
                            ? "" : sup.getFullyQualifiedName();

                    interfaces = type.getInterfaces().stream()
                            .map(JavaType.FullyQualified::getFullyQualifiedName)
                            .collect(Collectors.joining(","));
                } else {
                    // ---- 폴백: 구문 정보만으로 (컴파일 불가 코드베이스) ----
                    packageName = (cu != null && cu.getPackageDeclaration() != null)
                            ? cu.getPackageDeclaration().getExpression().printTrimmed(getCursor())
                            : "";
                    fqcn = packageName.isEmpty()
                            ? cd.getSimpleName()
                            : packageName + "." + cd.getSimpleName();

                    superclass = cd.getExtends() == null
                            ? "" : cd.getExtends().printTrimmed(getCursor());

                    List<String> impls = new ArrayList<>();
                    if (cd.getImplements() != null) {
                        for (TypeTree t : cd.getImplements()) {
                            impls.add(t.printTrimmed(getCursor()));
                        }
                    }
                    interfaces = String.join(",", impls);
                }

                String modifiers = cd.getModifiers().stream()
                        .map(m -> m.getType().name().toLowerCase())
                        .collect(Collectors.joining(" "));

                boolean nested = getCursor().getParentTreeCursor()
                        .firstEnclosing(J.ClassDeclaration.class) != null
                        || fqcn.contains("$");

                inventory.insertRow(ctx, new ClassInventory.Row(
                        projectName,
                        artifactId,
                        sourceSet,
                        packageName,
                        cd.getSimpleName(),
                        fqcn,
                        cd.getKind().name().toLowerCase(),
                        modifiers,
                        superclass,
                        interfaces,
                        nested ? "Y" : "N",
                        attributed ? "Y" : "N",
                        sf.getSourcePath().toString()
                ));

                return cd;
            }
        };
    }
}
```

### 요점

- `visitClassDeclaration`은 **중첩 클래스까지 방문**합니다. `Nested` 컬럼으로 구분되므로, 최상위만 필요하면 CSV에서 `Nested = N`으로 필터하세요.
- `getSupertype()`이 `java.lang.Object`인 경우는 빈 값으로 처리했습니다. 명시적 상속만 남깁니다.
- `Type attributed` 컬럼을 반드시 확인하세요. `N`이 많으면 타입 정보가 안 붙은 것이라 상위 클래스·인터페이스가 **단순명(FQCN 아님)** 으로 기록됩니다.

---

## 10. 레시피 모듈 빌드와 실행

### 10-1. 레시피 모듈 pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.mycompany</groupId>
  <artifactId>rewrite-inventory</artifactId>
  <version>1.0.0</version>

  <properties>
    <maven.compiler.release>17</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.openrewrite.recipe</groupId>
        <artifactId>rewrite-recipe-bom</artifactId>
        <version>4.16.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.openrewrite</groupId>
      <artifactId>rewrite-java</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>1.18.34</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>
</project>
```

> 레시피 모듈 자체는 **Java 17 이상**으로 빌드해야 합니다. 분석 대상 코드가 Java 1.5여도 무관합니다 — 레시피는 별도 JVM 설정으로 돌아갑니다.

```bash
mvn -f rewrite-inventory/pom.xml clean install
```

### 10-2. 실행

```bash
export MAVEN_OPTS="-Xmx12g -Xss8m -XX:+UseG1GC"

mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.recipeArtifactCoordinates=com.mycompany:rewrite-inventory:1.0.0 \
  -Drewrite.activeRecipes=com.mycompany.rewrite.ExtractClassInventory \
  -Drewrite.exportDatatables=true \
  -Drewrite.exclusions='**/target/**,**/generated/**' \
  -Drewrite.failOnInvalidActiveRecipes=true
```

멀티모듈이면 **루트에서 한 번** 실행하면 전체 모듈이 처리되고, 각 행의 `Project` 컬럼에 모듈명이 들어갑니다.

출력:

```
target/rewrite/datatables/<timestamp>/com.mycompany.rewrite.ClassInventory.csv
```

---

## 11. 결과 활용

### 11-1. Excel로 넘기기

```bash
CSV=$(find target/rewrite/datatables -name "*ClassInventory.csv" | head -1)

python3 - "$CSV" <<'PY'
import sys, pandas as pd
df = pd.read_csv(sys.argv[1])
top = df[df["Nested"] == "N"]
with pd.ExcelWriter("class-inventory.xlsx", engine="openpyxl") as w:
    top.to_excel(w, sheet_name="전체", index=False)
    for proj, g in top.groupby("Project"):
        g.to_excel(w, sheet_name=str(proj)[:31], index=False)
print(f"총 {len(df)}행 / 최상위 {len(top)}개 클래스 / 모듈 {top['Project'].nunique()}개")
PY
```

### 11-2. DuckDB로 분석

```sql
CREATE TABLE inv AS
  SELECT * FROM read_csv_auto('target/rewrite/datatables/**/*ClassInventory.csv');

-- 모듈 × 패키지 분포
SELECT "Project", "Package", COUNT(*) AS classes
FROM inv WHERE "Nested" = 'N'
GROUP BY 1,2 ORDER BY classes DESC LIMIT 40;

-- 특정 상위 클래스를 상속한 클래스 (프레임워크 이관 대상 파악)
SELECT "Project", "FQCN", "Superclass"
FROM inv
WHERE "Superclass" LIKE '%HttpServlet%'
   OR "Superclass" LIKE '%SessionBean%'
ORDER BY "Project", "FQCN";

-- 인터페이스별 구현 클래스 수
SELECT trim(iface) AS iface, COUNT(*) AS impls
FROM inv, UNNEST(string_split("Interfaces", ',')) AS t(iface)
WHERE "Interfaces" <> ''
GROUP BY 1 ORDER BY impls DESC LIMIT 30;

-- 타입 정보가 안 붙은 비율 (신뢰도 지표)
SELECT "Project",
       SUM(CASE WHEN "Type attributed" = 'N' THEN 1 ELSE 0 END) AS unattributed,
       COUNT(*) AS total,
       ROUND(100.0 * SUM(CASE WHEN "Type attributed" = 'N' THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct
FROM inv GROUP BY 1 ORDER BY pct DESC;
```

### 11-3. 이동 매핑 초안 만들기

인벤토리가 있으면 Part 1의 매핑 CSV를 여기서 생성할 수 있습니다.

```sql
COPY (
  SELECT "FQCN" AS old_fqcn,
         replace("FQCN", 'com.oldcorp.legacy', 'com.newcorp.domain') AS new_fqcn
  FROM inv
  WHERE "Nested" = 'N'
    AND "Package" LIKE 'com.oldcorp.legacy%'
) TO 'class-map.csv' (HEADER false, DELIMITER ',');
```

---

## 12. 컴파일 불가 코드베이스 대응

레거시 코드가 현재 JDK로 컴파일되지 않으면 타입 정보(type attribution)가 붙지 않습니다. 9장 레시피는 폴백 경로를 갖고 있어 **동작은 하지만**, 결과의 성격이 달라집니다.

| | 타입 정보 있음 | 없음 (구문 폴백) |
|---|---|---|
| `Superclass` | FQCN | 소스에 적힌 그대로 (단순명일 수 있음) |
| `Interfaces` | FQCN | 소스에 적힌 그대로 |
| `Package` | 정확 | `package` 선언에서 추출 (정확) |
| 와일드카드 import 하의 참조 | 해석됨 | 해석 안 됨 |

### 대응 순서

1. `FindCompileErrors`로 먼저 규모 파악
2. `Type attributed = N` 비율이 높은 모듈부터 컴파일 가능하게 만들기 (의존성 보충이 대부분)
3. 그래도 안 되면 구문 폴백 결과를 쓰되, `Superclass` · `Interfaces`를 import 문과 대조해 FQCN으로 보정

```sql
-- 단순명으로 기록된 상위 클래스 (점이 없으면 미해석)
SELECT "Project", "FQCN", "Superclass"
FROM inv
WHERE "Superclass" <> '' AND "Superclass" NOT LIKE '%.%'
ORDER BY "Project";
```

**Part 1의 이동 작업도 같은 제약을 받습니다.** 타입 정보가 없으면 `ChangePackage`는 `package` 선언과 `import` 문 같은 구문적 요소는 잘 바꾸지만, 와일드카드 import 하에서 짧은 이름으로만 참조된 클래스는 놓칠 수 있습니다. **인벤토리 추출 → 컴파일 가능 상태 확보 → 이동** 순서를 권합니다.

---

## 13. 참고 링크

- [Rename package name (ChangePackage) — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/changepackage)
- [Change type (ChangeType) — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/changetype)
- [Find class hierarchy — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/search/findclasshierarchy)
- [Recipes with Data Tables — OpenRewrite Docs](https://docs.openrewrite.org/reference/recipes-with-data-tables)
- [Creating recipes with data tables — OpenRewrite Docs](https://docs.openrewrite.org/authoring-recipes/data-tables)
- [Framework provided markers — OpenRewrite Docs](https://docs.openrewrite.org/reference/framework-provided-markers)
- [Maven plugin configuration — OpenRewrite Docs](https://docs.openrewrite.org/reference/rewrite-maven-plugin)
- [ChangePackage.java — GitHub](https://github.com/openrewrite/rewrite/blob/main/rewrite-java/src/main/java/org/openrewrite/java/ChangePackage.java)
- [ChangeType.java — GitHub](https://github.com/openrewrite/rewrite/blob/main/rewrite-java/src/main/java/org/openrewrite/java/ChangeType.java)
- [JavaProject.java (marker) — GitHub](https://github.com/openrewrite/rewrite/blob/main/rewrite-java/src/main/java/org/openrewrite/java/marker/JavaProject.java)
