# jQAssistant 기반 Maven 프로젝트 의존성·구조 분석 가이드

> 대상: Maven 기반 Java 애플리케이션 (레거시 포함)
> 기준 버전: jQAssistant 2.9.x (Neo4j 5 내장, JDK 17 이상 권장)
> 목적: 프로젝트 + 모든 의존 라이브러리(JAR)의 클래스·패키지·메서드 간 참조관계를 그래프 DB로 만들고 Cypher로 분석

---

## 목차

1. [개요와 데이터 모델](#1-개요와-데이터-모델)
2. [사전 준비](#2-사전-준비)
3. [Maven으로 실행하기 (상세)](#3-maven으로-실행하기-상세)
4. [pom.xml 수정 없이 Maven으로 실행하기](#4-pomxml-수정-없이-maven으로-실행하기)
5. [CLI로 실행하기](#5-cli로-실행하기)
6. [내장 규칙(Concept/Constraint) 적용](#6-내장-규칙conceptconstraint-적용)
7. [결과 탐색과 내보내기](#7-결과-탐색과-내보내기)
8. [분석 쿼리 모음](#8-분석-쿼리-모음)
9. [커스텀 규칙 파일로 자동화](#9-커스텀-규칙-파일로-자동화)
10. [트러블슈팅](#10-트러블슈팅)
11. [참고 자료](#11-참고-자료)

---

## 1. 개요와 데이터 모델

jQAssistant는 빌드 산출물(클래스 파일, JAR, pom.xml 등)을 스캔해 Neo4j 그래프로 저장하고, Cypher 쿼리(규칙)로 분석합니다.

```
scan  →  (Neo4j 그래프)  →  analyze (concept/constraint)  →  report / server(브라우저 탐색)
```

### 1.1 주요 노드

| 라벨 | 의미 | 주요 속성 |
|---|---|---|
| `:Java:Artifact:Directory` | 디렉터리 산출물 (예: `target/classes`) | `fileName` |
| `:Java:Artifact:Jar:Archive` | JAR 파일 | `fileName` |
| `:Java:Package` | 패키지 | `fqn`, `name` |
| `:Java:Type` (`:Class`, `:Interface`, `:Enum`, `:Record`, `:Annotation`) | 타입 | `fqn`, `name`, `visibility`, `abstract`, `final`, `byteCodeVersion` |
| `:Java:Method` (`:Constructor`) | 메서드 | `name`, `signature`, `visibility`, `static`, `cyclomaticComplexity`, `effectiveLineCount` |
| `:Java:Field` | 필드 | `name`, `signature`, `static`, `final` |

### 1.2 주요 관계

| 관계 | 의미 |
|---|---|
| `(Artifact)-[:CONTAINS]->(Package/Type)` | 산출물이 포함하는 패키지/클래스 |
| `(Artifact)-[:REQUIRES]->(Type)` | 산출물 내 클래스가 필요로 하는 외부 타입 |
| `(Package)-[:CONTAINS]->(Type)` | 패키지가 포함하는 타입 |
| `(Type)-[:DEPENDS_ON {weight}]->(Type)` | 타입 의존 (모든 참조). `weight`는 참조 횟수 |
| `(Type)-[:EXTENDS]->(Type)` / `[:IMPLEMENTS]` | 상속 / 구현 (인터페이스 간 상속도 `IMPLEMENTS`) |
| `(Type)-[:DECLARES]->(Method/Field/Type)` | 멤버 및 내부 클래스 선언 |
| `(Method)-[:INVOKES {lineNumber}]->(Method)` | 메서드 호출 |
| `(Method)-[:READS/WRITES]->(Field)` | 필드 읽기/쓰기 |
| `(Method)-[:THROWS/CATCHES]->(Type)` | 예외 선언·throw / catch |
| `(*)-[:ANNOTATED_BY]->(Annotation)-[:OF_TYPE]->(Type)` | 어노테이션 |

### 1.3 ⚠️ 반드시 알아야 할 점: 참조용(stub) 타입 노드

**산출물(JAR/디렉터리)마다 자기만의 Type 노드를 가집니다.** A.jar의 클래스가 B.jar의 클래스를 참조하면, `DEPENDS_ON`은 B.jar 안의 실제 노드가 아니라 **A.jar 쪽에서 만든 참조용 노드**(`fqn`만 있음, `(A.jar)-[:REQUIRES]->`로 연결)를 가리킵니다.

따라서 **JAR 간 / 모듈 간 관계를 볼 때는 노드가 아니라 `fqn` 문자열로 조인**해야 합니다. 이 문서의 쿼리는 모두 이 점을 반영했습니다.

```cypher
// 잘못된 예 (JAR 간 관계가 거의 안 나옴)
MATCH (a1:Artifact)-[:CONTAINS]->(:Type)-[:DEPENDS_ON]->(:Type)<-[:CONTAINS]-(a2:Artifact)

// 올바른 예 (fqn으로 조인)
MATCH (a1:Artifact)-[:CONTAINS]->(:Type)-[:DEPENDS_ON]->(r:Type)
MATCH (a2:Artifact)-[:CONTAINS]->(t2:Type) WHERE t2.fqn = r.fqn
```

---

## 2. 사전 준비

| 항목 | 요구사항 |
|---|---|
| JDK | 17 이상 권장 (Neo4j 5 내장). JDK 11이면 Neo4j 4 배포판 사용 |
| Maven | 3.5 이상 |
| 분석 대상 | **분석 대상 코드가 Java 1.5여도 무관** — 바이트코드를 스캔하므로 컴파일 결과물만 있으면 됨 |

### 2.1 Maven 단축 명령 등록 (선택)

`~/.m2/settings.xml`에 추가하면 `mvn jqassistant:scan`처럼 짧게 실행할 수 있습니다.

```xml
<settings>
  <pluginGroups>
    <pluginGroup>com.buschmais.jqassistant</pluginGroup>
  </pluginGroups>
</settings>
```

> pom.xml에 플러그인을 선언하지 않고 실행하는 방법은 **4장**에 정리했습니다.

### 2.2 메모리 설정

라이브러리가 수백 개면 힙이 부족할 수 있습니다.

```bash
# Maven 실행 시
export MAVEN_OPTS="-Xmx4g"

# CLI 실행 시
export JQASSISTANT_OPTS="-Xmx4g"
```

Windows는 `set MAVEN_OPTS=-Xmx4g`.

---

## 3. Maven으로 실행하기 (상세)

### 3.1 핵심 원리

jQAssistant Maven 플러그인은 기본적으로 **각 모듈의 `target/classes`, `target/test-classes`, `pom.xml`만** 스캔합니다. 라이브러리 JAR 내부까지 분석하려면:

1. `maven-dependency-plugin`으로 의존 JAR를 한 폴더로 복사하고
2. `.jqassistant.yml`의 `jqassistant.scan.include.files`로 그 폴더를 스캔 대상에 추가합니다.

### 3.2 pom.xml에 플러그인 등록

```xml
<properties>
    <jqassistant.version>2.9.1</jqassistant.version>  <!-- 최신 버전은 Maven Central에서 확인 -->
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>com.buschmais.jqassistant</groupId>
            <artifactId>jqassistant-maven-plugin</artifactId>
            <version>${jqassistant.version}</version>
            <!-- 빌드 라이프사이클에 자동 연결하지 않고 수동 실행만 할 경우 executions 생략 가능 -->
        </plugin>
    </plugins>
</build>
```

빌드할 때마다 자동으로 돌게 하려면 `executions`를 추가합니다 (보통 분석 전용 프로파일에 넣는 것을 권장, 3.7절 참고).

```xml
<executions>
    <execution>
        <id>default-cli</id>
        <goals>
            <goal>scan</goal>
            <goal>analyze</goal>
        </goals>
    </execution>
</executions>
```

### 3.3 `.jqassistant.yml` 작성 (프로젝트 루트)

```yaml
jqassistant:
  scan:
    reset: true                 # 스캔 전 DB 초기화
    continue-on-error: true     # 깨진 클래스 파일이 있어도 계속 진행 (레거시에서 유용)
    include:
      files:
        - ${project.basedir}/target/jqa-libs    # 복사한 의존 JAR 폴더

  analyze:
    report:
      fail-on-severity: NEVER   # 분석 목적일 때는 빌드 실패 방지
      warn-on-severity: MINOR

  server:
    open-browser: true          # server 실행 시 브라우저 자동 오픈
```

> `${project.basedir}` 같은 Maven 속성을 `.jqassistant.yml`에서 사용할 수 있습니다.

### 3.4 단계별 실행 명령 (단일 모듈)

```bash
# 1) 컴파일 (테스트 생략)
mvn clean package -DskipTests

# 2) 의존 JAR(전이 의존성 포함)를 한 폴더로 복사
mvn dependency:copy-dependencies \
    -DoutputDirectory=target/jqa-libs \
    -DincludeScope=runtime

# 3) 스캔: target/classes + pom.xml + target/jqa-libs/*.jar → Neo4j
mvn jqassistant:scan

# 4) 분석: 내장/커스텀 규칙 적용, 리포트 생성
mvn jqassistant:analyze

# 5) 내장 Neo4j 서버 실행 → http://localhost:7474 (Enter 누르면 종료)
mvn jqassistant:server
```

한 줄로:

```bash
mvn clean package dependency:copy-dependencies jqassistant:scan jqassistant:analyze \
    -DskipTests -DoutputDirectory=target/jqa-libs -DincludeScope=runtime \
  && mvn jqassistant:server
```

> Windows PowerShell에서는 줄바꿈 `\` 대신 백틱(`` ` ``)을 쓰거나 한 줄로 입력하세요.

### 3.5 `dependency:copy-dependencies` 주요 옵션

| 옵션 | 설명 | 예시 |
|---|---|---|
| `-DoutputDirectory` | 복사 위치 | `target/jqa-libs` |
| `-DincludeScope` | 포함할 스코프 (runtime은 compile+runtime 포함) | `runtime`, `compile`, `test` |
| `-DexcludeScope` | 제외할 스코프 | `provided` |
| `-DincludeGroupIds` | 특정 그룹만 포함 (콤마 구분) | `com.mycompany,org.apache.commons` |
| `-DexcludeGroupIds` | 특정 그룹 제외 | `org.springframework` |
| `-DincludeArtifactIds` / `-DexcludeArtifactIds` | 아티팩트 단위 포함/제외 | `my-core,my-common` |
| `-DexcludeTransitive` | 직접 의존성만 복사 | `true` |
| `-DprependGroupId` | 파일명에 groupId 붙이기 (동명 JAR 구분) | `true` |
| `-DuseRepositoryLayout` | 로컬 저장소처럼 groupId 폴더 구조로 복사 | `true` |

**용도별 추천 조합**

```bash
# 자사 라이브러리만 깊게 분석 (그래프 크기 대폭 축소)
mvn dependency:copy-dependencies -DoutputDirectory=target/jqa-libs \
    -DincludeGroupIds=com.mycompany

# 모든 라이브러리 (Java 21 마이그레이션 영향도 등 전수 조사)
mvn dependency:copy-dependencies -DoutputDirectory=target/jqa-libs \
    -DincludeScope=runtime -DprependGroupId=true

# provided(예: servlet-api)까지 포함 — WAS 제공 API 사용 현황 분석 시
mvn dependency:copy-dependencies -DoutputDirectory=target/jqa-libs \
    -DincludeScope=compile
```

> `includeScope=compile`은 compile+provided+system을, `runtime`은 compile+runtime을 포함합니다.

### 3.6 멀티모듈 프로젝트

멀티모듈에서는 모듈마다 `target/jqa-libs`가 생기고, 형제 모듈의 JAR도 의존성으로 복사됩니다. 권장 방식:

```bash
# 루트에서 실행: 모든 모듈의 의존 JAR를 루트의 한 폴더로 모으기 (절대경로 사용)
mvn clean install -DskipTests
mvn dependency:copy-dependencies \
    -DoutputDirectory=$(pwd)/target/jqa-libs \
    -DincludeScope=runtime \
    -DexcludeGroupIds=com.mycompany.myapp     # 형제 모듈 JAR 중복 제외 (선택)
```

`.jqassistant.yml` (루트):

```yaml
jqassistant:
  maven:
    use-execution-root-as-project-root: true   # DB/규칙을 루트 모듈 하나로 통합
  scan:
    include:
      files:
        - ${maven.multiModuleProjectDirectory}/target/jqa-libs
```

```bash
mvn jqassistant:scan jqassistant:analyze
mvn jqassistant:effective-configuration     # 실제 적용된 설정 확인
mvn jqassistant:server
```

> 환경에 따라 `include.files`가 모듈마다 반복 스캔될 수 있습니다. 스캔 시간이 비정상적으로 길거나 같은 JAR 노드가 여러 개 생기면 **5장 CLI 방식**(루트에서 한 번에 스캔)을 사용하세요.

### 3.7 분석 전용 Maven 프로파일 (재사용 추천)

평소 빌드에 영향 없이 `mvn verify -Pjqa`로만 돌게 만드는 구성입니다.

```xml
<profiles>
  <profile>
    <id>jqa</id>
    <build>
      <plugins>
        <!-- 1) package 단계에서 의존 JAR 복사 -->
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-dependency-plugin</artifactId>
          <executions>
            <execution>
              <id>copy-libs-for-jqa</id>
              <phase>package</phase>
              <goals><goal>copy-dependencies</goal></goals>
              <configuration>
                <outputDirectory>${project.build.directory}/jqa-libs</outputDirectory>
                <includeScope>runtime</includeScope>
              </configuration>
            </execution>
          </executions>
        </plugin>
        <!-- 2) verify 단계에서 scan + analyze -->
        <plugin>
          <groupId>com.buschmais.jqassistant</groupId>
          <artifactId>jqassistant-maven-plugin</artifactId>
          <version>${jqassistant.version}</version>
          <executions>
            <execution>
              <id>default-cli</id>
              <goals>
                <goal>scan</goal>
                <goal>analyze</goal>
              </goals>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

```bash
mvn clean verify -Pjqa -DskipTests
mvn jqassistant:server
```

### 3.8 Maven goal 정리

| Goal | 설명 |
|---|---|
| `jqassistant:scan` | 프로젝트 산출물 + 설정된 include 파일을 스캔 |
| `jqassistant:analyze` | 규칙 실행, 리포트 생성 |
| `jqassistant:server` | 내장 Neo4j 서버 실행 (기본 http://localhost:7474) |
| `jqassistant:reset` | DB 전체 삭제 |
| `jqassistant:available-rules` | 사용 가능한 규칙 목록 |
| `jqassistant:effective-rules` | 이번 analyze에서 실제 실행될 규칙 목록 |
| `jqassistant:effective-configuration` | 최종 적용된 설정을 YAML로 출력 |
| `jqassistant:available-scopes` | 스캔 시 사용 가능한 scope 목록 |
| `jqassistant:report` | XML 리포트를 HTML로 변환 |

### 3.9 자주 쓰는 `-D` 옵션

| 옵션 | 설명 |
|---|---|
| `-Djqassistant.skip=true` | jQAssistant 전체 건너뛰기 |
| `-Djqassistant.scan.reset=false` | 기존 DB 유지하며 추가 스캔 |
| `-Djqassistant.scan.continue-on-error=true` | 스캔 오류 무시 |
| `-Djqassistant.analyze.concepts=java:VirtualInvokes,java:VirtualDependsOn` | 실행할 concept 지정 |
| `-Djqassistant.analyze.constraints=java:AvoidCyclicPackageDependencies` | 실행할 constraint 지정 |
| `-Djqassistant.analyze.groups=my-rules:Report` | 실행할 group 지정 |
| `-Djqassistant.analyze.report.fail-on-severity=NEVER` | 위반이 있어도 빌드 성공 |
| `-Djqassistant.analyze.rule-parameters.prefix=com.mycompany.` | 규칙 파라미터 전달 |
| `-Djqassistant.server.daemon=true` | 서버를 데몬으로 (Enter 대기 없이 계속 실행) |
| `-Djqassistant.store.uri=bolt://localhost:7687` | 외부 Neo4j 서버 사용 |

### 3.10 산출물 위치

| 항목 | 기본 경로 |
|---|---|
| Neo4j DB | `target/jqassistant/store` |
| XML 리포트 | `target/jqassistant/jqassistant-report.xml` |
| HTML 리포트 | `target/jqassistant/report/asciidoc/index.html` (Asciidoc 규칙 사용 시) |
| CSV 등 추가 리포트 | `target/jqassistant/report/` |

---

## 4. pom.xml 수정 없이 Maven으로 실행하기

고객사 소스라서 pom.xml을 바꿀 수 없거나, 형상관리에 분석용 설정을 남기고 싶지 않을 때 쓰는 방법입니다. Maven은 **pom.xml에 선언되지 않은 플러그인도 좌표(`groupId:artifactId:version:goal`)를 명령행에 직접 지정하면 다운로드해 실행**합니다. jQAssistant 설정은 pom.xml 대신 `-D` 시스템 속성이나 프로젝트 밖의 YAML 파일로 넘깁니다.

### 4.1 기본: 플러그인 전체 좌표로 실행

```bash
# 버전을 변수로 두면 편합니다 (최신 버전은 Maven Central에서 확인)
JQA=com.buschmais.jqassistant:jqassistant-maven-plugin:2.9.1
DEP=org.apache.maven.plugins:maven-dependency-plugin:3.8.1

# 1) 빌드
mvn clean package -DskipTests

# 2) 의존 JAR 수집 (dependency 플러그인도 좌표로 지정)
mvn $DEP:copy-dependencies \
    -DoutputDirectory=target/jqa-libs \
    -DincludeScope=runtime

# 3) 스캔 — 라이브러리 폴더를 -D로 스캔 대상에 추가
mvn $JQA:scan \
    -Djqassistant.scan.include.files[0]=$(pwd)/target/jqa-libs \
    -Djqassistant.scan.continue-on-error=true

# 4) 분석
mvn $JQA:analyze -Djqassistant.analyze.report.fail-on-severity=NEVER

# 5) 서버
mvn $JQA:server
```

- `dependency:copy-dependencies`처럼 접두어만 써도 대부분 동작하지만(Maven 기본 플러그인 그룹), 사내 Maven 버전에 따라 오래된 dependency 플러그인이 잡힐 수 있어 좌표 지정을 권장합니다.
- `scan`, `analyze`, `server`를 각각 실행해도 같은 DB(`target/jqassistant/store`)를 사용하므로 결과가 이어집니다.
- zsh에서는 `[0]`이 glob으로 해석될 수 있으니 `'-Djqassistant.scan.include.files[0]=...'`처럼 따옴표로 감싸세요.

### 4.2 `settings.xml`로 명령 짧게 쓰기

`~/.m2/settings.xml`에 pluginGroup을 등록하면 pom.xml에 선언이 없어도 접두어로 실행할 수 있습니다.

```xml
<settings>
  <pluginGroups>
    <pluginGroup>com.buschmais.jqassistant</pluginGroup>
  </pluginGroups>
</settings>
```

```bash
mvn jqassistant:scan       # 버전 미지정 → 저장소의 최신 릴리스로 해석
mvn jqassistant:2.9.1:scan # ❌ 이 형식은 지원되지 않음
```

> 버전 없이 실행하면 저장소 메타데이터의 최신 릴리스가 쓰이므로 실행 시점마다 버전이 달라질 수 있습니다. **재현성이 중요한 분석(보고서 제출 등)은 4.1처럼 버전을 고정**하세요.

### 4.3 설정 파일을 프로젝트 밖에 두기

`.jqassistant.yml`을 프로젝트에 만들지 않고, 별도 위치의 설정 파일을 지정할 수 있습니다.

`~/jqa/analysis.yml` (예시):

```yaml
jqassistant:
  scan:
    reset: true
    continue-on-error: true
    include:
      files:
        - ${maven.multiModuleProjectDirectory}/target/jqa-libs
  analyze:
    report:
      fail-on-severity: NEVER
  server:
    open-browser: true
```

```bash
mvn $JQA:scan    -Djqassistant.configuration.locations=$HOME/jqa/analysis.yml
mvn $JQA:analyze -Djqassistant.configuration.locations=$HOME/jqa/analysis.yml
mvn $JQA:server  -Djqassistant.configuration.locations=$HOME/jqa/analysis.yml
```

설정 적용 범위별 선택지:

| 방법 | 위치 | 적용 범위 |
|---|---|---|
| `-D` 시스템 속성 | 명령행 | 해당 실행만 |
| `-Djqassistant.configuration.locations=<파일>` | 임의 경로 | 지정한 실행만 |
| `~/.jqassistant.yml` | 사용자 홈 | 모든 프로젝트에 공통 (우선순위 가장 낮음) |
| 환경변수 (`JQASSISTANT_SCAN_CONTINUE_ON_ERROR=true` 등) | 셸/CI | 해당 셸 세션 |

> 커스텀 규칙도 프로젝트 밖에 둘 수 있습니다: `-Djqassistant.analyze.rule.directory=$HOME/jqa/rules`

### 4.4 DB를 `target` 밖에 두기 (선택)

기본 DB 위치는 `target/jqassistant/store`라서 `mvn clean`을 하면 지워집니다. 스캔 결과를 보존하려면 저장 위치를 바꿉니다.

```bash
mvn $JQA:scan   -Djqassistant.store.uri=file://$HOME/jqa/store/myproject
mvn $JQA:server -Djqassistant.store.uri=file://$HOME/jqa/store/myproject
```

> 모든 goal에 같은 `store.uri`를 줘야 같은 DB를 봅니다. 4.3의 설정 파일에 `jqassistant.store.uri`로 넣어두면 편합니다.

### 4.5 멀티모듈 프로젝트

루트에서 좌표 지정 goal을 실행하면 reactor의 모든 모듈에 대해 수행되고, 기본적으로 실행 위치(루트)의 DB 하나에 모입니다.

```bash
mvn clean install -DskipTests
mvn $DEP:copy-dependencies -DoutputDirectory=$(pwd)/target/jqa-libs -DincludeScope=runtime
mvn $JQA:scan \
    -Djqassistant.maven.use-execution-root-as-project-root=true \
    -Djqassistant.scan.include.files[0]=$(pwd)/target/jqa-libs
```

- `include.files`가 모듈마다 반복 스캔되면 시간이 크게 늘어납니다. 이 경우 **5장 CLI 방식**이 더 확실합니다.
- 특정 모듈만 빼려면 `-pl '!module-batch'`처럼 Maven 표준 옵션을 함께 사용하세요.

### 4.6 한 번에 실행하는 스크립트

**macOS / Linux (`jqa-run.sh`)**

```bash
#!/usr/bin/env bash
# 사용법: 프로젝트 루트에서 ./jqa-run.sh [server]
set -euo pipefail

JQA_VER=${JQA_VER:-2.9.1}
JQA=com.buschmais.jqassistant:jqassistant-maven-plugin:$JQA_VER
DEP=org.apache.maven.plugins:maven-dependency-plugin:3.8.1
ROOT=$(pwd)
export MAVEN_OPTS=${MAVEN_OPTS:-"-Xmx4g"}

mvn -q clean install -DskipTests
mvn -q $DEP:copy-dependencies -DoutputDirectory="$ROOT/target/jqa-libs" -DincludeScope=runtime
mvn $JQA:scan \
    "-Djqassistant.scan.include.files[0]=$ROOT/target/jqa-libs" \
    -Djqassistant.scan.continue-on-error=true \
    -Djqassistant.maven.use-execution-root-as-project-root=true
mvn $JQA:analyze \
    -Djqassistant.analyze.concepts=java:VirtualInvokes,java:Deprecated \
    -Djqassistant.analyze.report.fail-on-severity=NEVER

if [[ "${1:-}" == "server" ]]; then
  mvn $JQA:server -Djqassistant.server.open-browser=true
fi
```

**Windows (`jqa-run.cmd`)**

```bat
@echo off
setlocal
set JQA=com.buschmais.jqassistant:jqassistant-maven-plugin:2.9.1
set DEP=org.apache.maven.plugins:maven-dependency-plugin:3.8.1
set ROOT=%CD%
if "%MAVEN_OPTS%"=="" set MAVEN_OPTS=-Xmx4g

call mvn -q clean install -DskipTests || exit /b 1
call mvn -q %DEP%:copy-dependencies -DoutputDirectory="%ROOT%\target\jqa-libs" -DincludeScope=runtime || exit /b 1
call mvn %JQA%:scan "-Djqassistant.scan.include.files[0]=%ROOT%\target\jqa-libs" -Djqassistant.scan.continue-on-error=true -Djqassistant.maven.use-execution-root-as-project-root=true || exit /b 1
call mvn %JQA%:analyze -Djqassistant.analyze.concepts=java:VirtualInvokes,java:Deprecated -Djqassistant.analyze.report.fail-on-severity=NEVER || exit /b 1

if /I "%1"=="server" call mvn %JQA%:server -Djqassistant.server.open-browser=true
endlocal
```

> 스크립트는 프로젝트 밖(예: `~/bin`)에 두고 프로젝트 루트에서 호출하면 소스 트리에 아무 파일도 남지 않습니다. 빌드 산출물(`target/`)만 생깁니다.

### 4.7 방식 비교

| 방식 | pom.xml 수정 | 장점 | 단점 |
|---|---|---|---|
| 3장: pom.xml에 플러그인 선언 | 필요 | 팀 공유·CI 연동 쉬움 | 소스 변경 필요 |
| **4장: 좌표 지정 실행** | **불필요** | 소스 무변경, Maven 빌드 정보(pom, 모듈 구조) 함께 스캔 | 명령이 길어짐 (스크립트로 해결) |
| 5장: CLI | 불필요 | 스캔 대상을 완전히 직접 통제, 멀티모듈 중복 없음 | 별도 배포판 설치, Maven 모듈 정보는 수동 지정 |

---

## 5. CLI로 실행하기

pom.xml을 건드릴 수 없거나, 멀티모듈 전체를 한 번에 스캔할 때 가장 확실한 방법입니다.

### 5.1 설치

- Maven Central에서 `jqassistant-commandline-neo4jv5` 의 `*-distribution.zip` 다운로드 후 압축 해제
- 또는 SDKMAN: `sdk install jqassistant`

### 5.2 실행

```bash
# 1) 빌드 + 의존 JAR 수집
mvn clean package -DskipTests
mvn dependency:copy-dependencies -DoutputDirectory=$(pwd)/target/jqa-libs -DincludeScope=runtime

# 2) 스캔 (콤마로 여러 경로 지정)
jqassistant scan \
  -f java:classpath::target/classes,target/jqa-libs \
  -D jqassistant.scan.reset=true

# 멀티모듈: 각 모듈 classes 경로를 모두 나열
jqassistant scan \
  -f java:classpath::module-a/target/classes,java:classpath::module-b/target/classes,target/jqa-libs

# 3) 분석 / 서버
jqassistant analyze -D jqassistant.analyze.concepts=java:VirtualInvokes
jqassistant server
```

> 구버전 배포판은 스크립트 이름이 `bin/jqassistant.sh`(Linux/macOS), `bin\jqassistant.cmd`(Windows)입니다.

| CLI task | 설명 |
|---|---|
| `scan -f <파일/폴더>` / `scan -u <URL>` | 스캔 (`java:classpath::` 같은 scope 접두어 가능) |
| `analyze` | 규칙 실행 |
| `server` | 내장 Neo4j 서버 |
| `available-rules`, `effective-rules`, `effective-configuration`, `available-scopes` | Maven goal과 동일 |
| `report` | XML → HTML |

---

## 6. 내장 규칙(Concept/Constraint) 적용

Concept은 그래프에 라벨/관계를 추가해 쿼리를 쉽게 만들고, Constraint는 위반을 찾습니다.

### 6.1 유용한 Java 플러그인 Concept

| Concept | 효과 |
|---|---|
| `java:VirtualInvokes` | 인터페이스/상위 클래스 메서드 호출을 실제 구현 메서드 호출로도 연결 (콜 그래프 정확도 향상) |
| `java:VirtualDependsOn` | 위와 같은 원리로 타입 의존 보강 |
| `java:MethodOverrides` | 오버라이드 메서드 간 `OVERRIDES` 관계 추가 |
| `java:InnerType`, `java:AnonymousInnerType` | 내부/익명 클래스에 라벨 부여 |
| `java:TypeAssignableFrom` | 대입 가능 관계(`ASSIGNABLE_FROM`) 추가 |
| `java:Deprecated` | `@Deprecated` 요소에 `:Deprecated` 라벨 |
| `java:Exception`, `java:RuntimeException` | 예외 타입 라벨 |

### 6.2 유용한 Constraint

| Constraint | 검사 내용 |
|---|---|
| `java:AvoidCyclicPackageDependencies` | 패키지 순환 의존 |
| `java:AvoidCyclicArtifactDependencies` | 산출물(JAR/모듈) 간 순환 의존 |

### 6.3 실행 예

```bash
mvn jqassistant:analyze \
  -Djqassistant.analyze.concepts=java:VirtualInvokes,java:MethodOverrides,java:Deprecated \
  -Djqassistant.analyze.constraints=java:AvoidCyclicPackageDependencies,java:AvoidCyclicArtifactDependencies \
  -Djqassistant.analyze.report.fail-on-severity=NEVER

# 전체 규칙 목록 확인
mvn jqassistant:available-rules
```

---

## 7. 결과 탐색과 내보내기

### 7.1 Neo4j 브라우저

`mvn jqassistant:server` 실행 후 http://localhost:7474 접속 (인증 없음, Bolt `bolt://localhost:7687`).

- 결과 테이블 우측 상단 다운로드 버튼 → **CSV/JSON 내보내기**
- 그래프 뷰에서 노드 더블클릭으로 관계 확장

### 7.2 외부 도구 연결

서버를 데몬으로 띄우면 DBeaver, Neo4j Desktop, Python(`neo4j` 드라이버) 등에서 Bolt로 접속할 수 있습니다.

```bash
mvn jqassistant:server -Djqassistant.server.daemon=true
```

### 7.3 성능 팁

`fqn` 조인 쿼리가 느리면 인덱스를 확인·생성합니다 (Neo4j 5 문법).

```cypher
SHOW INDEXES;
CREATE INDEX type_fqn IF NOT EXISTS FOR (t:Type) ON (t.fqn);
```

---

## 8. 분석 쿼리 모음

> **공통 규칙**
> - `'com.mycompany.'` 부분을 자사 패키지 접두어로 바꿔 사용하세요. 접두어 끝에 `.`을 붙여야 오탐이 없습니다.
> - `EXISTS { ... }` 서브쿼리는 Neo4j 5 문법입니다. Neo4j 4에서는 패턴 조건 `AND (a)-[:R]->(b)` 형태로 바꾸세요.
> - 여러 JAR에 같은 클래스가 중복되는 경우를 고려해 `count(DISTINCT ...)`를 사용했습니다.

### 8.1 전체 현황

**스캔된 산출물별 클래스·메서드 수**

```cypher
MATCH (a:Artifact)-[:CONTAINS]->(t:Type)
OPTIONAL MATCH (t)-[:DECLARES]->(m:Method)
RETURN a.fileName AS artifact,
       labels(a) AS kind,
       count(DISTINCT t) AS types,
       count(m) AS methods
ORDER BY types DESC
```

**그래프 전체 노드 라벨 분포**

```cypher
MATCH (n)
UNWIND labels(n) AS label
RETURN label, count(*) AS cnt
ORDER BY cnt DESC
```

### 8.2 라이브러리(JAR) 분석

**JAR ↔ JAR 참조관계 (클래스 의존 집계)**

```cypher
MATCH (a1:Artifact)-[:CONTAINS]->(t1:Type)-[:DEPENDS_ON]->(r:Type)
WHERE NOT EXISTS { (a1)-[:CONTAINS]->(:Type {fqn: r.fqn}) }   // 자기 내부 참조 제외
MATCH (a2:Artifact)-[:CONTAINS]->(t2:Type)
WHERE t2.fqn = r.fqn AND a1 <> a2
RETURN a1.fileName AS fromArtifact,
       a2.fileName AS toArtifact,
       count(DISTINCT t1.fqn + '>' + r.fqn) AS typeRefs,
       count(DISTINCT t1) AS fromClasses,
       count(DISTINCT r.fqn) AS toClasses
ORDER BY typeRefs DESC
```

**애플리케이션 코드가 직접 사용하는 라이브러리와 사용 클래스 수**

```cypher
MATCH (app:Artifact:Directory)-[:CONTAINS]->(t:Type)-[:DEPENDS_ON]->(r:Type)
MATCH (jar:Jar:Archive)-[:CONTAINS]->(lt:Type)
WHERE lt.fqn = r.fqn
RETURN jar.fileName AS library,
       count(DISTINCT r.fqn) AS usedLibClasses,
       count(DISTINCT t) AS appClassesUsingIt
ORDER BY appClassesUsingIt DESC
```

**애플리케이션 코드가 직접 참조하지 않는 JAR (불필요 의존성 후보)**

```cypher
MATCH (app:Artifact:Directory)-[:CONTAINS]->(:Type)-[:DEPENDS_ON]->(r:Type)
WITH collect(DISTINCT r.fqn) AS usedFqns
MATCH (jar:Jar:Archive)
WHERE NOT EXISTS {
  MATCH (jar)-[:CONTAINS]->(t:Type) WHERE t.fqn IN usedFqns
}
RETURN jar.fileName AS notDirectlyUsed
ORDER BY notDirectlyUsed
```

> 다른 라이브러리가 내부적으로 쓰는 전이 의존성, 리플렉션·XML 설정·SPI로만 로딩되는 라이브러리(JDBC 드라이버, 로깅 구현체 등)도 여기 나옵니다. **삭제 후보 목록이지 삭제 대상 확정이 아닙니다.**

**특정 라이브러리를 사용하는 애플리케이션 클래스 목록 (교체 영향도)**

```cypher
WITH 'commons-lang' AS libName
MATCH (jar:Jar:Archive)-[:CONTAINS]->(lt:Type)
WHERE jar.fileName CONTAINS libName
WITH collect(DISTINCT lt.fqn) AS libFqns
MATCH (app:Artifact:Directory)-[:CONTAINS]->(t:Type)-[:DEPENDS_ON]->(r:Type)
WHERE r.fqn IN libFqns
RETURN t.fqn AS appClass, collect(DISTINCT r.fqn) AS usedLibClasses
ORDER BY appClass
```

**여러 JAR에 중복 존재하는 클래스 (클래스패스 충돌 위험)**

```cypher
MATCH (a:Artifact)-[:CONTAINS]->(t:Type)
WHERE NOT t.fqn STARTS WITH 'module-info'
WITH t.fqn AS fqn, collect(DISTINCT a.fileName) AS artifacts
WHERE size(artifacts) > 1
RETURN fqn, artifacts, size(artifacts) AS cnt
ORDER BY cnt DESC, fqn
LIMIT 500
```

**어느 스캔 산출물에도 없는 참조 타입 (누락 의존성 / JDK 제공 API)**

```cypher
MATCH (a:Artifact)-[:REQUIRES]->(r:Type)
WHERE NOT r.fqn STARTS WITH 'java.'
  AND NOT EXISTS { MATCH (:Artifact)-[:CONTAINS]->(t:Type) WHERE t.fqn = r.fqn }
RETURN r.fqn AS missingType, collect(DISTINCT a.fileName) AS requiredBy
ORDER BY missingType
```

### 8.3 패키지 분석

**패키지별 소속 JAR (특정 접두어 제외)**

```cypher
WITH ['java.', 'javax.', 'sun.', 'com.sun.', 'org.w3c.'] AS excludes
MATCH (jar:Jar:Archive)-[:CONTAINS]->(p:Package)
WHERE NONE(prefix IN excludes WHERE p.fqn STARTS WITH prefix)
  AND EXISTS { (p)-[:CONTAINS]->(:Type) }     // 클래스가 실제로 있는 패키지만
RETURN p.fqn AS package, collect(DISTINCT jar.fileName) AS jars
ORDER BY package
```

**특정 접두어 패키지만 조회 (포함 조건)**

```cypher
WITH ['com.mycompany.', 'org.apache.commons.'] AS includes
MATCH (jar:Jar:Archive)-[:CONTAINS]->(p:Package)
WHERE ANY(prefix IN includes WHERE p.fqn STARTS WITH prefix)
  AND EXISTS { (p)-[:CONTAINS]->(:Type) }
RETURN p.fqn AS package, collect(DISTINCT jar.fileName) AS jars
ORDER BY package
```

**Split Package (같은 패키지가 여러 JAR에 분산 — JPMS/Java 9+ 문제 요인)**

```cypher
WITH ['java.', 'javax.', 'sun.'] AS excludes
MATCH (a:Artifact)-[:CONTAINS]->(p:Package)
WHERE NONE(prefix IN excludes WHERE p.fqn STARTS WITH prefix)
  AND EXISTS { (p)-[:CONTAINS]->(:Type) }
WITH p.fqn AS package, collect(DISTINCT a.fileName) AS artifacts
WHERE size(artifacts) > 1
RETURN package, artifacts, size(artifacts) AS cnt
ORDER BY cnt DESC, package
```

**패키지·JAR별 클래스 수**

```cypher
WITH ['java.', 'javax.', 'sun.'] AS excludes
MATCH (jar:Jar:Archive)-[:CONTAINS]->(p:Package)-[:CONTAINS]->(t:Type)
WHERE NONE(prefix IN excludes WHERE p.fqn STARTS WITH prefix)
RETURN p.fqn AS package, jar.fileName AS jar, count(t) AS classCount
ORDER BY package, jar
```

### 8.4 패키지 결합도 (N레벨 집계)

클래스의 `fqn`에서 패키지를 잘라 N레벨(기본 3)로 묶어 집계합니다. `fqn` 기반이라 참조용 노드도 누락되지 않습니다. 예: `com.mycompany.order.service.impl.X` → 3레벨 `com.mycompany.order`.

**① N레벨 패키지 간 참조관계**

```cypher
WITH 'com.mycompany.' AS prefix, 3 AS level
MATCH (t1:Type)-[:DEPENDS_ON]->(t2:Type)
WHERE t1.fqn STARTS WITH prefix AND t2.fqn STARTS WITH prefix
WITH level, t1, t2,
     split(t1.fqn, '.')[0..-1] AS p1,          // 클래스명 제거 → 패키지
     split(t2.fqn, '.')[0..-1] AS p2
WITH t1, t2, p1[0..level] AS s, p2[0..level] AS d
WITH t1, t2,
     reduce(x = head(s), y IN tail(s) | x + '.' + y) AS fromPkg,
     reduce(x = head(d), y IN tail(d) | x + '.' + y) AS toPkg
WHERE fromPkg <> toPkg
RETURN fromPkg, toPkg,
       count(DISTINCT t1.fqn + '>' + t2.fqn) AS dependencies,  // 클래스→클래스 참조 건수
       count(DISTINCT t1.fqn) AS fromClasses,
       count(DISTINCT t2.fqn) AS toClasses
ORDER BY dependencies DESC
```

**② 패키지별 결합도 지표 (Ca / Ce / 불안정도 I)**

```cypher
WITH 'com.mycompany.' AS prefix, 3 AS level
MATCH (t1:Type)-[:DEPENDS_ON]->(t2:Type)
WHERE t1.fqn STARTS WITH prefix AND t2.fqn STARTS WITH prefix
WITH level, split(t1.fqn, '.')[0..-1] AS p1, split(t2.fqn, '.')[0..-1] AS p2
WITH p1[0..level] AS s, p2[0..level] AS d
WITH reduce(x = head(s), y IN tail(s) | x + '.' + y) AS fromPkg,
     reduce(x = head(d), y IN tail(d) | x + '.' + y) AS toPkg
WHERE fromPkg <> toPkg
WITH DISTINCT fromPkg, toPkg
WITH collect({f: fromPkg, t: toPkg}) AS edges
UNWIND edges AS e
WITH edges, collect(DISTINCT e.f) + collect(DISTINCT e.t) AS allPkgs
UNWIND allPkgs AS pkg
WITH DISTINCT pkg, edges
WITH pkg,
     size([e IN edges WHERE e.t = pkg]) AS Ca,    // 구심 결합: 나를 참조하는 패키지 수
     size([e IN edges WHERE e.f = pkg]) AS Ce     // 원심 결합: 내가 참조하는 패키지 수
RETURN pkg, Ca, Ce,
       round(toFloat(Ce) / (Ca + Ce) * 100) / 100 AS instability
ORDER BY instability DESC, Ce DESC
```

| 지표 | 해석 |
|---|---|
| I ≈ 0 | 많이 의존받는 안정 패키지 → 공통/도메인 계층에 적합 |
| I ≈ 1 | 남을 많이 참조하는 불안정 패키지 → 웹/어댑터 계층에 적합 |
| 점검 신호 | 공통 패키지인데 I가 높음, 컨트롤러 계층인데 Ca가 높음 |

**③ N레벨 패키지 간 순환 참조 (양방향)**

```cypher
WITH 'com.mycompany.' AS prefix, 3 AS level
MATCH (t1:Type)-[:DEPENDS_ON]->(t2:Type)
WHERE t1.fqn STARTS WITH prefix AND t2.fqn STARTS WITH prefix
WITH level, t1, t2, split(t1.fqn, '.')[0..-1] AS p1, split(t2.fqn, '.')[0..-1] AS p2
WITH t1, t2, p1[0..level] AS s, p2[0..level] AS d
WITH t1, t2,
     reduce(x = head(s), y IN tail(s) | x + '.' + y) AS a,
     reduce(x = head(d), y IN tail(d) | x + '.' + y) AS b
WHERE a <> b
WITH a, b,
     count(DISTINCT t1.fqn + '>' + t2.fqn) AS cnt,
     collect(DISTINCT t1.fqn + ' → ' + t2.fqn)[0..5] AS samples
WITH collect({a: a, b: b, cnt: cnt, samples: samples}) AS edges
UNWIND edges AS e1
UNWIND edges AS e2
WITH e1, e2 WHERE e1.a = e2.b AND e1.b = e2.a AND e1.a < e1.b
RETURN e1.a AS pkgA, e1.b AS pkgB,
       e1.cnt AS aToB, e2.cnt AS bToA,
       e1.samples AS sampleAtoB, e2.samples AS sampleBtoA
ORDER BY aToB + bToA DESC
```

> 건수가 적은 방향이 보통 끊기 쉬운 쪽입니다. 3개 이상 패키지가 얽힌 순환은 6.2의 `java:AvoidCyclicPackageDependencies`로 확인하세요.

**④ (선택) 패키지 의존 그래프 시각화 — APOC 필요**

```cypher
WITH 'com.mycompany.' AS prefix, 3 AS level
MATCH (t1:Type)-[:DEPENDS_ON]->(t2:Type)
WHERE t1.fqn STARTS WITH prefix AND t2.fqn STARTS WITH prefix
WITH level, split(t1.fqn, '.')[0..-1] AS p1, split(t2.fqn, '.')[0..-1] AS p2
WITH p1[0..level] AS s, p2[0..level] AS d
WITH reduce(x = head(s), y IN tail(s) | x + '.' + y) AS a,
     reduce(x = head(d), y IN tail(d) | x + '.' + y) AS b
WHERE a <> b
WITH a, b, count(*) AS cnt
CALL apoc.create.vNode(['Pkg'], {name: a}) YIELD node AS na
CALL apoc.create.vNode(['Pkg'], {name: b}) YIELD node AS nb
RETURN na, nb, apoc.create.vRelationship(na, 'DEPENDS_ON', {count: cnt}, nb) AS rel
```

APOC가 없다면 ①의 결과를 CSV로 내려받아 Gephi·draw.io 등에서 그리면 됩니다.

### 8.5 클래스 수준 결합도

**클래스별 Fan-out(참조하는 클래스 수) / Fan-in(참조받는 클래스 수) Top 30**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t1:Type)-[:DEPENDS_ON]->(t2:Type)
WHERE t1.fqn STARTS WITH prefix AND t2.fqn STARTS WITH prefix
  AND split(t1.fqn, '$')[0] <> split(t2.fqn, '$')[0]        // 자기 내부 클래스 제외
WITH split(t1.fqn, '$')[0] AS src, split(t2.fqn, '$')[0] AS dst
WITH DISTINCT src, dst
WITH collect({s: src, d: dst}) AS edges
UNWIND edges AS e
WITH edges, collect(DISTINCT e.s) + collect(DISTINCT e.d) AS all
UNWIND all AS cls
WITH DISTINCT cls, edges
RETURN cls,
       size([e IN edges WHERE e.s = cls]) AS fanOut,
       size([e IN edges WHERE e.d = cls]) AS fanIn
ORDER BY fanOut + fanIn DESC
LIMIT 30
```

**외부 라이브러리 의존이 많은 클래스 (라이브러리 교체 시 고위험)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DEPENDS_ON]->(r:Type)
WHERE t.fqn STARTS WITH prefix
  AND NOT r.fqn STARTS WITH prefix
  AND NOT r.fqn STARTS WITH 'java.'
RETURN t.fqn AS cls, count(DISTINCT r.fqn) AS externalDeps,
       collect(DISTINCT split(r.fqn, '.')[0] + '.' + split(r.fqn, '.')[1])[0..10] AS externalRoots
ORDER BY externalDeps DESC
LIMIT 30
```

**인터페이스 구현체가 하나뿐인 경우 (과잉 추상화 후보)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (c:Type:Class)-[:IMPLEMENTS]->(i:Type)
WHERE i.fqn STARTS WITH prefix
WITH i.fqn AS iface, collect(DISTINCT c.fqn) AS impls
WHERE size(impls) = 1
RETURN iface, impls[0] AS onlyImpl
ORDER BY iface
```

**상속 깊이 (DIT) Top 20**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH path = (c:Type:Class)-[:EXTENDS*1..10]->(:Type)
WHERE c.fqn STARTS WITH prefix
RETURN c.fqn AS cls, max(length(path)) AS depth
ORDER BY depth DESC
LIMIT 20
```

> 모듈/JAR 경계를 넘는 상속은 참조용 노드에서 끊기므로 실제보다 얕게 나올 수 있습니다.

### 8.6 메서드 호출·필드 접근

**특정 클래스(라이브러리 포함)의 메서드를 호출하는 곳 — 라인번호 포함**

```cypher
MATCH (caller:Type)-[:DECLARES]->(cm:Method)-[i:INVOKES]->(m:Method)<-[:DECLARES]-(target:Type)
WHERE target.fqn = 'org.apache.commons.lang.StringUtils'
RETURN caller.fqn AS callerClass, cm.name AS callerMethod,
       m.signature AS calledMethod, i.lineNumber AS line
ORDER BY callerClass, line
```

**특정 메서드에서 시작하는 호출 그래프 (깊이 3)**

```cypher
MATCH (t:Type {fqn: 'com.mycompany.order.service.OrderService'})-[:DECLARES]->(m:Method {name: 'createOrder'})
MATCH path = (m)-[:INVOKES*1..3]->(:Method)
RETURN path
LIMIT 200
```

> 인터페이스 호출까지 구현체로 따라가려면 먼저 `java:VirtualInvokes` concept를 적용하세요.

**특정 메서드를 호출하는 쪽 역추적 (깊이 3)**

```cypher
MATCH (t:Type {fqn: 'com.mycompany.common.util.DateUtil'})-[:DECLARES]->(m:Method {name: 'parse'})
MATCH path = (caller:Method)-[:INVOKES*1..3]->(m)
RETURN path
LIMIT 200
```

**리플렉션 사용 지점 (모듈 시스템 / Java 17+ 강한 캡슐화 영향)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(cm:Method)-[i:INVOKES]->(m:Method)<-[:DECLARES]-(rt:Type)
WHERE t.fqn STARTS WITH prefix
  AND ( (rt.fqn = 'java.lang.Class' AND m.name IN ['forName', 'newInstance', 'getDeclaredField', 'getDeclaredMethod'])
     OR (rt.fqn STARTS WITH 'java.lang.reflect.' AND m.name IN ['invoke', 'setAccessible', 'newInstance']) )
RETURN t.fqn AS cls, cm.name AS method, rt.fqn + '.' + m.name AS reflectiveCall, i.lineNumber AS line
ORDER BY cls
```

**static 가변 필드 (스레드 안전성 위험)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(f:Field)
WHERE t.fqn STARTS WITH prefix
  AND f.static = true AND coalesce(f.final, false) = false
  AND NOT coalesce(f.synthetic, false)
OPTIONAL MATCH (m:Method)-[:WRITES]->(f)
RETURN t.fqn AS cls, f.signature AS field, count(m) AS writerMethods
ORDER BY writerMethods DESC
```

**포괄적 예외 catch (`Exception`, `Throwable`)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(m:Method)-[c:CATCHES]->(e:Type)
WHERE t.fqn STARTS WITH prefix
  AND e.fqn IN ['java.lang.Exception', 'java.lang.Throwable', 'java.lang.RuntimeException']
RETURN t.fqn AS cls, m.name AS method, e.fqn AS caught, c.firstLineNumber AS line
ORDER BY cls
```

### 8.7 코드 품질 지표

**순환 복잡도(Cyclomatic Complexity) 상위 메서드**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(m:Method)
WHERE t.fqn STARTS WITH prefix AND m.cyclomaticComplexity IS NOT NULL
RETURN t.fqn AS cls, m.name AS method,
       m.cyclomaticComplexity AS complexity,
       m.effectiveLineCount AS loc
ORDER BY complexity DESC
LIMIT 30
```

**거대 클래스 (메서드/필드 수/총 LOC)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type:Class)
WHERE t.fqn STARTS WITH prefix
OPTIONAL MATCH (t)-[:DECLARES]->(m:Method)
WITH t, count(m) AS methods, sum(coalesce(m.effectiveLineCount, 0)) AS loc,
     sum(coalesce(m.cyclomaticComplexity, 0)) AS wmc
OPTIONAL MATCH (t)-[:DECLARES]->(f:Field)
RETURN t.fqn AS cls, methods, count(f) AS fields, loc, wmc AS weightedComplexity
ORDER BY loc DESC
LIMIT 30
```

**패키지별 규모·복잡도 요약 (3레벨)**

```cypher
WITH 'com.mycompany.' AS prefix, 3 AS level
MATCH (t:Type)-[:DECLARES]->(m:Method)
WHERE t.fqn STARTS WITH prefix
WITH level, t, m, split(t.fqn, '.')[0..-1] AS p
WITH t, m, p[0..level] AS s
WITH reduce(x = head(s), y IN tail(s) | x + '.' + y) AS pkg, t, m
RETURN pkg,
       count(DISTINCT t) AS classes,
       count(m) AS methods,
       sum(coalesce(m.effectiveLineCount, 0)) AS loc,
       round(avg(coalesce(m.cyclomaticComplexity, 0)) * 100) / 100 AS avgComplexity
ORDER BY loc DESC
```

### 8.8 Java 버전 마이그레이션 (Java 1.5 → 21 등)

**JAR별 바이트코드 버전 분포**

```cypher
MATCH (a:Artifact)-[:CONTAINS]->(t:Type)
WHERE t.byteCodeVersion IS NOT NULL
RETURN a.fileName AS artifact,
       min(t.byteCodeVersion) AS minVer,
       max(t.byteCodeVersion) AS maxVer,
       count(t) AS classes
ORDER BY maxVer DESC, artifact
```

| byteCodeVersion | Java |
|---|---|
| 49 | 5 |
| 50 | 6 |
| 51 | 7 |
| 52 | 8 |
| 55 | 11 |
| 61 | 17 |
| 65 | 21 |

> Java 21 런타임으로 옮길 때 `maxVer`가 65를 넘는 JAR는 실행 불가이고, 매우 오래된 JAR는 내부 API 사용 여부를 아래 쿼리로 확인해야 합니다.

**JDK에서 제거되었거나 내부 API인 패키지 사용 현황**

```cypher
WITH [
  'javax.xml.bind.', 'javax.xml.ws.', 'javax.xml.soap.', 'javax.jws.',   // JAXB/JAX-WS: Java 11에서 제거
  'javax.activation.', 'org.omg.', 'javax.rmi.CORBA.',                   // JAF/CORBA: Java 11에서 제거
  'sun.misc.BASE64', 'sun.misc.', 'sun.reflect.', 'sun.security.',       // 내부 API
  'com.sun.image.codec.', 'com.sun.org.apache.'
] AS removed
MATCH (a:Artifact)-[:CONTAINS]->(t:Type)-[:DEPENDS_ON]->(r:Type)
WHERE ANY(p IN removed WHERE r.fqn STARTS WITH p)
  AND NOT EXISTS { MATCH (:Artifact)-[:CONTAINS]->(x:Type) WHERE x.fqn = r.fqn }  // 별도 JAR로 제공되는 경우 제외
RETURN a.fileName AS artifact, t.fqn AS cls, collect(DISTINCT r.fqn) AS removedApis
ORDER BY artifact, cls
```

> `javax.annotation.*`(`@PostConstruct` 등)도 Java 11에서 제거되었지만 별도 JAR로 흔히 제공되므로, 위 쿼리의 `NOT EXISTS` 조건이 이를 걸러냅니다.

**javax → jakarta 전환 대상 (Spring 6 / Spring Boot 3 이전 시)**

```cypher
WITH 'com.mycompany.' AS prefix,
     ['javax.servlet.', 'javax.persistence.', 'javax.validation.', 'javax.transaction.',
      'javax.annotation.', 'javax.inject.', 'javax.ws.rs.', 'javax.mail.', 'javax.jms.'] AS jee
MATCH (t:Type)-[:DEPENDS_ON]->(r:Type)
WHERE t.fqn STARTS WITH prefix AND ANY(p IN jee WHERE r.fqn STARTS WITH p)
WITH [p IN jee WHERE r.fqn STARTS WITH p][0] AS apiPackage, t
RETURN apiPackage, count(DISTINCT t) AS affectedClasses,
       collect(DISTINCT t.fqn)[0..10] AS samples
ORDER BY affectedClasses DESC
```

**제거·위험 메서드 호출 (`Thread.stop` 등, `finalize` 오버라이드)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(cm:Method)-[i:INVOKES]->(m:Method)<-[:DECLARES]-(rt:Type)
WHERE t.fqn STARTS WITH prefix
  AND ( (rt.fqn = 'java.lang.Thread' AND m.name IN ['stop', 'suspend', 'resume', 'countStackFrames'])
     OR (rt.fqn = 'java.lang.Runtime' AND m.name = 'runFinalizersOnExit')
     OR (rt.fqn = 'java.lang.System' AND m.name IN ['runFinalizersOnExit', 'getSecurityManager', 'setSecurityManager']) )
RETURN t.fqn AS cls, cm.name AS method, rt.fqn + '.' + m.name AS call, i.lineNumber AS line
UNION
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(m:Method {name: 'finalize'})
WHERE t.fqn STARTS WITH prefix AND m.signature = 'void finalize()'
RETURN t.fqn AS cls, 'finalize' AS method, 'overrides Object.finalize()' AS call, m.firstLineNumber AS line
```

**Deprecated API 호출 (`java:Deprecated` concept 적용 후)**

```cypher
// 먼저: mvn jqassistant:analyze -Djqassistant.analyze.concepts=java:Deprecated
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(cm:Method)-[i:INVOKES]->(m:Method)<-[:DECLARES]-(ct:Type)
WHERE t.fqn STARTS WITH prefix AND NOT ct.fqn STARTS WITH prefix
MATCH (dt:Type)-[:DECLARES]->(dm:Method:Deprecated)
WHERE dt.fqn = ct.fqn AND dm.signature = m.signature
RETURN t.fqn AS cls, cm.name AS method, ct.fqn + '#' + m.signature AS deprecatedApi, i.lineNumber AS line
ORDER BY cls
```

> 스캔된 라이브러리 JAR의 `@Deprecated`만 잡힙니다 (JDK 클래스는 스캔하지 않으므로 대상 아님).

### 8.9 Spring 계층 구조 검사

**스테레오타입별 클래스 수**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:ANNOTATED_BY]->()-[:OF_TYPE]->(a:Type)
WHERE t.fqn STARTS WITH prefix
  AND a.fqn STARTS WITH 'org.springframework.'
RETURN a.fqn AS annotation, count(DISTINCT t) AS classes
ORDER BY classes DESC
```

**계층 위반: Controller가 DAO/Repository/Mapper를 직접 참조**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (c:Type)-[:ANNOTATED_BY]->()-[:OF_TYPE]->(ca:Type)
WHERE c.fqn STARTS WITH prefix
  AND ca.fqn IN ['org.springframework.stereotype.Controller',
                 'org.springframework.web.bind.annotation.RestController']
MATCH (c)-[:DEPENDS_ON]->(d:Type)
WHERE d.fqn STARTS WITH prefix
  AND (d.fqn ENDS WITH 'Dao' OR d.fqn ENDS WITH 'DAO' OR d.fqn ENDS WITH 'Repository' OR d.fqn ENDS WITH 'Mapper')
RETURN c.fqn AS controller, collect(DISTINCT d.fqn) AS directDataAccess
ORDER BY controller
```

**Service 간 상호 참조 (순환 주입 위험)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (s1:Type)-[:ANNOTATED_BY]->()-[:OF_TYPE]->(:Type {fqn: 'org.springframework.stereotype.Service'})
WHERE s1.fqn STARTS WITH prefix
MATCH (s1)-[:DEPENDS_ON]->(r:Type)
MATCH (s2:Type)-[:ANNOTATED_BY]->()-[:OF_TYPE]->(:Type {fqn: 'org.springframework.stereotype.Service'})
WHERE s2.fqn = r.fqn AND s1.fqn < s2.fqn
  AND EXISTS { MATCH (s2)-[:DEPENDS_ON]->(back:Type) WHERE back.fqn = s1.fqn }
RETURN s1.fqn AS serviceA, s2.fqn AS serviceB
```

> 레거시 XML 설정 기반 프로젝트는 어노테이션이 없으므로 이름 규칙(`*Controller`, `*Service`, `*Dao`)으로 조건을 바꿔 사용하세요.

### 8.10 데드코드 후보

**어디서도 참조되지 않는 클래스**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (o:Type)-[:DEPENDS_ON]->(r:Type)
WHERE r.fqn STARTS WITH prefix
  AND split(o.fqn, '$')[0] <> split(r.fqn, '$')[0]
WITH prefix, collect(DISTINCT split(r.fqn, '$')[0]) AS used
MATCH (t:Type:Class)
WHERE t.fqn STARTS WITH prefix
  AND NOT t.fqn CONTAINS '$'
  AND NOT t.fqn IN used
RETURN t.fqn AS unreferencedClass
ORDER BY unreferencedClass
```

> Spring 빈, 컨트롤러, 서블릿/필터, 스케줄러 진입점, XML·JSP·리플렉션으로만 참조되는 클래스, `main` 클래스도 결과에 포함됩니다. **후보 목록으로만 사용**하세요.

**호출되지 않는 private 메서드 (정확도 높음)**

```cypher
WITH 'com.mycompany.' AS prefix
MATCH (t:Type)-[:DECLARES]->(m:Method)
WHERE t.fqn STARTS WITH prefix
  AND m.visibility = 'private'
  AND NOT m:Constructor
  AND NOT coalesce(m.synthetic, false)
  AND NOT m.name IN ['readObject', 'writeObject', 'readResolve', 'writeReplace']
  AND NOT EXISTS { (:Method)-[:INVOKES]->(m) }
RETURN t.fqn AS cls, m.signature AS unusedPrivateMethod
ORDER BY cls
```

---

## 9. 커스텀 규칙 파일로 자동화

자주 쓰는 쿼리를 규칙으로 저장하면 `analyze` 때마다 CSV 리포트가 자동 생성됩니다.

`jqassistant/analysis-rules.xml` (프로젝트 루트):

```xml
<jqassistant-rules xmlns="http://schema.jqassistant.org/rule/v2.9"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://schema.jqassistant.org/rule/v2.9 https://jqassistant.github.io/jqassistant/current/schema/jqassistant-rule-v2.9.xsd">

    <group id="analysis:Report">
        <includeConcept refId="analysis:*"/>
        <includeConstraint refId="java:AvoidCyclicPackageDependencies"/>
    </group>

    <concept id="analysis:ArtifactDependencies" severity="info">
        <description>JAR/모듈 간 참조관계</description>
        <cypher><![CDATA[
            MATCH (a1:Artifact)-[:CONTAINS]->(t1:Type)-[:DEPENDS_ON]->(r:Type)
            WHERE NOT EXISTS { (a1)-[:CONTAINS]->(:Type {fqn: r.fqn}) }
            MATCH (a2:Artifact)-[:CONTAINS]->(t2:Type)
            WHERE t2.fqn = r.fqn AND a1 <> a2
            RETURN a1.fileName AS fromArtifact, a2.fileName AS toArtifact,
                   count(DISTINCT t1.fqn + '>' + r.fqn) AS typeRefs
            ORDER BY typeRefs DESC
        ]]></cypher>
        <report reportType="csv"/>
    </concept>

    <concept id="analysis:PackageCoupling" severity="info">
        <requiresParameter name="prefix" type="String" defaultValue="com.mycompany."/>
        <requiresParameter name="level" type="int" defaultValue="3"/>
        <description>N레벨 패키지 간 참조관계</description>
        <cypher><![CDATA[
            MATCH (t1:Type)-[:DEPENDS_ON]->(t2:Type)
            WHERE t1.fqn STARTS WITH $prefix AND t2.fqn STARTS WITH $prefix
            WITH t1, t2, split(t1.fqn, '.')[0..-1] AS p1, split(t2.fqn, '.')[0..-1] AS p2
            WITH t1, t2, p1[0..$level] AS s, p2[0..$level] AS d
            WITH t1, t2,
                 reduce(x = head(s), y IN tail(s) | x + '.' + y) AS fromPkg,
                 reduce(x = head(d), y IN tail(d) | x + '.' + y) AS toPkg
            WHERE fromPkg <> toPkg
            RETURN fromPkg, toPkg, count(DISTINCT t1.fqn + '>' + t2.fqn) AS dependencies
            ORDER BY dependencies DESC
        ]]></cypher>
        <report reportType="csv"/>
    </concept>

</jqassistant-rules>
```

실행:

```bash
mvn jqassistant:analyze \
  -Djqassistant.analyze.groups=analysis:Report \
  -Djqassistant.analyze.rule-parameters.prefix=com.mycompany. \
  -Djqassistant.analyze.rule-parameters.level=3 \
  -Djqassistant.analyze.report.fail-on-severity=NEVER

# 결과: target/jqassistant/report/ 아래 CSV 파일
```

> Concept은 결과가 0건이면 경고가 출력됩니다(정상 동작). 위 쿼리를 다른 8장 쿼리로 바꿔 넣어 원하는 리포트를 늘리면 됩니다.

---

## 10. 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| JAR 간 관계 쿼리 결과가 거의 없음 | 노드로 조인했기 때문. **`fqn`으로 조인** (1.3절 참고) |
| 라이브러리 클래스가 그래프에 없음 | `copy-dependencies` 미실행 또는 `scan.include.files` 경로 오류. `mvn jqassistant:effective-configuration`으로 확인 |
| `OutOfMemoryError` | `MAVEN_OPTS` / `JQASSISTANT_OPTS`에 `-Xmx4g` 이상, 또는 `-DincludeGroupIds`로 스캔 범위 축소 |
| 스캔 중 특정 클래스 파일 오류로 중단 | `-Djqassistant.scan.continue-on-error=true` |
| 같은 JAR가 여러 번 스캔됨 (멀티모듈) | 루트 한 곳에서 CLI로 스캔하거나 `use-execution-root-as-project-root: true` 설정 |
| `analyze`에서 빌드 실패 | `-Djqassistant.analyze.report.fail-on-severity=NEVER` |
| 서버 포트 충돌 (7474/7687) | `-Djqassistant.store.embedded.http-port=17474 -Djqassistant.store.embedded.bolt-port=17687` |
| `EXISTS { }` 문법 오류 | Neo4j 4 배포판 사용 중. 패턴 조건 `AND (a)-[:R]->(b)`로 변경하거나 Neo4j 5 배포판 사용 |
| `apoc.*` 함수 없음 | `jqassistant.store.embedded.apoc-enabled: true` 설정 확인 |
| 이전 스캔 결과가 섞임 | `mvn jqassistant:reset` 후 재스캔, 또는 `scan.reset: true` |
| `No plugin found for prefix 'jqassistant'` | pom.xml에 선언이 없고 `settings.xml`의 pluginGroup도 없음. 4.1처럼 전체 좌표로 실행 |
| `mvn clean` 후 스캔 결과가 사라짐 | DB가 `target` 아래에 있기 때문. 4.4처럼 `jqassistant.store.uri` 변경 |
| 쿼리가 매우 느림 | `CREATE INDEX type_fqn IF NOT EXISTS FOR (t:Type) ON (t.fqn)`, 조회 범위를 `prefix`로 좁히기, `LIMIT` 사용 |

---

## 11. 참고 자료

- jQAssistant 사용자 매뉴얼 (최신): https://jqassistant.github.io/jqassistant/current/
- jQAssistant GitHub: https://github.com/jqassistant
- Maven Dependency Plugin `copy-dependencies`: https://maven.apache.org/plugins/maven-dependency-plugin/copy-dependencies-mojo.html
- Neo4j Cypher Manual: https://neo4j.com/docs/cypher-manual/current/
