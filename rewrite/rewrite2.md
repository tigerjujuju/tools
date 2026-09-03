# OpenRewrite JDK 업그레이드 마이그레이션 가이드

> JDK 업그레이드에 따른 deprecated / removed API를 자동으로 수정하는 방법.
> **pom.xml을 수정하는 방식**과 **수정하지 않는 방식** 두 가지를 모두 다룹니다.

---

## 목차

1. [OpenRewrite로 무엇을 자동화할 수 있나](#1-openrewrite로-무엇을-자동화할-수-있나)
2. [소스를 직접 수정하나?](#2-소스를-직접-수정하나)
3. [실행 방식 선택하기](#3-실행-방식-선택하기)
4. [방식 A — pom.xml 수정 없이 실행 (CLI 전용)](#4-방식-a--pomxml-수정-없이-실행-cli-전용)
5. [방식 B — pom.xml에 플러그인 추가](#5-방식-b--pomxml에-플러그인-추가)
6. [방식 C — rewrite.yml 만 추가 (중간 지점)](#6-방식-c--rewriteyml-만-추가-중간-지점)
7. [`-Drewrite.*` 프로퍼티 전체 목록](#7--drewrite-프로퍼티-전체-목록)
8. [실무 운영 팁](#8-실무-운영-팁)
9. [트러블슈팅](#9-트러블슈팅)
10. [참고 링크](#10-참고-링크)

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
| **패턴화된 것만 커버** | Security Manager 제거, `Unsafe` 사용, reflection 기반 접근(strong encapsulation), 구버전 ASM / CGLib / Byte Buddy 문제는 **수작업**입니다. |
| **서드파티 호환성** | Spring Boot · Jackson · Lombok 등은 별도 레시피를 함께 돌려야 하고, 그래도 남는 건 직접 올려야 합니다. |
| **큰 diff** | 코드 스타일 현대화까지 섞이므로 composite 레시피를 한 번에 돌리면 리뷰가 어렵습니다. → [8장 참고](#8-실무-운영-팁) |

---

## 2. 소스를 직접 수정하나?

**네. `run` goal은 워킹 디렉토리의 소스를 제자리에서(in-place) 덮어씁니다.**

별도 출력 디렉토리를 만들지 않고, 백업 파일도 남기지 않습니다. `.java`, `pom.xml`, `build.gradle` 모두 직접 수정됩니다.

| Goal | 소스 수정 | 결과물 |
|---|:---:|---|
| `dryRun` | ❌ | `target/site/rewrite/rewrite.patch` 에 diff 생성 |
| `run` | ✅ | 소스 직접 수정 |

### 안전하게 돌리는 절차

사실상 필수 절차입니다.

1. **`git status`가 clean한 상태에서 시작** — 그래야 이후 `git diff`가 곧 OpenRewrite의 변경분이 됩니다.
2. **전용 브랜치를 파고** 거기서 실행
3. `dryRun`으로 패치 먼저 검토
4. `run` 실행 → `git diff` 검증 → 빌드 · 테스트 통과 확인
5. 마음에 안 들면 `git checkout .` 으로 통째로 롤백

> ⚠️ **버전관리되지 않는 디렉토리에서는 절대 `run`을 돌리지 마세요.** Git이 유일한 undo 버튼입니다.

> 📌 참고: `-Drewrite.recipeArtifactCoordinates` 방식으로 pom을 건드리지 않더라도, **소스 파일은 똑같이 수정됩니다.** "pom 수정 없음"은 *빌드 설정을 오염시키지 않는다*는 뜻이지 *아무것도 안 바뀐다*는 뜻이 아닙니다.

---

## 3. 실행 방식 선택하기

세 가지 방식이 있고, 상황에 따라 고르면 됩니다.

| | **A. CLI 전용** | **B. pom에 플러그인 추가** | **C. rewrite.yml만 추가** |
|---|---|---|---|
| 빌드 파일 변경 | 없음 | pom.xml 수정 | `rewrite.yml` 신규 파일만 |
| 명령어 길이 | 김 | 짧음 (`mvn rewrite:run`) | 중간 |
| 팀 공유 | 어려움 (명령 복붙) | 쉬움 (커밋됨) | 쉬움 (커밋됨) |
| 여러 레시피 조합 | 불편 | 편함 | 가장 편함 |
| CI 통합 | 가능하나 장황 | 쉬움 | 쉬움 |
| **적합한 경우** | **일회성 시험, 남의 저장소, 빌드 오염 금지** | 반복 실행, 팀 표준화, CI 게이트 | 단계별 마이그레이션 설계 |

> 💡 **권장 흐름**: 먼저 **A**로 시험 → 실제 진행이 결정되면 **C** 또는 **B**로 전환.

---

## 4. 방식 A — pom.xml 수정 없이 실행 (CLI 전용)

Maven은 플러그인을 pom에 선언하지 않아도 **완전한 좌표(GAV)로 직접 goal을 호출**할 수 있습니다. OpenRewrite는 이 방식을 공식 지원하며, 레시피 아티팩트도 커맨드라인에서 지정할 수 있습니다.

### 4.1 기본 형태

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeToJava21
```

명령 구조를 뜯어보면:

```
org.openrewrite.maven : rewrite-maven-plugin : 6.47.0 : dryRun
└─ groupId ──────────┘ └─ artifactId ──────┘ └ version ┘ └ goal ┘
```

| 요소 | 설명 |
|---|---|
| `-U` | 스냅샷 / `RELEASE` 버전 메타데이터를 강제로 갱신. `RELEASE`를 쓸 때 붙여 주는 게 안전합니다. |
| `:6.47.0` | 버전을 명시. **생략하면 최신 버전이 자동 선택**되어 실행할 때마다 결과가 달라질 수 있으니, 재현성이 필요하면 고정하세요. |
| `-Drewrite.recipeArtifactCoordinates` | 레시피가 들어있는 아티팩트. `groupId:artifactId:version` 형식. |
| `-Drewrite.activeRecipes` | 실행할 레시피 ID. |

### 4.2 실제 사용 순서

```bash
# 0) 안전장치 — 깨끗한 상태에서 브랜치 생성
git status                       # clean 확인
git switch -c chore/java21-openrewrite

# 1) 로드된 레시피 확인 (오타 · 좌표 검증)
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:discover \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE

# 2) dry-run — 소스 수정 없이 diff만
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeToJava21

# 3) 패치 검토
less target/site/rewrite/rewrite.patch

# 4) 실제 적용
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeToJava21

# 5) 검증
git diff
mvn clean verify
```

### 4.3 자주 쓰는 조합

**레시피 여러 개 — 쉼표로 구분**

```bash
-Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeBuildToJava21,org.openrewrite.java.migrate.net.URLConstructorToURICreate
```

**레시피 아티팩트 여러 개 — 역시 쉼표로 구분**

```bash
-Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE,org.openrewrite.recipe:rewrite-spring:RELEASE
```

**특정 경로 제외**

```bash
-Drewrite.exclusions=**/generated/**,**/target/**,**/src/test/resources/**
```

**core 라이브러리 레시피는 좌표 지정 불필요**

`org.openrewrite.java.RemoveUnusedImports` 처럼 core에 포함된 레시피는 `recipeArtifactCoordinates` 없이 바로 돌아갑니다.

```bash
mvn org.openrewrite.maven:rewrite-maven-plugin:6.47.0:run \
  -Drewrite.activeRecipes=org.openrewrite.java.RemoveUnusedImports
```

**레시피 파라미터 전달**

```bash
-Drewrite.options=comment='TODO: review',methodPattern="com.foo.Bar baz(..)"
```

> 파라미터가 여러 개거나 값에 특수문자가 섞이면 셸 이스케이프가 지옥이 됩니다. 이 경우는 [방식 C의 `rewrite.yml`](#6-방식-c--rewriteyml-만-추가-중간-지점)을 쓰세요.

### 4.4 셸 별칭으로 짧게 쓰기

명령이 길어서 반복 입력이 괴롭다면 별칭을 잡아 두면 됩니다. 이것도 프로젝트 파일은 전혀 건드리지 않습니다.

```bash
# ~/.zshrc 또는 ~/.bashrc
export RW_PLUGIN="org.openrewrite.maven:rewrite-maven-plugin:6.47.0"
export RW_ARTIFACT="org.openrewrite.recipe:rewrite-migrate-java:RELEASE"

alias rwdiscover='mvn -U $RW_PLUGIN:discover -Drewrite.recipeArtifactCoordinates=$RW_ARTIFACT'
alias rwdry='mvn -U $RW_PLUGIN:dryRun -Drewrite.recipeArtifactCoordinates=$RW_ARTIFACT -Drewrite.activeRecipes'
alias rwrun='mvn -U $RW_PLUGIN:run    -Drewrite.recipeArtifactCoordinates=$RW_ARTIFACT -Drewrite.activeRecipes'
```

```bash
rwdry=org.openrewrite.java.migrate.UpgradeToJava21
rwrun=org.openrewrite.java.migrate.UpgradeToJava21
```

### 4.5 `~/.m2/settings.xml` 프로파일로 옮기기 (개인 환경 전용)

프로젝트 pom 대신 **사용자 홈의 Maven 설정**에 프로파일을 두는 방법도 있습니다. 프로젝트 저장소에는 아무 변경도 남지 않습니다.

```xml
<!-- ~/.m2/settings.xml -->
<settings>
  <profiles>
    <profile>
      <id>rewrite-cli</id>
      <properties>
        <rewrite.recipeArtifactCoordinates>
          org.openrewrite.recipe:rewrite-migrate-java:RELEASE
        </rewrite.recipeArtifactCoordinates>
        <rewrite.exclusions>**/generated/**,**/target/**</rewrite.exclusions>
      </properties>
    </profile>
  </profiles>
</settings>
```

```bash
mvn -Prewrite-cli org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeToJava21
```

> ⚠️ 이 방식은 **본인 PC에만 적용**됩니다. 팀원은 같은 명령을 써도 동작하지 않으니, 공유가 필요하면 방식 B나 C로 가세요.

### 4.6 방식 A의 주의사항

| 항목 | 설명 |
|---|---|
| **매 실행마다 LST 재구축** | 명령 한 번에 레시피 하나씩 돌리면, 돌릴 때마다 전체 프로젝트를 다시 파싱합니다. 큰 프로젝트에서는 이게 누적되어 매우 느려집니다. → 여러 레시피를 한 번에 넘기거나 `rewrite.yml`을 쓰세요. |
| **버전 미지정 시 비결정적** | `:run` 앞의 버전을 생략하면 그때그때 최신 플러그인이 받아집니다. 팀·CI에서는 반드시 고정하세요. |
| **인증이 필요한 레시피** | Maven Central의 OSS 레시피는 인증이 필요 없습니다. 다만 Moderne 등 상용 레시피 아티팩트를 쓸 경우 `~/.m2/settings.xml`에 해당 저장소를 `<repository>`와 `<pluginRepository>` **양쪽 모두**에 등록하고 자격증명을 넣어야 합니다. |
| **소스는 그대로 수정됨** | 반복하지만, pom을 안 건드리는 것과 소스를 안 건드리는 것은 별개입니다. |
| **Gradle에는 해당 없음** | Gradle은 이런 식의 임시 플러그인 호출을 지원하지 않습니다. → [4.7 참고](#47-gradle-프로젝트의-경우) |

### 4.7 Gradle 프로젝트의 경우

Gradle에는 Maven 같은 "플러그인 좌표 직접 호출"이 없습니다. 빌드 스크립트를 건드리지 않으려면 **init script**를 씁니다. 프로젝트 파일이 아니라 별도 파일로 주입하는 방식입니다.

```groovy
// rewrite-init.gradle  (프로젝트 밖 아무 곳에나 두면 됩니다)
initscript {
    repositories { maven { url "https://plugins.gradle.org/m2" } }
    dependencies { classpath("org.openrewrite:plugin:7.19.0") }
}

rootProject {
    plugins.apply(org.openrewrite.gradle.RewritePlugin)
    dependencies {
        rewrite("org.openrewrite.recipe:rewrite-migrate-java:3.19.0")
    }
    rewrite {
        activeRecipe("org.openrewrite.java.migrate.UpgradeToJava21")
    }
    afterEvaluate {
        if (repositories.isEmpty()) {
            repositories { mavenCentral() }
        }
    }
}
```

```bash
./gradlew --init-script rewrite-init.gradle rewriteDryRun
./gradlew --init-script rewrite-init.gradle rewriteRun
```

---

## 5. 방식 B — pom.xml에 플러그인 추가

반복 실행하거나 팀에서 공유할 때는 pom에 넣는 편이 훨씬 편합니다. **`mvn rewrite:run`** 처럼 짧게 쓸 수 있는 게 가장 큰 이점입니다.

### 5.1 기본 설정

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
> `rewrite-maven-plugin` 자체에는 레시피가 거의 없습니다. 실제 레시피는 `rewrite-migrate-java` 같은 별도 아티팩트에 들어 있어서, 이 블록이 빠지면 `Recipe not found` 에러가 납니다. (CLI 방식의 `-Drewrite.recipeArtifactCoordinates`가 이 블록에 대응합니다.)

### 5.2 BOM으로 버전 관리 (권장)

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

### 5.3 프로파일로 분리 (실무 권장)

마이그레이션은 일회성 작업인데 플러그인이 pom에 계속 남아 있으면 지저분합니다. 프로파일로 격리하면 평소 빌드에 영향이 없습니다. **"pom을 최소한으로만 건드린다"**는 절충안이기도 합니다.

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

### 5.4 실행

```bash
mvn rewrite:discover   # 로드된 레시피 확인
mvn rewrite:dryRun     # → target/site/rewrite/rewrite.patch
mvn rewrite:run        # 실제 적용
```

### 5.5 사용 가능한 goal

| Goal | 설명 |
|---|---|
| `rewrite:run` | 레시피 실행 후 변경 적용 |
| `rewrite:runNoFork` | Maven lifecycle을 fork하지 않고 실행 |
| `rewrite:dryRun` | 변경사항 미리보기 (diff 파일 생성) |
| `rewrite:dryRunNoFork` | fork 없는 dry-run |
| `rewrite:discover` | 클래스패스의 사용 가능한 레시피 목록 출력 |
| `rewrite:typetable` | 여러 라이브러리 버전에 대한 타입 테이블 생성 |

---

## 6. 방식 C — rewrite.yml 만 추가 (중간 지점)

pom은 그대로 두고, 프로젝트 루트에 **`rewrite.yml` 파일 하나만** 추가하는 방식입니다. 빌드 설정은 오염되지 않으면서 레시피 조합은 파일로 관리·커밋할 수 있어, 실무에서 균형이 가장 좋습니다.

### 6.1 rewrite.yml 작성

```yaml
type: specs.openrewrite.org/v1beta/recipe
name: com.mycompany.Java21Step1
displayName: Java 21 — 빌드 설정과 deprecated API만
description: 리뷰 가능한 크기로 쪼갠 1단계.
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

파라미터가 있는 레시피는 YAML에서 훨씬 깔끔합니다:

```yaml
  - org.openrewrite.java.ChangePackage:
      oldPackageName: com.old.pkg
      newPackageName: com.new.pkg
      recursive: true
```

### 6.2 실행 — pom 수정 없이

`rewrite.yml`은 프로젝트 루트에 있으면 자동으로 읽힙니다.

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=com.mycompany.Java21Step1
```

다른 위치의 YAML을 쓰려면:

```bash
-Drewrite.configLocation=config/java21.yml
```

### 6.3 왜 이 방식이 효율적인가

- **LST를 한 번만 구축** — `recipeList`의 여러 레시피를 단일 패스로 적용합니다. 레시피마다 명령을 따로 돌리는 것보다 훨씬 빠릅니다.
- **셸 이스케이프 지옥 회피** — 파라미터가 있는 레시피를 `-Drewrite.options`로 넘기면 따옴표 처리가 까다롭습니다.
- **단계가 문서로 남음** — `Step1`, `Step2` 형태로 커밋해 두면 팀원이 진행 상황을 파악하기 쉽습니다.

---

## 7. `-Drewrite.*` 프로퍼티 전체 목록

CLI 방식에서 쓸 수 있는 프로퍼티입니다. pom의 `<configuration>` 항목과 1:1 대응합니다.

| User Property | 용도 |
|---|---|
| `rewrite.activeRecipes` | 실행할 레시피 (쉼표 구분) |
| `rewrite.activeStyles` | 적용할 코드 스타일 |
| `rewrite.recipeArtifactCoordinates` | 레시피 아티팩트 GAV (쉼표 구분) |
| `rewrite.configLocation` | `rewrite.yml` 경로 지정 |
| `rewrite.exclusions` | 파싱 제외 경로 패턴 |
| `rewrite.plainTextMasks` | 플레인 텍스트로 파싱할 파일 패턴 |
| `rewrite.additionalPlainTextMasks` | 기본 마스크를 덮어쓰지 않고 추가 |
| `rewrite.options` | 레시피 파라미터 (`key=value`) |
| `rewrite.skip` | 실행 건너뛰기 |
| `rewrite.failOnInvalidActiveRecipes` | 잘못된 레시피 ID일 때 빌드 실패 |
| `rewrite.exportDatatables` | 실행 중 생성된 데이터 테이블 내보내기 |
| `rewrite.runPerSubmodule` | 서브모듈 단위 실행 |
| `rewrite.recipeChangeLogLevel` | 변경 로그 레벨 |
| `rewrite.resolvePropertiesInYaml` | YAML 내 프로퍼티 치환 |
| `rewrite.pomCacheEnabled` | POM 캐시 사용 여부 |
| `rewrite.pomCacheDirectory` | POM 캐시 디렉토리 |
| `rewrite.checkstyleConfigFile` | Checkstyle 설정 파일 경로 |
| `rewrite.checkstyleDetectionEnabled` | Checkstyle 자동 감지 |
| `sizeThresholdMb` | 지정 크기 초과 non-Java 소스 무시 |
| `skipMavenParsing` | Maven 파싱 건너뛰기 |

`dryRun` 전용:

| User Property | 용도 |
|---|---|
| `rewrite.failOnDryRunResults` | 변경이 감지되면 빌드 실패 처리 (CI 게이트용) |

---

## 8. 실무 운영 팁

### 8.1 diff를 쪼개기

`UpgradeToJava21` 같은 composite 레시피는 한 번에 수백 개 파일을 건드려서 리뷰가 불가능한 diff가 나옵니다. 단계별로 나눠 커밋을 분리하세요.

```bash
# 커밋 1 — 빌드 설정만
-Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeBuildToJava21

# 커밋 2 — deprecated API만
-Drewrite.activeRecipes=org.openrewrite.java.migrate.net.URLConstructorToURICreate

# 커밋 3 — 코드 스타일 현대화
-Drewrite.activeRecipes=org.openrewrite.java.migrate.lang.UseTextBlocks
```

단, 단계마다 명령을 따로 돌리면 LST를 매번 재구축합니다. 단계 수가 많으면 `rewrite.yml`([방식 C](#6-방식-c--rewriteyml-만-추가-중간-지점))로 묶는 편이 빠릅니다.

### 8.2 포매터 먼저 돌리기

실행 **전에** 프로젝트 포매터를 한 번 돌려 커밋해 두세요. 그래야 이후 diff에서 "OpenRewrite가 고친 것"과 "포매팅이 바뀐 것"이 섞이지 않습니다.

### 8.3 멀티모듈 프로젝트

- 루트에서 실행하면 전체 모듈이 한 번에 처리됩니다. 각 모듈에 설정을 넣을 필요 없습니다.
- 모듈별로 따로 처리하려면 `-Drewrite.runPerSubmodule=true`
- CLI 방식도 루트에서 그대로 실행하면 됩니다.

### 8.4 CI 게이트로 활용

마이그레이션 완료 후, 구식 코드가 다시 들어오는 걸 막는 회귀 방지용으로 유용합니다.

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.47.0:dryRun \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeToJava21 \
  -Drewrite.failOnDryRunResults=true
```

CI에서는 플러그인 버전과 레시피 버전을 **`RELEASE`가 아닌 고정 버전**으로 박아 두세요. 그래야 어느 날 갑자기 파이프라인이 깨지지 않습니다.

### 8.5 권장 진행 순서

```
현재 JDK로 빌드 · 테스트 통과 확인
        ↓
포매터 실행 후 커밋
        ↓
전용 브랜치 생성 (git status clean)
        ↓
[방식 A] discover 로 레시피 로드 확인
        ↓
dryRun → rewrite.patch 검토
        ↓
단계별 run → 커밋 분리
        ↓
빌드 · 테스트 → 수동 보완
   (Security Manager, Unsafe, reflection, 바이트코드 라이브러리)
        ↓
PR 리뷰
        ↓
[선택] 방식 B/C로 전환하여 CI 게이트 구성
```

---

## 9. 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| `Recipe not found` | 레시피 아티팩트가 지정되지 않았습니다. CLI라면 `-Drewrite.recipeArtifactCoordinates`, pom이라면 플러그인 `<dependencies>`를 확인하세요. `discover` goal로 실제 로드된 목록을 볼 수 있습니다. |
| 변경이 하나도 안 일어남 | 타입 정보가 없어 매칭에 실패한 경우가 많습니다. `mvn clean` 직후가 아닌, 컴파일 가능한 상태에서 실행하세요. |
| 실행할 때마다 결과가 다름 | 플러그인 버전을 생략했거나 레시피 버전이 `RELEASE`입니다. 고정 버전으로 바꾸세요. |
| `OutOfMemoryError` | `MAVEN_OPTS="-Xmx4g"` 정도로 늘려 주세요. |
| 실행이 너무 느림 | 첫 실행은 의존성 다운로드 + LST 파싱으로 수 분 ~ 수십 분 걸립니다. 정상입니다. 레시피를 여러 번 나눠 돌리면 매번 재파싱되므로 `rewrite.yml`로 묶으세요. |
| `runNoFork`가 동작 안 함 | fork를 하지 않으므로 사전에 `mvn compile`이 되어 있어야 합니다. |
| 생성 코드가 변경됨 | `-Drewrite.exclusions=**/generated/**` 로 제외하세요. |
| 사내 저장소 레시피 401 / 403 | `~/.m2/settings.xml`에 해당 저장소를 `<repository>`와 `<pluginRepository>` **양쪽 모두**에 등록하고 `<server>`에 자격증명을 넣어야 합니다. |
| `-Drewrite.options` 파싱 오류 | 셸 따옴표 문제입니다. `rewrite.yml`에 파라미터를 적는 방식으로 바꾸세요. |

> 💡 버전 번호는 시간이 지나면 바뀝니다. 실행 전 [Maven Central](https://central.sonatype.com/artifact/org.openrewrite.maven/rewrite-maven-plugin/versions)에서 최신 버전을 확인하세요.

---

## 10. 참고 링크

- [빌드 수정 없이 Maven 프로젝트에서 실행하기 — OpenRewrite Docs](https://docs.openrewrite.org/running-recipes/running-rewrite-on-a-maven-project-without-modifying-the-build)
- [Maven plugin configuration — OpenRewrite Docs](https://docs.openrewrite.org/reference/rewrite-maven-plugin)
- [rewrite:run goal 파라미터 전체 — Plugin Docs](https://openrewrite.github.io/rewrite-maven-plugin/run-mojo.html)
- [Modernize (java/migrate 레시피 전체) — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/migrate)
- [Migrate to Java 21 — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava21)
- [Migrate to Java 25 — OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/migrate/upgradetojava25)
- [openrewrite/rewrite-migrate-java — GitHub](https://github.com/openrewrite/rewrite-migrate-java)
- [openrewrite/rewrite-maven-plugin — GitHub](https://github.com/openrewrite/rewrite-maven-plugin)
- [rewrite-maven-plugin — Maven Central (버전 확인)](https://central.sonatype.com/artifact/org.openrewrite.maven/rewrite-maven-plugin/versions)
