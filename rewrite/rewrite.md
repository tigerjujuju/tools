# OpenRewrite로 JDK 업그레이드 마이그레이션 하기

> JDK 업그레이드에 따른 deprecated / removed API를 자동으로 수정하는 방법.
> Maven 기준으로 작성했으며, Gradle 대응 명령도 함께 표기했습니다.

---

## 목차

1. [OpenRewrite로 무엇을 자동화할 수 있나](#1-openrewrite로-무엇을-자동화할-수-있나)
2. [소스를 직접 수정하나?](#2-소스를-직접-수정하나)
3. [pom.xml에 플러그인 추가하기](#3-pomxml에-플러그인-추가하기)
4. [실행하기](#4-실행하기)
5. [실무 운영 팁](#5-실무-운영-팁)
6. [트러블슈팅](#6-트러블슈팅)
7. [참고 링크](#7-참고-링크)

---

## 1. OpenRewrite로 무엇을 자동화할 수 있나

`rewrite-migrate-java` 모듈이 JDK 업그레이드 전용 레시피를 제공합니다.

### 핵심 레시피

LTS 단위 composite 레시피이며 **누적 적용**됩니다. 즉 `UpgradeToJava25`를 돌리면 Java 21 / 17 / 11 의 변경사항이 모두 포함됩니다.

| 레시피 ID | 대상 |
|---|---|
| `org.openrewrite.java.migrate.Java8toJava11` | Java 8 → 11 |
| `org.openrewrite.java.migrate.UpgradeToJava17` | → 17 |
| `org.openrewrite.java.migrate.UpgradeToJava21` | → 21 |
| `org.openrewrite.java.migrate.UpgradeToJava25` | → 25 |

### 자동으로 처리되는 deprecated 항목 (Java 21 기준 예시)

- `Thread.stop()` / `resume()` / `suspend()` 제거
- `new URL(String)` → `URI.create(String).toURL()`
- `new Locale(...)` → `Locale.of(...)`
- deprecated `Runtime.exec()` 오버로드 치환
- `javax.security.auth.Subject` 신규 메서드 채택
- `java.desktop` 의 `finalize()` 대응
- **Java 11 이전**: JDK에서 제거된 JAXB / JAX-WS 등을 외부 의존성으로 추가 (JEP 320)
- **Jakarta EE 9**: `javax.*` → `jakarta.*` 네임스페이스 전환

### 덤으로 따라오는 것들

- 빌드 파일 수정 — Maven / Gradle의 `source` · `target` · `release`, toolchain 설정
- **플러그인 자동 업그레이드** — 대상 JDK 호환 버전으로. 실무에서 제일 귀찮은 부분이라 체감 효과가 큽니다.
- 언어 기능 현대화 — switch expression, pattern matching, `SequencedCollection`, text block 등

### 한계 — 기대치 조정이 필요한 부분

| 항목 | 설명 |
|---|---|
| **컴파일 가능해야 함** | 타입 정보(LST) 기반이라 현재 JDK로 빌드가 되는 상태에서 돌려야 정확합니다. 이미 깨진 코드에는 잘 붙지 않습니다. |
| **패턴화된 것만 커버** | Security Manager 제거, `Unsafe` 사용, reflection 기반 접근(strong encapsulation), 구버전 ASM/CGLib/Byte Buddy 문제는 **수작업**입니다. |
| **서드파티 호환성** | Spring Boot · Jackson · Lombok 등은 별도 레시피를 함께 돌려야 하고, 그래도 남는 건 직접 올려야 합니다. |
| **큰 diff** | 코드 스타일 현대화까지 섞이므로 composite 레시피를 한 번에 돌리면 리뷰가 어렵습니다. → [5장 참고](#5-실무-운영-팁) |

---

## 2. 소스를 직접 수정하나?

**네. `run` goal은 워킹 디렉토리의 소스를 제자리에서(in-place) 덮어씁니다.**

별도 출력 디렉토리를 만들지 않고, 백업 파일도 남기지 않습니다. `.java`, `pom.xml`, `build.gradle` 모두 직접 수정됩니다.

### 두 가지 모드

| 명령 | 소스 수정 | 결과물 |
|---|:---:|---|
| `rewrite:dryRun` | ❌ | `target/site/rewrite/rewrite.patch` 에 diff 생성 |
| `rewrite:run` | ✅ | 소스 직접 수정 |

### 안전하게 돌리는 절차

사실상 필수 절차입니다.

1. **`git status`가 clean한 상태에서 시작** — 그래야 이후 `git diff`가 곧 OpenRewrite의 변경분이 됩니다.
2. **전용 브랜치를 파고** 거기서 실행
3. `dryRun`으로 패치 먼저 검토
4. `run` 실행 → `git diff` 검증 → 빌드 · 테스트 통과 확인
5. 마음에 안 들면 `git checkout .` 으로 통째로 롤백

> ⚠️ **버전관리되지 않는 디렉토리에서는 절대 `run`을 돌리지 마세요.** Git이 유일한 undo 버튼입니다.

---

## 3. pom.xml에 플러그인 추가하기

### 3.1 기본 설정

`<project>` → `<build>` → `<plugins>` 아래에 추가합니다.

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.openrewrite.maven</groupId>
      <artifactId>rewrite-maven-plugin</artifactId>
      <version>6.47.0</version>
      <configuration>
        <activeRecipes>
          <recipe>org.openrewrite.java.migrate.UpgradeToJava21</recipe>
        </activeRecipes>
      </configuration>
      <dependencies>
        <!-- 레시피가 들어있는 모듈. 이게 없으면 레시피를 못 찾습니다 -->
        <dependency>
          <groupId>org.openrewrite.recipe</groupId>
          <artifactId>rewrite-migrate-java</artifactId>
          <version>3.19.0</version>
        </dependency>
      </dependencies>
    </plugin>
  </plugins>
</build>
```

> 🔴 **가장 흔한 실수: `<dependencies>` 블록 누락**
> `rewrite-maven-plugin` 자체에는 레시피가 거의 없습니다. 실제 레시피는 `rewrite-migrate-java` 같은 별도 아티팩트에 들어 있어서, 이 블록이 빠지면 `Recipe not found` 에러가 납니다.

pom에 등록해 두면 긴 좌표 없이 **`mvn rewrite:run`** 으로 짧게 쓸 수 있습니다. pom에 넣는 가장 큰 이점입니다.

### 3.2 BOM으로 버전 관리 (권장)

레시피 모듈을 여러 개 쓸 때 개별 버전을 맞추기 번거롭습니다. `rewrite-recipe-bom`을 쓰면 버전이 자동 정렬됩니다.

프로젝트 최상위 `<dependencyManagement>`:

```xml
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
```

플러그인 쪽에서는 버전을 생략합니다:

```xml
<dependencies>
  <dependency>
    <groupId>org.openrewrite.recipe</groupId>
    <artifactId>rewrite-migrate-java</artifactId>
  </dependency>
  <dependency>
    <groupId>org.openrewrite.recipe</groupId>
    <artifactId>rewrite-spring</artifactId>
  </dependency>
</dependencies>
```

> BOM의 `import` scope는 플러그인의 `<dependencies>` 안에서 동작하지 않습니다. 반드시 최상위 `<dependencyManagement>`에 두세요.

### 3.3 프로파일로 분리 (실무 권장)

마이그레이션은 일회성 작업인데 플러그인이 pom에 계속 남아 있으면 지저분합니다. 프로파일로 격리하면 평소 빌드에 영향이 없습니다.

```xml
<profiles>
  <profile>
    <id>rewrite</id>
    <build>
      <plugins>
        <plugin>
          <groupId>org.openrewrite.maven</groupId>
          <artifactId>rewrite-maven-plugin</artifactId>
          <version>6.47.0</version>
          <configuration>
            <activeRecipes>
              <recipe>org.openrewrite.java.migrate.UpgradeToJava21</recipe>
            </activeRecipes>
            <exclusions>
              <exclusion>**/generated/**</exclusion>
              <exclusion>**/target/**</exclusion>
            </exclusions>
            <failOnDryRunResults>false</failOnDryRunResults>
          </configuration>
          <dependencies>
            <dependency>
              <groupId>org.openrewrite.recipe</groupId>
              <artifactId>rewrite-migrate-java</artifactId>
              <version>3.19.0</version>
            </dependency>
          </dependencies>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

```bash
mvn -Prewrite rewrite:dryRun
mvn -Prewrite rewrite:run
```

### 3.4 주요 설정 옵션

| 옵션 | 용도 |
|---|---|
| `activeRecipes` | 실행할 레시피 지정 |
| `activeStyles` | 적용할 코드 스타일 지정 |
| `exclusions` | 파싱에서 제외할 경로 패턴 |
| `plainTextMasks` | 플레인 텍스트로 파싱할 파일 지정 |
| `additionalPlainTextMasks` | 기본 마스크를 덮어쓰지 않고 추가 |
| `failOnDryRunResults` | dry-run에서 변경이 감지되면 빌드 실패 처리 (CI 게이트용) |
| `runPerSubmodule` | 통합 실행 대신 서브모듈 단위로 실행 |
| `sizeThresholdMb` | 지정 크기 초과 non-Java 소스 무시 |
| `exportDatatables` | 실행 중 생성된 데이터 테이블 내보내기 |

---

## 4. 실행하기

### Maven

```bash
# 1. 어떤 레시피가 로드됐는지 확인
mvn rewrite:discover

# 2. dry-run — 소스 수정 없이 diff만 생성
mvn rewrite:dryRun
#    → target/site/rewrite/rewrite.patch

# 3. 패치 내용 검토
cat target/site/rewrite/rewrite.patch

# 4. 실제 적용
mvn rewrite:run

# 5. 검증
git diff
mvn clean verify
```

### pom 수정 없이 일회성 실행

플러그인을 pom에 추가하기 전에 잠깐 시험해 보고 싶을 때:

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeToJava21
```

### Gradle

```groovy
plugins {
    id("org.openrewrite.rewrite") version "7.19.0"
}

rewrite {
    activeRecipe("org.openrewrite.java.migrate.UpgradeToJava21")
}

dependencies {
    rewrite("org.openrewrite.recipe:rewrite-migrate-java:3.19.0")
}
```

```bash
./gradlew rewriteDryRun   # → build/reports/rewrite/rewrite.patch
./gradlew rewriteRun
```

### 사용 가능한 goal

| Goal | 설명 |
|---|---|
| `rewrite:run` | 레시피 실행 후 변경 적용 |
| `rewrite:runNoFork` | Maven lifecycle을 fork하지 않고 실행 |
| `rewrite:dryRun` | 변경사항 미리보기 (diff 파일 생성) |
| `rewrite:dryRunNoFork` | fork 없는 dry-run |
| `rewrite:discover` | 클래스패스의 사용 가능한 레시피 목록 출력 |
| `rewrite:typetable` | 여러 라이브러리 버전에 대한 타입 테이블 생성 |

---

## 5. 실무 운영 팁

### 5.1 diff를 쪼개기

`UpgradeToJava21` 같은 composite 레시피는 한 번에 수백 개 파일을 건드려서 리뷰가 불가능한 diff가 나옵니다. 하위 레시피로 나눠 커밋을 분리하세요.

```bash
# 커밋 1 — 빌드 설정만
-Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeBuildToJava21

# 커밋 2 — deprecated API만
-Drewrite.activeRecipes=org.openrewrite.java.migrate.net.URLConstructorToURICreate

# 커밋 3 — 코드 스타일 현대화
-Drewrite.activeRecipes=org.openrewrite.java.migrate.lang.UseTextBlocks
```

### 5.2 `rewrite.yml`로 단계별 레시피 조합

프로젝트 루트에 `rewrite.yml`을 두고 커스텀 composite 레시피를 정의하면, 단계를 명시적으로 관리할 수 있습니다.

```yaml
type: specs.openrewrite.org/v1beta/recipe
name: com.mycompany.Java21Step1
displayName: Java 21 — 빌드 설정과 deprecated API만
recipeList:
  - org.openrewrite.java.migrate.UpgradeBuildToJava21
  - org.openrewrite.java.migrate.net.URLConstructorToURICreate
  - org.openrewrite.java.migrate.util.UseLocaleOf
---
type: specs.openrewrite.org/v1beta/recipe
name: com.mycompany.Java21Step2
displayName: Java 21 — 코드 현대화
recipeList:
  - org.openrewrite.java.migrate.lang.UseTextBlocks
  - org.openrewrite.staticanalysis.ReplaceStringBuilderWithString
```

pom의 `activeRecipes`에 `com.mycompany.Java21Step1`을 지정해서 단계별로 실행합니다.

### 5.3 포매터 먼저 돌리기

실행 **전에** 프로젝트 포매터를 한 번 돌려 커밋해 두세요. 그래야 이후 diff에서 "OpenRewrite가 고친 것"과 "포매팅이 바뀐 것"이 섞이지 않습니다.

### 5.4 멀티모듈 프로젝트

- 플러그인은 **루트 pom에만** 선언하고 루트에서 실행 → 전체 모듈이 한 번에 처리됩니다.
- 각 모듈에 넣을 필요 없습니다.
- 모듈별로 따로 처리하려면 `<runPerSubmodule>true</runPerSubmodule>`

### 5.5 CI 게이트로 활용

`failOnDryRunResults`를 켜면, 마이그레이션이 필요한 코드가 새로 들어올 때 CI가 실패합니다. 마이그레이션 완료 후 회귀 방지용으로 유용합니다.

```bash
mvn -Prewrite rewrite:dryRun -Drewrite.failOnDryRunResults=true
```

### 5.6 권장 진행 순서

```
현재 JDK로 빌드 · 테스트 통과 확인
        ↓
포매터 실행 후 커밋
        ↓
전용 브랜치 생성
        ↓
rewrite:discover 로 레시피 로드 확인
        ↓
rewrite:dryRun → 패치 검토
        ↓
단계별 rewrite:run → 커밋 분리
        ↓
빌드 · 테스트 → 수동 보완 (Security Manager, reflection 등)
        ↓
PR 리뷰
```

---

## 6. 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| `Recipe not found` | 플러그인 `<dependencies>`에 레시피 모듈이 빠졌습니다. `rewrite:discover`로 로드된 레시피를 확인하세요. |
| 변경이 하나도 안 일어남 | 타입 정보가 없어서 매칭에 실패한 경우가 많습니다. `mvn clean` 직후가 아닌, 컴파일 가능한 상태에서 실행하세요. |
| `OutOfMemoryError` | `MAVEN_OPTS="-Xmx4g"` 정도로 늘려 주세요. |
| 실행이 너무 느림 | 첫 실행은 의존성 다운로드 + LST 파싱으로 수 분 ~ 수십 분 걸립니다. 정상입니다. |
| `runNoFork`가 동작 안 함 | fork를 하지 않으므로 사전에 `mvn compile`이 되어 있어야 합니다. |
| 생성 코드가 변경됨 | `<exclusions>`에 `**/generated/**` 등을 추가하세요. |

> 💡 버전 번호는 시간이 지나면 바뀝니다. 실행 전 [Maven Central](https://central.sonatype.com/artifact/org.openrewrite.maven/rewrite-maven-plugin/versions)에서 최신 버전을 확인하세요.

---

## 7. 참고 링크

- [Maven plugin configuration — OpenRewrite Docs](https://docs.openrewrite.org/reference/rewrite-maven-plugin)
- [Modernize (java/migrate 레시피 전체) — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/migrate)
- [Migrate to Java 21 — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava21)
- [Migrate to Java 25 — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava25)
- [openrewrite/rewrite-migrate-java — GitHub](https://github.com/openrewrite/rewrite-migrate-java)
- [openrewrite/rewrite-maven-plugin — GitHub](https://github.com/openrewrite/rewrite-maven-plugin)
- [rewrite-maven-plugin — Maven Central (버전 확인)](https://central.sonatype.com/artifact/org.openrewrite.maven/rewrite-maven-plugin/versions)
