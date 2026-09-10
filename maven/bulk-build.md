# 복수 Maven 프로젝트 소스 일괄 컴파일 & JAR 생성 가이드

> 대상 구조 예시
>
> ```
> pff/
> ├── pff-a/
> │   ├── pff-a-core/src/main/java
> │   └── pff-a-web/src/main/java
> ├── pff-b/src/main/java
> ├── pff-c/
> │   └── pff-c-api/src/main/java
> └── lib/                     ← 로컬 JAR (있는 경우)
> ```
>
> 목표: 위 모든 `src/main/java` 를 **하나의 컴파일 단위로 합쳐서** 컴파일하고, 결과를 단일 JAR 로 묶는다.

---

## 목차

1. [먼저 정할 것 — Aggregator vs 통합 컴파일](#1-먼저-정할-것--aggregator-vs-통합-컴파일)
2. [사전 조사 (건너뛰면 반드시 후회함)](#2-사전-조사-건너뛰면-반드시-후회함)
3. [방법 A — Maven 단일 모듈 통합 컴파일](#3-방법-a--maven-단일-모듈-통합-컴파일)
4. [방법 A 확장 — JAR 생성 4가지](#4-방법-a-확장--jar-생성-4가지)
5. [방법 B — 소스 루트 자동 스캔](#5-방법-b--소스-루트-자동-스캔)
6. [의존성 처리 전략](#6-의존성-처리-전략)
7. [방법 C — javac 직접 실행 (경량)](#7-방법-c--javac-직접-실행-경량)
8. [방법 C 확장 — JAR 생성 상세](#8-방법-c-확장--jar-생성-상세)
9. [완성형 빌드 스크립트](#9-완성형-빌드-스크립트)
10. [Windows / PowerShell 버전](#10-windows--powershell-버전)
11. [트러블슈팅](#11-트러블슈팅)
12. [결과 검증 체크리스트](#12-결과-검증-체크리스트)
13. [방법 선택 기준](#13-방법-선택-기준)

---

## 1. 먼저 정할 것 — Aggregator vs 통합 컴파일

두 가지는 완전히 다른 작업입니다. 잘못 고르면 처음부터 다시 해야 합니다.

| | Aggregator (`<modules>`) | 통합 컴파일 (이 문서의 주제) |
|---|---|---|
| POM `packaging` | `pom` | **`jar`** |
| 하위 프로젝트 취급 | 각각 독립 모듈로 순차 빌드 | 소스 파일 더미로 취급 |
| 산출물 | 프로젝트별 JAR N개 | **단일 JAR 1개** |
| 프로젝트 간 참조 | `<dependency>` 선언 필요 | 자동 해결 (같은 컴파일 단위) |
| 각 프로젝트 pom.xml | 반드시 필요 | **불필요** (있어도 무시됨) |
| 주 용도 | 정상적인 릴리스 빌드 | 마이그레이션 영향도 분석, 정적 분석, 전수 컴파일 검증 |

> **핵심**: 통합 컴파일 POM 에는 `<modules>` 를 **절대 넣지 마세요.** 넣는 순간 리액터가 각 프로젝트를 따로 빌드해 버려서 "합쳐서 한 번에"가 성립하지 않습니다.

통합 컴파일이 유리한 상황:

- Java 21 마이그레이션 전 **전체 컴파일 에러 전수 조사**
- 프로젝트 경계를 넘나드는 순환 참조 파악
- OpenRewrite / EMT4J 등 도구에 **타입 정보가 붙은** 단일 트리를 물릴 때
- 각 프로젝트 POM 이 깨져 있어(Maven 2.2.1 시절 등) 정상 빌드가 불가능할 때

---

## 2. 사전 조사 (건너뛰면 반드시 후회함)

### 2-1. 소스 루트 목록 뽑기

```bash
cd pff
find . -type d -path '*/src/main/java' -not -path '*/target/*' | sort
```

`<source>` 태그 형태로 바로 붙여넣을 수 있게 변환:

```bash
find . -type d -path '*/src/main/java' -not -path '*/target/*' | sort \
  | sed 's|^\./|        <source>${project.basedir}/|; s|$|</source>|'
```

### 2-2. 중복 FQCN 검사 — **가장 중요**

SI 코드베이스는 `com.company.common.StringUtil` 같은 유틸이 프로젝트마다 복붙돼 있는 경우가 대단히 흔합니다. 동일 FQCN 이 두 개 이상이면 javac 가 즉시 실패합니다.

```bash
find . -path '*/src/main/java/*.java' -not -path '*/target/*' \
  | sed 's|.*/src/main/java/||' | sort | uniq -d
```

실제 에러 형태:

```
./pff-a/pff-a-core/src/main/java/com/acme/a/core/AUtil.java:2: error: duplicate class: com.acme.a.core.AUtil
public final class AUtil {
             ^
1 error
```

중복이 나오면 셋 중 하나를 선택해야 합니다.

1. 중복이 있는 소스 루트 하나를 제외한다 (가장 현실적)
2. 프로젝트를 **충돌하지 않는 그룹**으로 나눠 여러 번 컴파일한다
3. 실제로 중복인지 확인 후 하나를 정본으로 삼고 나머지를 삭제한다 (`diff` 로 내용 비교)

```bash
# 중복 파일들의 실제 내용이 같은지 확인
find . -path '*/src/main/java/com/acme/a/core/AUtil.java' | xargs md5sum
```

### 2-3. 인코딩 혼재 검사

```bash
find . -path '*/src/main/java/*.java' -not -path '*/target/*' \
  | while read f; do
      enc=$(file -bi "$f" | sed 's/.*charset=//')
      echo "$enc"
    done | sort | uniq -c
```

`unknown-8bit` 이나 `iso-8859-1` 로 잡히는 게 EUC-KR/MS949 파일입니다. 컴파일 전에 통일하세요.

```bash
# 백업 후 EUC-KR → UTF-8 일괄 변환
find . -path '*/src/main/java/*.java' -not -path '*/target/*' | while read f; do
  if ! iconv -f UTF-8 -t UTF-8 "$f" >/dev/null 2>&1; then   # UTF-8 이 아니면
    iconv -f EUC-KR -t UTF-8 "$f" > "$f.tmp" && mv "$f.tmp" "$f"
    echo "converted: $f"
  fi
done
```

> 원본 저장소를 건드리면 안 되는 상황이면, 작업용 사본을 만들어 그쪽만 변환하세요.
> `rsync -a pff/ pff-work/` 후 `pff-work` 에서 작업.

### 2-4. 규모 파악

```bash
echo "프로젝트: $(find . -maxdepth 1 -mindepth 1 -type d | wc -l)"
echo "소스루트: $(find . -type d -path '*/src/main/java' -not -path '*/target/*' | wc -l)"
echo "java 파일: $(find . -path '*/src/main/java/*.java' -not -path '*/target/*' | wc -l)"
echo "로컬 JAR: $(find . -name '*.jar' -not -path '*/target/*' | wc -l)"
```

`.java` 가 3만 개를 넘어가면 javac 힙과 명령행 길이를 신경 써야 합니다 ([11장](#11-트러블슈팅) 참고).

---

## 3. 방법 A — Maven 단일 모듈 통합 컴파일

`pff/pom.xml` 을 새로 만듭니다. 하위 프로젝트의 기존 `pom.xml` 은 **전혀 손대지 않습니다.**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>local.merge</groupId>
  <artifactId>pff-all</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>          <!-- pom 아님! -->
  <name>PFF 통합 컴파일</name>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.parameters>true</maven.compiler.parameters>
    <main.class>com.acme.c.CApi</main.class>
  </properties>

  <build>
    <finalName>pff-all</finalName>

    <!-- 기본 소스 디렉터리를 존재하지 않는 경로로 돌려서 무력화 -->
    <sourceDirectory>${project.basedir}/.no-default-src</sourceDirectory>

    <plugins>
      <!-- (1) 모든 소스/리소스 루트를 추가 -->
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>build-helper-maven-plugin</artifactId>
        <version>3.6.0</version>
        <executions>
          <execution>
            <id>add-sources</id>
            <phase>generate-sources</phase>
            <goals><goal>add-source</goal></goals>
            <configuration>
              <sources>
                <source>${project.basedir}/pff-a/pff-a-core/src/main/java</source>
                <source>${project.basedir}/pff-a/pff-a-web/src/main/java</source>
                <source>${project.basedir}/pff-b/src/main/java</source>
                <source>${project.basedir}/pff-c/pff-c-api/src/main/java</source>
              </sources>
            </configuration>
          </execution>

          <execution>
            <id>add-resources</id>
            <phase>generate-resources</phase>
            <goals><goal>add-resource</goal></goals>
            <configuration>
              <resources>
                <resource>
                  <directory>${project.basedir}/pff-a/pff-a-core/src/main/resources</directory>
                </resource>
                <resource>
                  <directory>${project.basedir}/pff-b/src/main/resources</directory>
                </resource>
              </resources>
            </configuration>
          </execution>
        </executions>
      </plugin>

      <!-- (2) 컴파일러 -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
        <configuration>
          <proc>none</proc>                  <!-- 애노테이션 프로세싱 비활성 (분석 목적일 때) -->
          <failOnError>false</failOnError>   <!-- 에러가 나도 전수 수집 -->
          <compilerArgs>
            <arg>-Xmaxerrs</arg><arg>100000</arg>
            <arg>-Xlint:-options</arg>
            <arg>-nowarn</arg>
          </compilerArgs>
        </configuration>
      </plugin>

      <!-- (3) JAR 생성 -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.2</version>
        <configuration>
          <archive>
            <manifest>
              <mainClass>${main.class}</mainClass>
              <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
            </manifest>
            <manifestEntries>
              <Built-By>pff-all</Built-By>
              <Build-Jdk-Spec>21</Build-Jdk-Spec>
            </manifestEntries>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>

  <dependencies>
    <!-- 6장 참고 -->
  </dependencies>
</project>
```

### 실행

```bash
cd pff
mvn clean package                  # 컴파일 + JAR
mvn clean compile                  # 컴파일만
mvn clean compile -o               # 오프라인 (폐쇄망)
```

### 각 설정의 의미

| 설정 | 왜 필요한가 |
|---|---|
| `<packaging>jar</packaging>` | `pom` 이면 컴파일 자체를 안 함 |
| `<sourceDirectory>` 를 없는 경로로 | 기본 `src/main/java` 를 찾다 경고 내는 것 방지. build-helper 로 추가한 루트만 쓰게 함 |
| `<proc>none</proc>` | Lombok 등이 없는 상태에서 프로세서 탐색 실패로 멈추지 않게 |
| `<failOnError>false</failOnError>` | 첫 에러에서 멈추지 않고 전체 에러 목록 확보 |
| `-Xmaxerrs 100000` | javac 기본값 100 → 레거시에서는 순식간에 잘림 |
| `<finalName>` | `target/pff-all.jar` 로 고정 |

> **주의**: `<failOnError>false</failOnError>` 상태에서는 컴파일이 실패해도 `package` 가 진행되어 **불완전한 JAR** 이 만들어집니다. 배포용 JAR 을 만들 때는 반드시 `true`(기본값)로 되돌리세요.

---

## 4. 방법 A 확장 — JAR 생성 4가지

### 4-1. 일반 JAR (`maven-jar-plugin`)

위 POM 그대로. 산출물: `target/pff-all.jar` — 자기 클래스 + 리소스만 포함.

```bash
mvn clean package
jar --list --file target/pff-all.jar | head
unzip -p target/pff-all.jar META-INF/MANIFEST.MF
```

### 4-2. 실행 가능 Fat JAR (`maven-shade-plugin`) — 권장

의존성까지 전부 하나로 합칩니다. **`META-INF/services` 병합 transformer 가 핵심**입니다. 이게 없으면 JDBC 드라이버, SLF4J 바인딩, JAXB 구현체 로딩이 런타임에 조용히 실패합니다.

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>3.6.0</version>
  <executions>
    <execution>
      <phase>package</phase>
      <goals><goal>shade</goal></goals>
      <configuration>
        <createDependencyReducedPom>false</createDependencyReducedPom>
        <shadedArtifactAttached>true</shadedArtifactAttached>
        <shadedClassifierName>all</shadedClassifierName>
        <transformers>
          <transformer implementation=
            "org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
            <mainClass>${main.class}</mainClass>
          </transformer>
          <!-- SPI 파일 병합: 반드시 필요 -->
          <transformer implementation=
            "org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
          <!-- spring.handlers / spring.schemas 병합 (Spring 사용 시) -->
          <transformer implementation=
            "org.apache.maven.plugins.shade.resource.AppendingTransformer">
            <resource>META-INF/spring.handlers</resource>
          </transformer>
          <transformer implementation=
            "org.apache.maven.plugins.shade.resource.AppendingTransformer">
            <resource>META-INF/spring.schemas</resource>
          </transformer>
        </transformers>
        <filters>
          <filter>
            <artifact>*:*</artifact>
            <excludes>
              <!-- 서명된 JAR 을 합치면 SecurityException 발생 → 서명 제거 -->
              <exclude>META-INF/*.SF</exclude>
              <exclude>META-INF/*.DSA</exclude>
              <exclude>META-INF/*.RSA</exclude>
              <exclude>module-info.class</exclude>
            </excludes>
          </filter>
        </filters>
      </configuration>
    </execution>
  </executions>
</plugin>
```

산출물: `target/pff-all-all.jar`

```bash
java -jar target/pff-all-all.jar
```

### 4-3. Fat JAR (`maven-assembly-plugin`) — 간단한 대안

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-assembly-plugin</artifactId>
  <version>3.7.1</version>
  <configuration>
    <descriptorRefs>
      <descriptorRef>jar-with-dependencies</descriptorRef>
    </descriptorRefs>
    <archive>
      <manifest><mainClass>${main.class}</mainClass></manifest>
    </archive>
  </configuration>
  <executions>
    <execution>
      <id>make-fat</id>
      <phase>package</phase>
      <goals><goal>single</goal></goals>
    </execution>
  </executions>
</plugin>
```

산출물: `target/pff-all-jar-with-dependencies.jar`

> shade 와 달리 **SPI 파일을 병합하지 않고 덮어씁니다.** SPI 를 쓰는 라이브러리(JDBC, JAXB, SLF4J)가 하나라도 있으면 shade 를 쓰세요.

### 4-4. Thin JAR + `lib/` 디렉터리

배포 시 라이브러리를 별도로 두는 전통적인 방식. WAS 배포나 크기 제한이 있을 때 유용합니다.

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <version>3.7.1</version>
  <executions>
    <execution>
      <id>copy-deps</id>
      <phase>prepare-package</phase>
      <goals><goal>copy-dependencies</goal></goals>
      <configuration>
        <outputDirectory>${project.build.directory}/lib</outputDirectory>
        <includeScope>runtime</includeScope>
      </configuration>
    </execution>
  </executions>
</plugin>

<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <version>3.4.2</version>
  <configuration>
    <archive>
      <manifest>
        <mainClass>${main.class}</mainClass>
        <addClasspath>true</addClasspath>
        <classpathPrefix>lib/</classpathPrefix>
      </manifest>
    </archive>
  </configuration>
</plugin>
```

산출물: `target/pff-all.jar` + `target/lib/*.jar` (통째로 복사해서 배포)

### 4-5. 부가 산출물 — 소스 JAR

마이그레이션 검토용으로 소스를 한 덩어리로 묶어두면 편합니다.

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-source-plugin</artifactId>
  <version>3.3.1</version>
  <executions>
    <execution>
      <id>attach-sources</id>
      <phase>package</phase>
      <goals><goal>jar-no-fork</goal></goals>
    </execution>
  </executions>
</plugin>
```

### 4-6. JAR 방식 비교

| 방식 | 산출물 크기 | 단독 실행 | SPI 안전 | 적합한 상황 |
|---|---|---|---|---|
| jar-plugin | 작음 | ✗ | — | 라이브러리, 분석용 |
| shade | 큼 | ✓ | ✓ | **CLI 도구, 단독 배포** |
| assembly | 큼 | ✓ | ✗ | SPI 없는 단순 프로젝트 |
| thin + lib/ | 작음 | ✓ (lib 동반) | ✓ | WAS 배포, 라이브러리 교체가 잦을 때 |

---

## 5. 방법 B — 소스 루트 자동 스캔

프로젝트가 수십 개라 `<source>` 를 일일이 관리하기 싫을 때. `generate-sources` 단계에서 Groovy 로 훑어 추가합니다.

```xml
<plugin>
  <groupId>org.codehaus.gmavenplus</groupId>
  <artifactId>gmavenplus-plugin</artifactId>
  <version>3.0.2</version>
  <dependencies>
    <dependency>
      <groupId>org.apache.groovy</groupId>
      <artifactId>groovy</artifactId>
      <version>4.0.22</version>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
  <executions>
    <execution>
      <id>scan-source-roots</id>
      <phase>generate-sources</phase>
      <goals><goal>execute</goal></goals>
      <configuration>
        <scripts>
          <script><![CDATA[
            int srcCount = 0, resCount = 0
            def sep = File.separator
            project.basedir.eachDirRecurse { d ->
              if (d.absolutePath.contains("${sep}target${sep}")) return
              def parent = d.parentFile
              def grand  = parent?.parentFile
              if (parent?.name == 'main' && grand?.name == 'src') {
                if (d.name == 'java') {
                  project.addCompileSourceRoot(d.absolutePath); srcCount++
                } else if (d.name == 'resources') {
                  def r = new org.apache.maven.model.Resource()
                  r.directory = d.absolutePath
                  project.addResource(r); resCount++
                }
              }
            }
            log.info("source roots: ${srcCount}, resource roots: ${resCount}")
          ]]></script>
        </scripts>
      </configuration>
    </execution>
  </executions>
</plugin>
```

이 플러그인을 쓰면 build-helper 의 `add-source` / `add-resource` execution 은 제거해도 됩니다.

> **폐쇄망 주의**: `gmavenplus-plugin` + `groovy` 아티팩트가 Nexus 에 미러링돼 있어야 합니다. 부담스러우면 [2-1](#2-1-소스-루트-목록-뽑기)의 `find`+`sed` 로 `<source>` 목록을 생성해서 붙여넣는 편이 확실합니다.

---

## 6. 의존성 처리 전략

소스를 합치는 순간 **모든 프로젝트의 서드파티 의존성이 하나의 classpath** 에 올라와야 합니다. 실무에서 시간이 가장 많이 드는 부분입니다.

### 6-1. 각 프로젝트 의존성 수집

```bash
cd pff
for p in $(find . -name pom.xml -not -path './pom.xml' -not -path '*/target/*'); do
  d=$(dirname "$p")
  (cd "$d" && mvn -q -B dependency:list \
      -DoutputFile="$(pwd)/deps.txt" -DincludeScope=compile -DoutputAbsoluteArtifactFilename=false) \
    2>/dev/null || echo "  [skip] $d"
done

# 병합 + 중복 제거
find . -name deps.txt | xargs cat \
  | grep -E '^\s+[a-zA-Z]' | sed 's/^\s*//' | sort -u > all-deps.txt
wc -l all-deps.txt
```

### 6-2. 버전 충돌 찾기

같은 `groupId:artifactId` 가 서로 다른 버전으로 나오는 것들:

```bash
cut -d: -f1,2 all-deps.txt | sort | uniq -d
```

나온 것들은 `<dependencyManagement>` 로 하나로 고정합니다.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>6.1.12</version>   <!-- 최신 하나로 통일 -->
    </dependency>
  </dependencies>
</dependencyManagement>
```

### 6-3. `<dependency>` 블록 자동 생성

```bash
awk -F: '{printf "    <dependency>\n      <groupId>%s</groupId>\n      <artifactId>%s</artifactId>\n      <version>%s</version>\n    </dependency>\n", $1, $2, $4}' \
  all-deps.txt > deps-block.xml
```

> `dependency:list` 출력 포맷은 `groupId:artifactId:type:version:scope` 입니다. 버전 위치가 다르면 `$4` 를 조정하세요.

### 6-4. 로컬 JAR (`lib/*.jar`) 처리

레거시 프로젝트는 Maven 좌표 없는 JAR 을 `lib/` 에 두는 경우가 많습니다. 세 가지 방법이 있습니다.

**(a) 로컬 저장소에 일괄 등록 — 권장**

```bash
cd pff
for j in $(find . -name '*.jar' -not -path '*/target/*'); do
  base=$(basename "$j" .jar)
  mvn -q install:install-file \
    -Dfile="$j" \
    -DgroupId=local.lib \
    -DartifactId="$base" \
    -Dversion=1.0 \
    -Dpackaging=jar \
    -DgeneratePom=true
  echo "    <dependency><groupId>local.lib</groupId><artifactId>$base</artifactId><version>1.0</version></dependency>"
done | tee local-deps.xml
```

출력된 `<dependency>` 블록을 POM 에 붙여넣습니다.

**(b) `system` scope — 비권장이지만 빠름**

```xml
<dependency>
  <groupId>local.lib</groupId>
  <artifactId>legacy-util</artifactId>
  <version>1.0</version>
  <scope>system</scope>
  <systemPath>${project.basedir}/lib/legacy-util.jar</systemPath>
</dependency>
```

Maven 3.9+ 에서 경고가 나오고 JAR 패키징 시 포함되지 않습니다. 컴파일 검증 목적이면 충분합니다.

**(c) javac 로 넘어가기**

`maven-compiler-plugin` 에는 `additionalClasspathElements` 같은 파라미터가 **없습니다.** `<compilerArgs>` 로 `-cp` 를 주면 Maven 이 만든 classpath 와 충돌합니다. 로컬 JAR 이 수십 개라면 [방법 C](#7-방법-c--javac-직접-실행-경량) 가 훨씬 간단합니다 — `-cp "lib/*"` 한 줄이면 끝납니다.

---

## 7. 방법 C — javac 직접 실행 (경량)

**컴파일 가능 여부 확인이 목적이라면 이쪽이 압도적으로 빠르고 단순합니다.** POM 유지보수가 전혀 없고, 반복 실행이 수 초 단위입니다.

### 7-1. 최소 형태

```bash
cd pff
mkdir -p build/classes

# 1) 소스 목록 → argfile (명령행 길이 제한 회피)
find . -path '*/src/main/java/*.java' -not -path '*/target/*' -not -path './build/*' \
  > build/sources.txt

# 2) 컴파일
javac --release 21 \
      -encoding UTF-8 \
      -proc:none \
      -nowarn \
      -Xmaxerrs 100000 \
      -cp "lib/*" \
      -d build/classes \
      @build/sources.txt 2>&1 | tee build/compile.log
```

`@build/sources.txt` 형태의 **argfile** 이 핵심입니다. 파일이 수만 개여도 명령행 길이 제한(Windows 8191자, Linux ~2MB)에 걸리지 않습니다.

### 7-2. 주요 옵션

| 옵션 | 의미 |
|---|---|
| `--release 21` | 타깃 버전. 부트클래스패스까지 제한해 하위 호환 보장 |
| `-source 21 -target 21` | `--release` 대신. `--add-exports` 를 쓸 때는 이쪽이어야 함 |
| `-encoding UTF-8` | 소스 인코딩. JDK 18+ 기본값은 UTF-8 이지만 명시 권장 |
| `-proc:none` | 애노테이션 프로세싱 끔 (Lombok 없이 돌릴 때 필수) |
| `-Xmaxerrs 100000` | 에러 출력 상한 (기본 100) |
| `-Xmaxwarns 0` / `-nowarn` | 경고 억제 |
| `-nowarn -Xlint:-options` | `--release` 관련 잡음 제거 |
| `-d <dir>` | 클래스 출력 디렉터리 |
| `-cp "lib/*"` | 와일드카드로 디렉터리 내 모든 JAR (따옴표 필수 — 셸 확장 방지) |
| `-parameters` | 파라미터명 보존 (Spring MVC 필요) |
| `-g` | 전체 디버그 정보 (기본은 line+source만) |

### 7-3. 클래스패스 구성

**Maven 이 이미 도는 프로젝트가 하나라도 있으면** 거기서 classpath 를 뽑아 재사용합니다.

```bash
(cd pff-a/pff-a-core && mvn -q dependency:build-classpath -Dmdep.outputFile=/tmp/cp1.txt)
(cd pff-b && mvn -q dependency:build-classpath -Dmdep.outputFile=/tmp/cp2.txt)

CP="$(cat /tmp/cp1.txt):$(cat /tmp/cp2.txt):lib/*"
javac --release 21 -encoding UTF-8 -cp "$CP" -d build/classes @build/sources.txt
```

또는 `~/.m2/repository` 전체를 통째로 거는 무식하지만 확실한 방법:

```bash
find ~/.m2/repository -name '*.jar' ! -name '*-sources.jar' ! -name '*-javadoc.jar' \
  | tr '\n' ':' > build/cp.txt
javac --release 21 -encoding UTF-8 -cp "$(cat build/cp.txt)" -d build/classes @build/sources.txt
```

> 같은 라이브러리의 여러 버전이 섞이면 앞쪽 것이 이깁니다. 정확한 분석이 필요하면 쓰지 마세요. "일단 컴파일이나 되는지 보자" 단계에서만 유용합니다.

### 7-4. 에러 전수 분석

```bash
# 에러 총 건수
grep -c 'error:' build/compile.log

# 에러 유형별 빈도
grep -oE 'error: [a-z ]+' build/compile.log | sort | uniq -c | sort -rn | head -20

# 못 찾는 심볼 Top 50 (= 누락된 의존성 후보)
grep -A1 'symbol:' build/compile.log \
  | grep -oE 'symbol:\s+class\s+\S+' | sort | uniq -c | sort -rn | head -50

# 에러가 몰린 파일 Top 30
grep -oE '^[^:]+\.java' build/compile.log | sort | uniq -c | sort -rn | head -30

# 패키지별 에러 분포
grep -oE '^\./[^:]+\.java' build/compile.log \
  | sed 's|.*/src/main/java/||; s|/[^/]*\.java$||; s|/|.|g' \
  | sort | uniq -c | sort -rn | head -20
```

---

## 8. 방법 C 확장 — JAR 생성 상세

### 8-1. 리소스 병합

`.class` 만으로는 JAR 이 동작하지 않습니다. `src/main/resources` 를 모두 클래스 출력 디렉터리로 복사합니다.

```bash
for d in $(find . -type d -path '*/src/main/resources' -not -path '*/target/*'); do
  cp -R "$d"/. build/classes/
done
```

> 동일 경로의 리소스가 여러 프로젝트에 있으면 **뒤에 복사된 것이 덮어씁니다.** 중요한 설정 파일이라면 미리 확인하세요.
>
> ```bash
> find . -type d -path '*/src/main/resources' -not -path '*/target/*' \
>   | while read d; do (cd "$d" && find . -type f); done | sort | uniq -d
> ```

### 8-2. 기본 JAR

```bash
jar --create --file build/pff-all.jar -C build/classes .
```

구버전 문법도 동일하게 동작합니다: `jar cf build/pff-all.jar -C build/classes .`

| 옵션 | 의미 |
|---|---|
| `--create` / `-c` | 새 JAR 생성 |
| `--file` / `-f` | 출력 파일명 |
| `-C <dir> .` | 해당 디렉터리로 이동해서 그 안의 내용을 담음 (**경로 접두어 제거**) |
| `--verbose` / `-v` | 처리 내역 출력 |
| `--update` / `-u` | 기존 JAR 에 추가/갱신 |
| `--list` / `-t` | 내용 목록 |
| `--extract` / `-x` | 압축 해제 |
| `--main-class` / `-e` | Main-Class 지정 |
| `--manifest` / `-m` | 추가 매니페스트 파일 병합 |
| `--no-compress` / `-0` | 무압축 (대용량일 때 생성 속도 향상) |

> `-C build/classes .` 의 마지막 `.` 을 빠뜨리면 아무것도 안 담깁니다. 가장 흔한 실수입니다.

### 8-3. 실행 가능 JAR (Main-Class)

```bash
jar --create --file build/pff-all.jar \
    --main-class com.acme.c.CApi \
    -C build/classes .

java -jar build/pff-all.jar
```

매니페스트 확인:

```bash
unzip -p build/pff-all.jar META-INF/MANIFEST.MF
```

```
Manifest-Version: 1.0
Created-By: 21.0.10 (Ubuntu)
Main-Class: com.acme.c.CApi
```

### 8-4. Thin JAR + Class-Path 매니페스트

외부 JAR 을 매니페스트로 연결합니다.

```bash
cat > build/extra-manifest.txt <<'EOF'
Class-Path: ../lib/vendor-1.0.jar ../lib/other-2.0.jar
EOF

jar --create --file build/pff-thin.jar \
    --main-class com.acme.c.CApi \
    --manifest build/extra-manifest.txt \
    -C build/classes .

java -jar build/pff-thin.jar
```

**함정 세 가지:**

1. **`Class-Path` 는 JAR 파일이 놓인 위치 기준 상대경로**입니다. JAR 이 `build/` 에 있고 라이브러리가 `lib/` 에 있으면 `../lib/xxx.jar` 이어야 합니다. `lib/xxx.jar` 로 쓰면 `NoClassDefFoundError` 가 납니다.
2. **와일드카드(`lib/*`)를 지원하지 않습니다.** JAR 파일을 하나하나 나열해야 합니다.

   ```bash
   printf 'Class-Path: %s\n' "$(ls lib/*.jar | sed 's|^|../|' | tr '\n' ' ')" \
     > build/extra-manifest.txt
   ```
3. **한 줄이 72바이트를 넘으면 이어쓰기 규칙**(다음 줄 첫 칸에 공백 하나)을 지켜야 합니다. `jar --manifest` 는 자동으로 처리해 주지만, 직접 매니페스트를 쓸 때는 주의하세요. 파일 끝에 개행이 없으면 마지막 줄이 무시됩니다.

### 8-5. Fat JAR 수동 조립

```bash
FAT=build/fat
rm -rf "$FAT" && mkdir -p "$FAT"

# 1) 우리 클래스 + 리소스
cp -R build/classes/. "$FAT"/

# 2) 의존 JAR 전부 풀기
for j in lib/*.jar; do
  (cd "$FAT" && jar --extract --file "../../$j")
done

# 3) 서명 파일 및 개별 매니페스트 제거 (SecurityException 방지)
rm -f "$FAT"/META-INF/*.SF "$FAT"/META-INF/*.DSA "$FAT"/META-INF/*.RSA
rm -f "$FAT"/META-INF/MANIFEST.MF

# 4) 묶기
jar --create --file build/pff-fat.jar \
    --main-class com.acme.c.CApi \
    -C "$FAT" .

java -jar build/pff-fat.jar
```

> **SPI 파일 주의**: 여러 JAR 이 같은 `META-INF/services/xxx` 를 가지고 있으면 나중에 푼 것이 앞의 것을 **덮어씁니다.** JDBC 드라이버, SLF4J 바인딩, JAXB 구현체가 여기 해당합니다. 병합이 필요하면:
>
> ```bash
> mkdir -p build/svc
> for j in lib/*.jar; do
>   unzip -o -q "$j" 'META-INF/services/*' -d build/svc 2>/dev/null || true
> done
> # 수동으로 확인 후 병합
> ```
>
> 이런 경우가 있으면 손으로 하지 말고 **`maven-shade-plugin` 을 쓰세요** ([4-2](#4-2-실행-가능-fat-jar-maven-shade-plugin--권장)).

### 8-6. 소스 JAR

```bash
mkdir -p build/srcjar
while IFS= read -r f; do
  rel="${f#*/src/main/java/}"
  mkdir -p "build/srcjar/$(dirname "$rel")"
  cp "$f" "build/srcjar/$rel"
done < build/sources.txt

jar --create --file build/pff-all-sources.jar -C build/srcjar .
```

> 중복 FQCN 이 있으면 여기서도 덮어쓰기가 발생합니다.

---

## 9. 완성형 빌드 스크립트

`pff/build-all.sh` 로 저장하고 `chmod +x build-all.sh`.

```bash
#!/usr/bin/env bash
# 하위 모든 Maven 프로젝트의 src/main/java 를 한 번에 컴파일하고 JAR 로 묶는다.
set -euo pipefail

ROOT="$(cd "$(dirname "$0")" && pwd)"
BUILD="$ROOT/build"
CLASSES="$BUILD/classes"
RELEASE="${RELEASE:-21}"
ENC="${ENC:-UTF-8}"
MAIN_CLASS="${MAIN_CLASS:-}"
JAR_NAME="${JAR_NAME:-pff-all.jar}"

rm -rf "$BUILD"; mkdir -p "$CLASSES"

echo "==> 1. 소스 수집"
find "$ROOT" -path '*/src/main/java/*.java' \
     -not -path "$BUILD/*" -not -path '*/target/*' \
     | sort > "$BUILD/sources.txt"
echo "    $(wc -l < "$BUILD/sources.txt") 개 .java"

echo "==> 2. 중복 FQCN 검사"
DUP=$(sed 's|.*/src/main/java/||' "$BUILD/sources.txt" | sort | uniq -d || true)
if [ -n "$DUP" ]; then
  echo "    [경고] 동일 FQCN 중복 — 컴파일 실패 예상:"
  echo "$DUP" | sed 's/^/      /'
fi

echo "==> 3. 클래스패스 구성"
CP=""
[ -d "$ROOT/lib" ] && CP="$ROOT/lib/*"
if [ -f "$ROOT/cp.txt" ]; then
  CP="${CP:+$CP:}$(cat "$ROOT/cp.txt")"
fi
echo "    CP=${CP:-<없음>}"

echo "==> 4. 컴파일"
set +e
javac --release "$RELEASE" -encoding "$ENC" -proc:none -nowarn \
      -Xmaxerrs 100000 -Xlint:-options \
      ${CP:+-cp "$CP"} -d "$CLASSES" \
      @"$BUILD/sources.txt" 2> "$BUILD/compile.log"
RC=$?
set -e
grep -c 'error:' "$BUILD/compile.log" > "$BUILD/error-count.txt" || echo 0 > "$BUILD/error-count.txt"
echo "    에러 $(cat "$BUILD/error-count.txt") 건 (로그: build/compile.log)"
[ $RC -ne 0 ] && { echo "    컴파일 실패 — JAR 생성 생략"; exit $RC; }

echo "==> 5. 리소스 병합"
while IFS= read -r d; do
  cp -R "$d"/. "$CLASSES"/
done < <(find "$ROOT" -type d -path '*/src/main/resources' -not -path '*/target/*')

echo "==> 6. JAR 생성"
if [ -n "$MAIN_CLASS" ]; then
  jar --create --file "$BUILD/$JAR_NAME" --main-class "$MAIN_CLASS" -C "$CLASSES" .
else
  jar --create --file "$BUILD/$JAR_NAME" -C "$CLASSES" .
fi
echo "    $BUILD/$JAR_NAME ($(du -h "$BUILD/$JAR_NAME" | cut -f1), 엔트리 $(jar --list --file "$BUILD/$JAR_NAME" | wc -l))"
echo "==> 완료"
```

사용:

```bash
./build-all.sh                                   # JAR 만 생성
MAIN_CLASS=com.acme.c.CApi ./build-all.sh        # 실행 가능 JAR
RELEASE=17 ./build-all.sh                        # Java 17 로 컴파일
ENC=EUC-KR ./build-all.sh                        # 레거시 인코딩
```

실행 결과 예시:

```
==> 1. 소스 수집
    4 개 .java
==> 2. 중복 FQCN 검사
==> 3. 클래스패스 구성
    CP=/work/pff/lib/*
==> 4. 컴파일
    에러 0 건 (로그: build/compile.log)
==> 5. 리소스 병합
==> 6. JAR 생성
    /work/pff/build/pff-all.jar (4.0K, 엔트리 15)
==> 완료
```

---

## 10. Windows / PowerShell 버전

MobaXterm 이나 Git Bash 가 있으면 9장 스크립트를 그대로 쓸 수 있습니다. 순수 PowerShell 이 필요하다면:

```powershell
# build-all.ps1
$ErrorActionPreference = "Stop"
$Root    = $PSScriptRoot
$Build   = Join-Path $Root "build"
$Classes = Join-Path $Build "classes"
$Release = if ($env:RELEASE) { $env:RELEASE } else { "21" }

Remove-Item -Recurse -Force $Build -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force -Path $Classes | Out-Null

Write-Host "==> 1. 소스 수집"
$sources = Get-ChildItem -Path $Root -Recurse -Filter *.java |
  Where-Object { $_.FullName -match '\\src\\main\\java\\' -and $_.FullName -notmatch '\\target\\' }
$sources.FullName | Set-Content -Encoding UTF8 (Join-Path $Build "sources.txt")
Write-Host "    $($sources.Count) 개 .java"

Write-Host "==> 2. 중복 FQCN 검사"
$sources | ForEach-Object { ($_.FullName -split '\\src\\main\\java\\')[1] } |
  Group-Object | Where-Object Count -gt 1 |
  ForEach-Object { Write-Warning "  중복: $($_.Name)" }

Write-Host "==> 3. 컴파일"
$cp = Join-Path $Root "lib\*"
& javac --release $Release -encoding UTF-8 -proc:none -nowarn `
        -Xmaxerrs 100000 -cp $cp -d $Classes `
        "@$(Join-Path $Build 'sources.txt')" 2>&1 |
  Tee-Object -FilePath (Join-Path $Build "compile.log")

Write-Host "==> 4. 리소스 병합"
Get-ChildItem -Path $Root -Recurse -Directory |
  Where-Object { $_.FullName -match '\\src\\main\\resources$' -and $_.FullName -notmatch '\\target\\' } |
  ForEach-Object { Copy-Item -Path "$($_.FullName)\*" -Destination $Classes -Recurse -Force }

Write-Host "==> 5. JAR 생성"
& jar --create --file (Join-Path $Build "pff-all.jar") -C $Classes .
Write-Host "==> 완료"
```

> PowerShell 에서 argfile 을 넘길 때 `"@path"` 처럼 **따옴표로 감싸야** `@` 가 리터럴로 전달됩니다.
>
> 콘솔 한글이 깨지면: `chcp 65001` 실행 후 `$OutputEncoding = [Console]::OutputEncoding = [Text.Encoding]::UTF8`

---

## 11. 트러블슈팅

### 컴파일 단계

| 증상 | 원인 | 해결 |
|---|---|---|
| `error: duplicate class: X` | 동일 FQCN 이 여러 소스 루트에 존재 | [2-2](#2-2-중복-fqcn-검사--가장-중요) 참고. 소스 루트 제외 또는 그룹 분할 |
| `error: unmappable character` | 인코딩 불일치 | `-encoding EUC-KR` 로 맞추거나 [2-3](#2-3-인코딩-혼재-검사) 으로 일괄 변환 |
| `error: as of release 9, 'enum' is a keyword` | Java 1.4 이전 문법 | 해당 파일을 식별자 리네임. `--release 8` 로도 통과 안 됨 |
| `cannot find symbol: class XxxBean` | 의존성 누락 | [7-4](#7-4-에러-전수-분석) 의 심볼 Top 50 으로 누락 라이브러리 특정 |
| `package javax.xml.bind does not exist` | JDK 11+ 에서 제거된 모듈 | `jakarta.xml.bind-api` + `jaxb-runtime` 추가 |
| `package sun.jdbc.rowset does not exist` | 내부 API 제거 | `com.sun.rowset` 대체 또는 별도 구현으로 교체 |
| 에러가 100개에서 끊김 | `-Xmaxerrs` 기본값 | `-Xmaxerrs 100000` |
| `OutOfMemoryError` (javac) | 힙 부족 | `export JAVA_TOOL_OPTIONS="-Xmx4g"` 또는 `javac -J-Xmx4g` |
| `OutOfMemoryError` (Maven) | 힙 부족 | `export MAVEN_OPTS="-Xmx4g"` |
| `Argument list too long` | 명령행 길이 제한 | argfile(`@sources.txt`) 사용 — 9장 스크립트는 이미 적용됨 |
| `--release` 와 `--add-exports` 충돌 | 상호 배타 | `-source 21 -target 21` 로 변경 |

### JAR / 실행 단계

| 증상 | 원인 | 해결 |
|---|---|---|
| JAR 이 비어 있음 | `-C dir` 뒤의 `.` 누락 | `jar --create -f x.jar -C build/classes .` |
| `no main manifest attribute` | Main-Class 미지정 | `--main-class` 또는 `--manifest` 사용 |
| `NoClassDefFoundError` (thin JAR) | `Class-Path` 상대경로 기준 오해 | JAR **파일 위치** 기준. `../lib/x.jar` 형태 |
| `Class-Path` 의 `lib/*` 가 안 먹음 | 와일드카드 미지원 | JAR 을 개별 나열 ([8-4](#8-4-thin-jar--class-path-매니페스트)) |
| `SecurityException: Invalid signature file digest` | 서명된 JAR 을 fat JAR 로 합침 | `META-INF/*.SF,*.DSA,*.RSA` 제거 |
| `ServiceConfigurationError` / 드라이버 미탐지 | SPI 파일 덮어쓰기 | shade 의 `ServicesResourceTransformer` 사용 |
| 콘솔 한글 깨짐 | 표준출력 인코딩 | `java -Dstdout.encoding=UTF-8 ...` (JDK 19+), Windows 는 `chcp 65001` |
| `UnsupportedClassVersionError` | 실행 JVM 이 컴파일 타깃보다 낮음 | 실행 측 JDK 확인 또는 `--release` 낮춤 |

### 폐쇄망

- Maven 플러그인 버전을 **전부 고정**하세요. 미고정 시 슈퍼 POM 기본 버전이 잡혀 `release` 옵션이 조용히 무시되거나 메타데이터 조회로 실패합니다.
- `mvn -o` 로 오프라인 강제. 실패 캐시는 `~/.m2/repository/**/resolver-status.properties` 삭제로 초기화.
- **방법 C(javac)는 네트워크가 전혀 필요 없습니다.** 폐쇄망에서는 이쪽이 압도적으로 유리합니다.

---

## 12. 결과 검증 체크리스트

```bash
# 1) 바이트코드 버전 — Java 21 이면 65
javap -verbose -cp build/classes com.acme.b.BService | grep 'major version'
#   Java 8=52, 11=55, 17=61, 21=65

# 2) 소스 수 vs 클래스 수 (내부/익명 클래스 때문에 class 가 더 많은 게 정상)
echo "java : $(wc -l < build/sources.txt)"
echo "class: $(find build/classes -name '*.class' | wc -l)"
echo "top-level class: $(find build/classes -name '*.class' ! -name '*$*' | wc -l)"

# 3) JAR 내용 확인
jar --list --file build/pff-all.jar | wc -l
jar --list --file build/pff-all.jar | grep -c '\.class$'
unzip -p build/pff-all.jar META-INF/MANIFEST.MF

# 4) 리소스 누락 확인
jar --list --file build/pff-all.jar | grep -E '\.(properties|xml|sql)$'

# 5) 실행 스모크 테스트
java -Dstdout.encoding=UTF-8 -cp "build/pff-all.jar:lib/*" com.acme.c.CApi

# 6) 클래스 로딩 검증 (전 클래스 초기화 없이 링크만 확인)
find build/classes -name '*.class' ! -name '*$*' \
  | sed 's|build/classes/||; s|\.class$||; s|/|.|g' \
  | while read c; do
      javap -cp build/classes "$c" > /dev/null 2>&1 || echo "BROKEN: $c"
    done
```

---

## 13. 방법 선택 기준

```
목적이 무엇인가?
│
├─ 컴파일 에러 전수 조사 / 마이그레이션 영향도 분석
│   └─► 방법 C (javac 직접)          ★ 가장 빠름, 네트워크 불필요
│
├─ 단일 실행 가능 JAR 산출물이 필요
│   └─► 방법 A + shade                ★ SPI 안전, 표준적
│
├─ OpenRewrite / SonarQube 등 Maven 리액터를 요구하는 도구에 물림
│   └─► 방법 A (+ 방법 B 자동 스캔)
│
├─ 프로젝트가 수십 개이고 구조가 자주 바뀜
│   └─► 방법 B (Groovy 자동 스캔)
│
└─ 그냥 각 프로젝트를 순서대로 빌드하고 싶었던 것
    └─► Aggregator POM (<modules> 나열) — 이 문서의 대상이 아님
```

### 방법별 요약

| | 방법 A (Maven) | 방법 B (자동 스캔) | 방법 C (javac) |
|---|---|---|---|
| 초기 설정 비용 | 중 | 중 | **낮음** |
| 반복 실행 속도 | 느림 | 느림 | **빠름** |
| 의존성 관리 | Maven 자동 | Maven 자동 | 수동 |
| 로컬 JAR 처리 | 번거로움 | 번거로움 | **`-cp "lib/*"`** |
| 구조 변경 대응 | 수동 | **자동** | **자동** |
| 네트워크 필요 | 예 | 예 | **아니오** |
| 산출물 다양성 | **높음** | **높음** | 중간 |
| 외부 도구 연동 | **좋음** | **좋음** | 제한적 |

---

## 부록 — 검증 환경

이 문서의 방법 C(javac 직접 실행) 및 JAR 생성 절차는 아래 환경에서 실제 구동 확인했습니다.

- OpenJDK 21.0.10 (Ubuntu)
- Apache Maven 3.9.11
- 4개 소스 루트 / 3개 프로젝트 / 로컬 JAR 1개 / 리소스 2개 / 프로젝트 간 상호 참조 포함
- 확인 항목: 통합 컴파일, 중복 FQCN 에러 재현, 리소스 병합, 일반/thin/fat JAR 생성, `Class-Path` 상대경로 동작, fat JAR 단독 실행, 바이트코드 major version 65

방법 A(Maven POM)는 XML 스키마 검증까지만 수행했습니다. 사내 Nexus 환경에서 플러그인 버전을 확인하고 적용하세요.
