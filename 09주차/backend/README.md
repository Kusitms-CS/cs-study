# 9주차 - Spring Core

## 1. IoC / DI

### IoC (Inversion of Control, 제어의 역전)
- 객체를 **누가 만들고 관리하느냐**의 제어권이 개발자에서 **스프링(컨테이너)** 으로 넘어간 것
- 기존에는 개발자가 `new`로 직접 객체를 만들고 연결했다면, 스프링에서는 컨테이너가 객체를 만들고 필요한 곳에 넣어줌

```java
// 기존 방식: 내가 직접 생성
OrderService orderService = new OrderService(new MemoryOrderRepository());
```

### DI (Dependency Injection, 의존성 주입)
- 객체가 필요로 하는 다른 객체(의존성)를 **외부에서 넣어주는 것**
- IoC를 구현하는 대표적인 방법

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    // 생성자를 통해 스프링이 OrderRepository를 주입해줌
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

### DI 방식 3가지
| 방식 | 설명 | 비고 |
|---|---|---|
| 생성자 주입 | 생성자로 의존성 전달 | **가장 권장**, `final` 사용 가능, 필수 의존성 보장 |
| 세터 주입 | setter 메서드로 전달 | 선택적 의존성에 사용, 값이 바뀔 수 있음 |
| 필드 주입 | 필드에 `@Autowired` | 테스트하기 어려워서 비권장 |

### DI를 쓰는 이유
- 객체 간 **결합도가 낮아짐** (구현체를 바꿔도 코드 수정이 적음)
- **테스트하기 쉬워짐** (가짜 객체를 넣을 수 있음)

---

## 2. Bean

- **스프링 컨테이너가 관리하는 객체**
- `new`로 만든 객체는 Bean이 아니고, 스프링이 등록하고 관리해야 Bean

### Bean 등록 방법
```java
// 1) 컴포넌트 스캔: 클래스에 어노테이션을 붙이면 자동 등록
@Component   // @Service, @Repository, @Controller도 포함
public class MemberService { }

// 2) 설정 클래스에서 직접 등록
@Configuration
public class AppConfig {
    @Bean
    public MemberService memberService() {
        return new MemberService();
    }
}
```

### 특징
- 기본적으로 **싱글톤**: 컨테이너 안에 하나만 만들어서 계속 공유
- Bean 이름은 기본적으로 클래스명의 첫 글자를 소문자로 바꾼 것 (`MemberService` → `memberService`)

---

## 3. BeanFactory

- 스프링 컨테이너의 **가장 기본적인 인터페이스**
- Bean을 **등록, 조회, 관리**하는 핵심 기능을 담당
- `getBean()`으로 Bean을 꺼낼 수 있음
- Bean을 **실제로 필요할 때 생성**(지연 로딩)하는 방식

> 실무에서는 BeanFactory를 직접 쓰는 일은 거의 없고, 아래 ApplicationContext를 사용함

---

## 4. ApplicationContext

- **BeanFactory를 상속**받아 기능을 더 추가한 컨테이너
- 우리가 보통 말하는 "스프링 컨테이너"가 바로 이것

### BeanFactory와 차이
| | BeanFactory | ApplicationContext |
|---|---|---|
| Bean 관리 | O | O |
| 생성 시점 | 사용할 때 (지연) | 컨테이너 시작 시 (싱글톤 미리 생성) |
| 부가 기능 | X | O |

### ApplicationContext의 부가 기능
- **메시지 소스**: 국제화(다국어) 처리
- **환경 변수**: 프로파일(dev, prod 등), 설정값 관리
- **이벤트 발행**: 이벤트를 발행하고 구독
- **리소스 조회**: 파일, 클래스패스 등 편리하게 읽기

---

## 5. BeanDefinition

- Bean을 만들기 위한 **설계도(메타정보)**
- 스프링은 Bean을 바로 만드는 게 아니라, 먼저 BeanDefinition을 만들고 이걸 기반으로 Bean을 생성함

### 담고 있는 정보
- Bean 클래스 정보
- Bean 이름
- 스코프 (싱글톤, 프로토타입 등)
- 지연 초기화 여부 (`lazy`)
- 초기화 / 소멸 메서드
- 의존 관계 정보

### 왜 필요한가?
- **어떤 방식으로 등록했든**(어노테이션, 자바 설정, XML) 컨테이너는 BeanDefinition만 보고 동작함
- 덕분에 스프링이 다양한 설정 방식을 **같은 방식으로** 처리할 수 있음

