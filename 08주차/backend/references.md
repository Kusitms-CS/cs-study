# 8주차 참고 자료

## 입문

- [JVM Internal (네이버 D2)](https://d2.naver.com/helloworld/1230)
  - 바이트코드, 클래스 로더, Runtime Data Area, Execution Engine을 한 번에 훑는 JVM 내부 구조 해설
- [Java Garbage Collection (네이버 D2)](https://d2.naver.com/helloworld/1329)
  - Young/Old 영역, Minor/Full GC, Serial부터 G1까지 GC 방식별 동작 원리
- [Lesson: Generics (Oracle Java Tutorials)](https://docs.oracle.com/javase/tutorial/java/generics/index.html)
  - 제네릭 타입, 타입 제한, 와일드카드, 타입 소거와 제약 사항
- [Lesson: Exceptions (Oracle Java Tutorials)](https://docs.oracle.com/javase/tutorial/essential/exceptions/index.html)
  - Checked/Unchecked 구분, try-with-resources, 예외 체이닝과 Unchecked 예외 논쟁
- [Trail: Collections (Oracle Java Tutorials)](https://docs.oracle.com/javase/tutorial/collections/index.html)
  - 컬렉션 인터페이스와 구현체, 알고리즘, 커스텀 구현 방법

## 기본기

- [The Java Virtual Machine Specification, Java SE 21 - Chapter 2. Structure of the JVM](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html)
  - Runtime Data Area와 프레임 구조를 정의한 JVM 명세
- [The Java Virtual Machine Specification, Java SE 21 - Chapter 5. Loading, Linking, and Initializing](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html)
  - 클래스 로딩, 링킹, 초기화 과정과 클래스 로더 위임 규칙
- [HotSpot Glossary of Terms (OpenJDK)](https://openjdk.org/groups/hotspot/docs/HotSpotGlossary.html)
  - Card Table, Safepoint, TLAB 등 HotSpot JVM 용어 정리

## GC

- [HotSpot Virtual Machine Garbage Collection Tuning Guide (Java 21)](https://docs.oracle.com/en/java/javase/21/gctuning/index.html)
  - GC별 특징과 선택 기준, G1/ZGC 튜닝 옵션에 대한 공식 가이드
- [Java Reference와 GC (네이버 D2)](https://d2.naver.com/helloworld/329631)
  - Strong/Soft/Weak/Phantom Reference와 Reachability에 따른 GC 동작
- [Garbage Collection 모니터링 방법 (네이버 D2)](https://d2.naver.com/helloworld/6043)
  - jstat, verbosegc, VisualVM 등으로 GC 상태를 확인하는 방법
- [Garbage Collection 튜닝 (네이버 D2)](https://d2.naver.com/helloworld/37111)
  - GC 튜닝이 필요한 시점과 Heap 크기, GC 방식 결정 절차
- [ZGC의 기본 개념 이해하기 (네이버 D2)](https://d2.naver.com/helloworld/0128759)
  - Colored Pointer와 Load Barrier로 STW를 줄이는 ZGC의 동작 원리
- [JEP 439: Generational ZGC](https://openjdk.org/jeps/439)
  - Java 21에서 ZGC에 세대 구분을 도입한 배경과 설계
- [JEP 474: ZGC: Generational Mode by Default](https://openjdk.org/jeps/474)
  - Java 23에서 Generational ZGC를 기본 모드로 바꾼 결정

## Exception / 성능

- [Rolling Back a Declarative Transaction (Spring 공식 문서)](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)
  - `@Transactional`이 RuntimeException에만 롤백하는 기본 규칙과 `rollbackFor` 설정
- [자바 애플리케이션 성능 튜닝의 도(道) (네이버 D2)](https://d2.naver.com/helloworld/184615)
  - 성능 문제 접근 순서와 GC, 스레드 덤프 분석 사례
