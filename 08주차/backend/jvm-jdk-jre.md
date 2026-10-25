# JVM / JDK / JRE

## 세 가지의 관계

```text
JDK (Java Development Kit)
 ├─ 개발 도구: javac, jar, javadoc, jdb, jshell, jcmd ...
 └─ JRE (Java Runtime Environment)
      ├─ Java 표준 라이브러리 (java.lang, java.util ...)
      └─ JVM (Java Virtual Machine)
```

| | 역할 | 누가 필요한가 |
| --- | --- | --- |
| JVM | 바이트코드(`.class`)를 읽어 OS에 맞게 실행하는 가상 머신 | 실행 환경 |
| JRE | JVM + 실행에 필요한 표준 라이브러리 | 자바 프로그램을 **실행만** 하는 사람 |
| JDK | JRE + 컴파일러 등 개발 도구 | 자바 프로그램을 **개발**하는 사람 |

- Java 11부터 Oracle은 JRE를 따로 배포하지 않는다. 대신 JDK에 포함된 `jlink`로 애플리케이션에 필요한 모듈만 담은 **커스텀 런타임**을 만들 수 있다.
- 실무에서 Docker 이미지에 JDK 대신 JRE(또는 jlink로 만든 런타임) 기반 이미지를 쓰는 이유도 실행에는 컴파일러가 필요 없어 이미지 크기를 줄일 수 있기 때문이다.
- JDK 배포판은 Oracle JDK 외에도 OpenJDK 기반의 Eclipse Temurin, Amazon Corretto, Azul Zulu 등이 있다. 소스는 같은 OpenJDK이고 라이선스와 지원 정책이 다르다.

## 자바 프로그램의 실행 과정

```text
Hello.java ──(javac 컴파일)──▶ Hello.class (바이트코드) ──(java 실행)──▶ JVM ──▶ OS / CPU
```

1. `javac`가 `.java` 소스를 **바이트코드**(`.class`)로 컴파일한다. 바이트코드는 특정 CPU가 아닌 JVM이 이해하는 중간 언어다.
2. `java Hello`를 실행하면 JVM이 뜨고, [Class Loader](./jvm-architecture.md#class-loader)가 `.class` 파일을 메모리에 올린다.
3. **Execution Engine**이 바이트코드를 기계어로 바꿔 실행한다.
   - 인터프리터: 바이트코드를 한 줄씩 해석해 실행
   - JIT 컴파일러: 자주 실행되는 코드(Hot Spot)를 통째로 기계어로 컴파일해 캐시

```bash
javac Hello.java      # Hello.class 생성
javap -c Hello        # 바이트코드 확인
java Hello            # 실행
```

## 플랫폼 독립성

> Write Once, Run Anywhere

- C/C++는 소스를 특정 OS/CPU용 기계어로 컴파일하므로, Windows용으로 빌드한 실행 파일은 Linux에서 돌지 않는다.
- 자바는 OS와 상관없는 **바이트코드**로 컴파일하고, 각 OS에 맞는 **JVM**이 이를 실행한다.
- 즉 **자바 프로그램은 플랫폼에 독립적이지만, JVM은 플랫폼에 종속적**이다. OS별로 다른 JVM을 설치해야 한다.

단점과 보완

- 바이트코드를 한 번 더 해석해야 하므로 네이티브 코드보다 느릴 수 있다. → JIT 컴파일로 상당 부분 보완
- JVM이 뜨고 클래스를 로딩하는 **시작 시간**이 길다. → 서버리스, 컨테이너 환경에서 문제가 되어 GraalVM Native Image(AOT 컴파일), CDS(Class Data Sharing) 같은 방법이 나왔다.