```
[설정 정보] → BeanDefinition 생성 → Bean 생성 → 컨테이너에 보관
(어노테이션, @Bean, XML)
```

---

## 6. Bean Lifecycle

Bean이 **생성되고 사라지기까지의 과정**

### 전체 흐름
```
스프링 컨테이너 생성
→ 스프링 Bean 생성
→ 의존관계 주입
→ 초기화 콜백
→ 사용
→ 소멸 전 콜백
→ 스프링 종료
```

### 단계별 설명
1. **객체 생성**: 생성자 호출로 Bean 객체 생성
2. **의존관계 주입**: 필요한 다른 Bean을 주입
3. **초기화 콜백**: 모든 준비가 끝난 뒤 호출 (DB 연결, 캐시 로딩 등 초기 작업)
4. **사용**: 애플리케이션에서 Bean 사용
5. **소멸 전 콜백**: 컨테이너가 종료되기 직전 호출 (자원 정리, 연결 종료 등)

### 초기화 / 소멸 콜백 설정 방법
```java
@Component
public class NetworkClient {

    @PostConstruct   // 초기화 콜백
    public void init() {
        System.out.println("연결 시작");
    }

    @PreDestroy      // 소멸 전 콜백
    public void close() {
        System.out.println("연결 종료");
    }
}
```

| 방법 | 설명 |
|---|---|
| `@PostConstruct`, `@PreDestroy` | 어노테이션 방식, **가장 권장** |
| `@Bean(initMethod, destroyMethod)` | 외부 라이브러리 클래스에 적용할 때 유용 |
| `InitializingBean`, `DisposableBean` | 인터페이스 방식, 스프링에 종속적이라 거의 안 씀 |

### 생성자에서 초기화하지 않고 콜백을 쓰는 이유
- 생성자 시점에는 **의존관계 주입이 아직 끝나지 않았을 수 있음**
- 그래서 주입이 모두 끝난 뒤에 호출되는 초기화 콜백에서 초기 작업을 하는 것이 안전함

---

## 7. Proxy

- 실제 객체를 **대신하는 대리 객체**
- 클라이언트가 실제 객체를 호출하면 **Proxy가 중간에서 먼저 받아서** 부가 작업을 한 뒤 실제 객체를 호출함

```
Client → Proxy → 실제 객체(Target)
```

### 스프링에서 Proxy를 쓰는 이유
- 실제 코드를 건드리지 않고 **공통 기능(부가 기능)을 추가**할 수 있음
- 대표적인 예: `@Transactional`, `@Async`, `@Cacheable`, AOP

```java
@Service
public class MemberService {

    @Transactional
    public void join(Member member) {
        // 비즈니스 로직만 작성
        // 트랜잭션 시작/커밋/롤백은 Proxy가 대신 처리
    }
}
```

### 동작 과정
1. 스프링이 Bean을 만든 뒤 Proxy 적용이 필요한지 확인
2. 필요하면 **원본 대신 Proxy 객체를 Bean으로 등록**
3. 호출 시 Proxy가 부가 기능(트랜잭션 시작 등) → 실제 메서드 호출 → 부가 기능(커밋 등) 순서로 처리

### Proxy 생성 방식
| 방식 | 설명 |
|---|---|
| JDK Dynamic Proxy | **인터페이스** 기반으로 Proxy 생성 |
| CGLIB | **클래스를 상속**해서 Proxy 생성 (스프링 부트 기본값) |

### 주의할 점
- 같은 클래스 안에서 자기 메서드를 호출하면(**self-invocation**) Proxy를 거치지 않아서 부가 기능이 적용되지 않음
- CGLIB은 상속 방식이라 `final` 클래스나 `final` 메서드에는 적용할 수 없음

---

## 한눈에 정리

| 키워드 | 한 줄 요약 |
|---|---|
| IoC | 객체 제어권이 스프링으로 넘어감 |
| DI | 필요한 객체를 외부에서 주입받음 |
| Bean | 스프링이 관리하는 객체 |
| BeanFactory | Bean을 관리하는 가장 기본적인 컨테이너 |
| ApplicationContext | BeanFactory + 부가 기능, 실제로 쓰는 컨테이너 |
| BeanDefinition | Bean을 만들기 위한 설계도 |
| Bean Lifecycle | 생성 → 주입 → 초기화 → 사용 → 소멸 |
| Proxy | 실제 객체를 감싸서 부가 기능을 추가하는 대리 객체 |