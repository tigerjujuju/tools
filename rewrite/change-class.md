# OpenRewrite — 소수 클래스 이동 (Maven CLI)

> 클래스 3개 정도를 다른 패키지로 옮기는 작업.
> pom.xml을 수정하지 않고 CLI로만 처리합니다.

---

## 목차

1. [준비](#1-준비)
2. [rewrite.yml 작성](#2-rewriteyml-작성)
3. [실행](#3-실행)
4. [YAML 위치 지정 — configLocation](#4-yaml-위치-지정--configlocation)
5. [파일조차 만들기 싫다면](#5-파일조차-만들기-싫다면)
6. [검증](#6-검증)
7. [자주 걸리는 것들](#7-자주-걸리는-것들)

---

## 1. 준비

OpenRewrite의 `run` goal은 **소스를 제자리에서 덮어씁니다.** 백업 파일을 남기지 않으므로 Git이 유일한 undo 버튼입니다.

```bash
git status                              # clean 이어야 함
git switch -c refactor/move-classes
```

버전관리되지 않는 디렉토리에서는 `run`을 돌리지 마세요.

---

## 2. rewrite.yml 작성

프로젝트 루트에 `rewrite.yml`을 만듭니다.

```yaml
type: specs.openrewrite.org/v1beta/recipe
name: com.mycompany.MoveThreeClasses
displayName: 클래스 3개 이동
description: 레거시 패키지의 클래스를 새 구조로 이동.
recipeList:
  - org.openrewrite.java.ChangeType:
      oldFullyQualifiedTypeName: com.oldcorp.biz.OrderService
      newFullyQualifiedTypeName: com.newcorp.domain.OrderService
  - org.openrewrite.java.ChangeType:
      oldFullyQualifiedTypeName: com.oldcorp.biz.OrderDao
      newFullyQualifiedTypeName: com.newcorp.infra.OrderDao
  - org.openrewrite.java.ChangeType:
      oldFullyQualifiedTypeName: com.oldcorp.util.StringHelper
      newFullyQualifiedTypeName: com.newcorp.shared.StringHelper
```

`ChangeType`이 하는 일:

- `package` 선언 갱신
- `import` 문 갱신
- 코드 내 모든 참조 갱신
- **소스 파일을 새 경로로 이동** (조건은 [7장](#7-자주-걸리는-것들) 참조)

이름까지 바꾸려면 `newFullyQualifiedTypeName`의 마지막 단어를 다르게 쓰면 됩니다. 이동과 개명이 한 번에 처리됩니다.

```yaml
  - org.openrewrite.java.ChangeType:
      oldFullyQualifiedTypeName: com.oldcorp.biz.OrderDao
      newFullyQualifiedTypeName: com.newcorp.infra.OrderRepository
```

---

## 3. 실행

```bash
# 미리보기 — 소스 수정 없음
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.activeRecipes=com.mycompany.MoveThreeClasses

# 변경 내용 확인
cat target/site/rewrite/rewrite.patch

# 실제 적용
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:run \
  -Drewrite.activeRecipes=com.mycompany.MoveThreeClasses
```

### 명령 구조

```
org.openrewrite.maven : rewrite-maven-plugin : 6.47.0 : run
└─ groupId ──────────┘ └─ artifactId ──────┘ └ version ┘ └ goal ┘
```

| 요소 | 설명 |
|---|---|
| `-U` | 메타데이터 강제 갱신 |
| `:6.47.0` | 버전 명시. 생략하면 최신 버전이 자동 선택되어 실행마다 결과가 달라질 수 있음 |
| `activeRecipes` | YAML의 `name` 값 |

> **`recipeArtifactCoordinates`가 필요 없습니다.**
> `ChangeType`은 OpenRewrite core에 포함된 레시피라 별도 아티팩트 지정 없이 바로 동작합니다.
> `rewrite-migrate-java` 같은 외부 레시피를 함께 쓸 때만 필요합니다.

`rewrite.yml`이 프로젝트 루트에 있으면 자동으로 로드되므로 별도 옵션이 필요 없습니다.

---

## 4. YAML 위치 지정 — configLocation

루트가 아닌 곳에 두거나 다른 이름을 쓰려면 `-Drewrite.configLocation`으로 지정합니다.

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:run \
  -Drewrite.configLocation=config/move-classes.yml \
  -Drewrite.activeRecipes=com.mycompany.MoveThreeClasses
```

### 경로 규칙

| 값 | 기준 |
|---|---|
| 생략 | 프로젝트 루트의 `rewrite.yml` (기본값) |
| `config/move-classes.yml` | **실행 디렉토리 기준 상대경로** |
| `/Users/me/recipes/move.yml` | 절대경로 |
| `https://internal.example.com/rewrite/move.yml` | URL — 사내 표준을 중앙 배포할 때 |

`configLocation`은 "어디서 읽을지"만 정합니다. "무엇을 실행할지"는 여전히 `activeRecipes`로 지정해야 합니다.

### 멀티모듈 주의

`run`이 서브모듈을 순회할 때 상대경로가 각 모듈 디렉토리 기준으로 해석되어 파일을 못 찾는 경우가 있습니다. 멀티모듈이면 절대경로를 쓰세요.

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:run \
  -Drewrite.configLocation=$(pwd)/config/move-classes.yml \
  -Drewrite.activeRecipes=com.mycompany.MoveThreeClasses
```

### pom에 넣는 경우

```xml
<configuration>
  <configLocation>config/move-classes.yml</configLocation>
  <activeRecipes>
    <recipe>com.mycompany.MoveThreeClasses</recipe>
  </activeRecipes>
</configuration>
```

프로젝트 밖에 YAML을 두면 마이그레이션 설정이 저장소에 아예 남지 않아, 일회성 작업일 때 깔끔합니다.

---

## 5. 파일조차 만들기 싫다면

클래스당 명령 한 줄로 처리할 수 있습니다.

```bash
RW=org.openrewrite.maven:rewrite-maven-plugin:6.47.0

mvn -U $RW:run -Drewrite.activeRecipes=org.openrewrite.java.ChangeType \
  -Drewrite.options=oldFullyQualifiedTypeName=com.oldcorp.biz.OrderService,newFullyQualifiedTypeName=com.newcorp.domain.OrderService

mvn -U $RW:run -Drewrite.activeRecipes=org.openrewrite.java.ChangeType \
  -Drewrite.options=oldFullyQualifiedTypeName=com.oldcorp.biz.OrderDao,newFullyQualifiedTypeName=com.newcorp.infra.OrderDao

mvn -U $RW:run -Drewrite.activeRecipes=org.openrewrite.java.ChangeType \
  -Drewrite.options=oldFullyQualifiedTypeName=com.oldcorp.util.StringHelper,newFullyQualifiedTypeName=com.newcorp.shared.StringHelper
```

### 한계

- `-Drewrite.options`는 **한 번에 레시피 하나분만** 넘길 수 있습니다. 클래스 수만큼 명령을 반복해야 합니다.
- 매 실행마다 **전체 프로젝트를 재파싱**합니다. 작은 프로젝트면 괜찮지만, 3개만 넘어가도 YAML 쪽이 빠릅니다.
- 값에 특수문자가 섞이면 셸 이스케이프가 까다로워집니다.

### 방식 비교

| | rewrite.yml | CLI options |
|---|---|---|
| 파일 추가 | `rewrite.yml` 하나 | 없음 |
| 실행 횟수 | 1회 | 클래스 수만큼 |
| LST 파싱 | 1회 | 실행 횟수만큼 |
| 팀 공유 | 커밋하면 재현 가능 | 명령 복붙 |
| 권장 | **3개 이상** | 1~2개 |

---

## 6. 검증

```bash
# 1) 파일이 실제로 이동했는지 — "renamed:" 로 나와야 정상
git add -A
git status --find-renames=40%

# 2) 옛 FQCN 잔재 — 자바 밖도 함께 확인
grep -rn "com\.oldcorp\.biz\.OrderService" \
  --include=*.java --include=*.xml --include=*.jsp --include=*.properties .

# 3) 컴파일
mvn clean compile
```

**1번이 핵심입니다.** Git이 `renamed:`로 인식하면 실제 diff는 경로 변경 + import 몇 줄뿐이라 리뷰가 쉽습니다.

되돌리려면:

```bash
git checkout .
```

---

## 7. 자주 걸리는 것들

### 파일이 안 옮겨졌다

`ChangeType`은 아래 조건에서만 소스 파일 경로를 갱신합니다.

| 상황 | 참조 갱신 | 파일 이동 |
|---|:---:|:---:|
| `Foo.java` 의 top-level `Foo` | ✅ | ✅ |
| 중첩 클래스 (`Foo.Bar`) | ✅ | ❌ |
| 파일명 ≠ public 클래스명 | ✅ | ❌ |
| `ignoreDefinition: true` | ✅ | ❌ |

`deleted` + `new file`로 잡히거나 파일이 제자리에 남았다면 **파일명과 public 클래스명이 어긋난 경우**를 먼저 의심하세요.

```bash
# 파일명 ≠ public class 명 인 파일 찾기
find . -name "*.java" -path "*/src/*" | while read f; do
  base=$(basename "$f" .java)
  grep -qE "^[[:space:]]*public[[:space:]]+(final[[:space:]]+|abstract[[:space:]]+)?(class|interface|enum|record)[[:space:]]+$base\b" "$f" \
    || echo "MISMATCH: $f"
done
```

### 아무 변경도 안 일어났다

- **오타** — `ChangeType`은 매칭 실패 시 에러 없이 조용히 넘어갑니다. FQCN을 다시 확인하세요.
- **타입 정보 없음** — `mvn clean` 직후가 아닌, 컴파일 가능한 상태에서 실행해야 합니다.
- **레시피 이름 불일치** — `-Drewrite.failOnInvalidActiveRecipes=true`를 붙이면 잘못된 이름일 때 빌드가 실패해 바로 알 수 있습니다.

### XML·JSP에 FQCN이 남아 있다

`ChangeType`은 **자바 코드만** 다룹니다. 다음은 대상 밖입니다.

- Spring 빈 정의 XML의 `class=` 속성
- MyBatis / iBATIS 매퍼의 `type`, `resultType`, `parameterType`
- `web.xml` 의 서블릿·필터 클래스명
- `Class.forName("...")`, 리플렉션 문자열

YAML에 텍스트 치환을 추가하면 함께 처리됩니다.

```yaml
  - org.openrewrite.text.FindAndReplace:
      find: com.oldcorp.biz.OrderService
      replace: com.newcorp.domain.OrderService
      filePattern: '**/*.xml'
```

### 실행이 느리다

첫 실행은 의존성 다운로드 + LST 파싱으로 시간이 걸립니다. 메모리가 부족하면:

```bash
export MAVEN_OPTS="-Xmx4g"
```

---

## 참고 링크

- [Change type (ChangeType) — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/changetype)
- [Maven plugin configuration — OpenRewrite Docs](https://docs.openrewrite.org/reference/rewrite-maven-plugin)
- [rewrite:run goal 파라미터 — Plugin Docs](https://openrewrite.github.io/rewrite-maven-plugin/run-mojo.html)
- [빌드 수정 없이 Maven 프로젝트에서 실행하기 — OpenRewrite Docs](https://docs.openrewrite.org/running-recipes/running-rewrite-on-a-maven-project-without-modifying-the-build)
- [rewrite-maven-plugin — Maven Central (버전 확인)](https://central.sonatype.com/artifact/org.openrewrite.maven/rewrite-maven-plugin/versions)
