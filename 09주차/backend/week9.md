# Spring Core: 객체를 만들고 연결하는 일은 누가 맡을까?

## 1. IoC / DI: 필요한 객체를 직접 만드는 대신 전달받기

주문 서비스가 결제 수단으로 카드 결제를 사용한다고 하자. 서비스 안에서 `new CardPaymentClient()`를 호출하면 결제 구현의 선택과 생성까지 서비스가 맡는다. 테스트에서 가짜 결제 수단으로 바꾸려 해도 서비스 코드를 수정해야 할 수 있다.

```java
interface PaymentClient {
    void pay(long amount);
}

class OrderService {
    private final PaymentClient paymentClient;

    OrderService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    void order(long amount) {
        paymentClient.pay(amount);
    }
}
```

위 서비스는 결제 객체를 직접 생성하지 않고 생성자 인자로 받는다. 어떤 구현을 연결할지는 외부에서 결정한다. 이것이 DI(Dependency Injection, 의존성 주입)다. DI 자체는 Spring 없이도 사용할 수 있다.

IoC(Inversion of Control, 제어의 역전)는 객체 생성·연결이나 실행 흐름의 제어를 프레임워크 등 외부에 맡기는 더 넓은 개념이다. **DI는 IoC를 구현하는 방식 중 하나**이며, Spring의 IoC 컨테이너는 설정에 따라 객체를 생성하고 의존 관계를 연결한다.

```text
서비스가 구현체 선택·생성 → 서비스가 그 객체 사용
                         ↓ DI 적용
외부에서 구현체 선택·연결 → 서비스는 전달받은 객체 사용
```

생성자 주입은 필수 의존성을 드러내고 필드를 `final`로 둘 수 있게 한다. Spring이 관리하는 클래스에 생성자가 하나라면 보통 `@Autowired`를 생략할 수 있다. 생성자 주입만으로 모든 순환 의존성이 해결되는 것은 아니다. A의 생성에 B가 필요하고 B의 생성에 A가 필요하면 생성할 수 없으므로 역할과 의존 방향부터 다시 살펴야 한다.

여러 빈이 같은 타입을 구현한다면 `@Qualifier`나 `@Primary` 등으로 선택 기준을 명확히 한다. 주입은 단순히 변수 이름이 같은 객체를 찾아 넣는 기능으로 이해하면 부족하다.

---

## 2. Bean: Spring이 관리하는 객체

Bean은 Spring IoC 컨테이너가 관리하는 객체다. 클래스 자체가 아니라 **컨테이너에 등록되어 생성·주입·초기화 등의 관리를 받는 객체**를 가리킨다.

| 등록 방식 | 사용 예 |
| --- | --- |
| `@Component`와 컴포넌트 스캔 | 애플리케이션의 서비스나 저장소 클래스 발견 |
| `@Configuration`의 `@Bean` | 생성 과정을 직접 구성하거나 외부 라이브러리 객체 등록 |
| XML 또는 프로그래밍 방식 | 별도 설정으로 빈 정의 등록 |

`@Service`, `@Repository`, `@Controller`도 컴포넌트 스캔 대상이 되는 대표적인 stereotype이다. 애너테이션이 있어도 스캔이나 명시적 등록이 이루어지지 않으면 빈이 되지 않는다.

앞의 예제에 카드 결제 구현과 설정을 추가하면 다음과 같다. 출력문은 결제 흐름을 보여주기 위한 예시다.

```java
class CardPaymentClient implements PaymentClient {
    @Override
    public void pay(long amount) {
        System.out.println("카드 결제: " + amount);
    }
}
```

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
class OrderConfig {
    @Bean
    PaymentClient paymentClient() {
        return new CardPaymentClient();
    }

    @Bean
    OrderService orderService(PaymentClient paymentClient) {
        return new OrderService(paymentClient);
    }
}
```

`@Bean` 메서드 안에서 `new`를 사용해도 반환된 객체는 컨테이너가 관리할 수 있다. 따라서 `new` 사용 여부만으로 빈인지 판단할 수는 없다. 반면 애플리케이션 코드에서 별도로 `new OrderService(...)`를 하면 그 객체는 자동으로 컨테이너에 등록되지 않는다.

### 기본 singleton은 JVM 전체의 유일한 객체일까?

기본 스코프인 singleton은 **컨테이너별, 빈 정의별로 하나의 인스턴스를 공유**한다는 뜻이다. 같은 클래스를 서로 다른 빈으로 등록하면 여러 인스턴스가 생길 수 있다.

| 스코프 | 생성·공유 기준 |
| --- | --- |
| singleton | 컨테이너 안에서 해당 빈을 공유 |
| prototype | 컨테이너에 해당 빈의 생성을 요청할 때 새 객체 생성 |
| request | 웹 환경에서 HTTP 요청별 객체 |
| session | 웹 환경에서 HTTP 세션별 객체 |

singleton은 스레드 안전성을 보장하지 않는다. 여러 요청이 서비스 필드의 고객 ID나 주문 금액을 함께 수정하면 경쟁 조건이 생길 수 있다. 요청별 값은 지역 변수나 인자로 처리하는 편이 안전하다.

또한 singleton에 prototype 빈을 한 번 주입하면 그 주입된 객체를 계속 사용한다. 매번 새 객체가 필요하다면 `ObjectProvider` 같은 지연 조회 방법 등을 별도로 검토해야 한다.

---

## 3. BeanFactory: 빈 조회와 관리의 기본 계약

BeanFactory는 Spring IoC 기능의 기반이 되는 인터페이스다. `getBean()` 등의 API로 이름이나 타입에 맞는 빈을 조회할 수 있다. 실제 빈 정의 등록·탐색·생성은 `DefaultListableBeanFactory` 같은 구현체와 관련 확장 인터페이스가 담당한다.

```text
빈 이름·타입으로 조회
    ↓
이미 생성된 singleton이 있는가?
    ├─ 있으면 공유 객체 반환
    └─ 없으면 정의와 스코프에 따라 생성·주입·초기화 후 반환
```

이 그림은 일반적인 singleton 조회 흐름을 단순화한 것이다. 모든 조회가 생성자를 호출하는 것은 아니며, 반환값은 원본 객체 대신 프록시일 수도 있다.

BeanFactory 자체를 애플리케이션의 모든 서비스에 주입해 `getBean()`을 호출하면 서비스가 컨테이너에 의존하게 된다. 일반적인 업무 코드에서는 생성자 주입으로 필요한 객체를 직접 받는 편이 의존 관계를 파악하고 테스트하기 쉽다.

`FactoryBean`은 이름이 비슷하지만 다른 개념이다. BeanFactory가 컨테이너의 기본 계약이라면, FactoryBean은 복잡한 객체의 생성을 캡슐화하는 빈의 계약이다. FactoryBean으로 등록된 이름을 조회하면 보통 생성 결과를 받고, 팩토리 자체는 `&빈이름`으로 조회한다.

---

## 4. ApplicationContext: BeanFactory에 애플리케이션 기능 더하기

ApplicationContext는 BeanFactory 계열의 기능을 포함하면서 애플리케이션에 필요한 기능을 제공하는 상위 컨테이너 인터페이스다. 대표적으로 이벤트 발행, 메시지 국제화, 리소스 접근과 확장 지점의 자동 인식 등을 지원한다.

| 비교 | BeanFactory 계층 | ApplicationContext |
| --- | --- | --- |
| 빈 조회·의존 관계 관리 | 기반 기능 제공 | 내부 BeanFactory를 통해 제공 |
| 후처리기 적용 준비 | 직접 사용할 때 명시적인 설정·등록 필요 | 컨텍스트 준비 과정에서 자동 탐색·등록 |
| 이벤트·국제화·리소스 | 기본 BeanFactory 계약의 범위 밖 | 관련 기능 제공 |
| 일반 애플리케이션 사용 | 기반 구현·프레임워크 통합 등 | 주로 사용하는 컨테이너 |

BeanFactory는 무조건 지연 생성, ApplicationContext는 무조건 즉시 생성이라고 나누면 부정확하다. 일반적인 ApplicationContext는 refresh 과정에서 지연 설정이 없는 singleton을 미리 생성하지만, `@Lazy`나 스코프에 따라 시점이 달라진다. 지연 빈도 즉시 생성되는 다른 빈의 필수 의존성이면 시작할 때 만들어질 수 있다.

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

// 앞의 PaymentClient, OrderService, CardPaymentClient, OrderConfig를 등록한 예제
class OrderExample {
    public static void main(String[] args) {
        try (var context = new AnnotationConfigApplicationContext(OrderConfig.class)) {
            var first = context.getBean(OrderService.class);
            var second = context.getBean(OrderService.class);

            System.out.println(first == second); // 기본 singleton: true
            first.order(12000);
        }
    }
}
```

이 생성자는 설정 클래스 등록과 컨텍스트 refresh를 수행한다. 예제는 `spring-context`가 클래스패스에 있는 환경을 전제로 한다. try-with-resources가 끝나면 컨텍스트를 닫아 관리 중인 singleton의 종료 처리를 수행한다.

---

## 5. BeanDefinition: 객체를 만들기 위한 설정 정보

BeanDefinition은 실제 빈 객체가 아니라 **빈을 어떻게 만들고 관리할지 표현한 메타데이터**다. 클래스, 팩토리 메서드, 스코프, 지연 초기화 여부, 주입할 값과 참조, 초기화·종료 메서드 등을 담을 수 있다.

```text
컴포넌트 스캔 / @Bean / XML 등
    ↓
BeanDefinition 등록
    ↓
정의 후처리·의존 관계 해석
    ↓
실제 객체 생성과 초기화
```

`OrderConfig`의 `orderService()`에서는 컨테이너가 팩토리 메서드와 인자 정보를 바탕으로 PaymentClient를 준비하고 OrderService를 만든다. 빈 정의를 등록했다고 그 순간 모든 객체가 만들어지는 것은 아니다.

| 확장 지점 | 다루는 대상 | 역할 예 |
| --- | --- | --- |
| BeanFactoryPostProcessor | 빈 정의 등 컨테이너 메타데이터 | 일반 빈 생성 전에 설정 정보 변경 |
| BeanPostProcessor | 생성 과정의 빈 인스턴스 | 초기화 전후 처리, 프록시 래핑 등 |

정의 후처리기에서 일반 빈을 성급하게 조회하면 예정된 후처리가 적용되기 전에 객체가 만들어질 수 있다. 설정 정보를 다루는 단계와 인스턴스를 다루는 단계를 구분해야 한다.

---

## 6. Bean Lifecycle: 생성자 호출 이후에도 할 일이 남는다

일반적인 singleton 빈 생성 경로를 단순화하면 다음과 같다. 생성자 주입에 필요한 의존성은 생성자 호출 전에 준비하고, 필드·setter 주입은 객체 생성 뒤 진행한다.

```text
BeanDefinition 준비 및 관련 후처리
    ↓
생성자 인자 해결 → 인스턴스 생성
    ↓
필드·setter 등 의존성 주입
    ↓
관련 Aware 콜백
    ↓
초기화 전 BeanPostProcessor
    └─ @PostConstruct도 이 단계의 후처리기가 호출
    ↓
InitializingBean.afterPropertiesSet()
    ↓
사용자 지정 init 메서드
    ↓
초기화 후 BeanPostProcessor → 필요하면 AOP 프록시 반환
    ↓
빈 사용
    ↓ 컨텍스트 정상 종료
@PreDestroy → DisposableBean.destroy() → 사용자 지정 destroy 메서드
```

순환 참조, 특수 후처리기, 스코프 등에 따른 세부 경로는 생략했다. Aware 콜백도 종류에 따라 처리 주체가 다르므로 모든 생명주기를 단일 목록으로 완전히 표현한 것은 아니다.

서로 다른 초기화 메서드가 등록되면 `@PostConstruct`, `afterPropertiesSet()`, 사용자 지정 init 메서드 순으로 호출한다. 종료 콜백은 `@PreDestroy`, `destroy()`, 사용자 지정 destroy 메서드 순이다. 같은 메서드를 여러 방식으로 지정했다고 중복 호출하는 것은 아니다.

생성자에서는 아직 필드 주입이 끝나지 않았고, `@PostConstruct` 시점에는 자기 자신의 AOP 프록시를 통한 호출을 기대해서도 안 된다. 이 차이는 DB 작업이나 비동기 처리를 초기화 코드에서 시작할 때 중요하다.

초기화에서는 필요한 자원을 준비하고, 종료에서는 스레드 풀·연결 같은 자원을 정리한다. 강제 프로세스 종료 시 종료 콜백까지 보장되지는 않는다. prototype 빈은 컨테이너가 생성·주입·초기화하지만 이후 종료 콜백은 자동 관리하지 않으므로 사용 측의 정리가 필요하다.

---

## 7. Proxy: 실제 객체 앞에서 공통 기능을 적용하기

Proxy는 실제 객체에 대한 접근을 대신 받는 객체다. Spring AOP는 프록시를 통해 메서드 호출 전후에 트랜잭션, 로깅 등의 부가 동작을 적용할 수 있다.

```text
호출자 → 프록시 → 부가 동작 → 실제 빈 메서드
                         ← 결과·예외 처리
```

모든 빈이 자동으로 AOP 프록시가 되는 것은 아니다. 적용할 advisor와 설정 등에 따라 프록시가 만들어진다. 그래서 컨테이너에서 조회한 객체나 주입받은 객체가 작성한 구현 클래스 그 자체라고 가정하면 안 된다.

| 방식 | 기반 | 주요 제약 |
| --- | --- | --- |
| JDK 동적 프록시 | 인터페이스 구현 | 인터페이스에 노출된 호출을 중심으로 사용하며 구현 클래스 타입으로 캐스팅할 수 없음 |
| CGLIB 프록시 | 대상 클래스 상속 | final 클래스는 상속 불가. final·private 메서드는 오버라이드 기반 advice 적용 불가 |

일반적인 Spring AOP 설정은 대상의 인터페이스 유무와 클래스 프록시 강제 설정에 따라 방식을 선택한다. Spring Boot 등 상위 설정이 기본값을 바꿀 수 있으므로 인터페이스가 있으면 반드시 JDK 프록시라고 단정하지 않는다.

### 같은 객체 안의 메서드 호출은 왜 문제가 될까?

```java
import org.springframework.transaction.annotation.Transactional;

class PaymentService {
    public void checkout() {
        this.pay();
    }

    @Transactional
    public void pay() {
        // 결제에 필요한 DB 변경
    }
}
```

이 코드는 PaymentService가 빈이고 트랜잭션 인프라가 활성화된 프록시 방식이라는 가정이다. 외부에서 `checkout()`을 호출해도 실제 객체 안의 `this.pay()`는 프록시를 다시 지나지 않는다. 따라서 **pay()의 애너테이션만으로 새 트랜잭션 경계가 적용되지 않는다.** 이미 진행 중인 트랜잭션이 있다면 그 안에서 실행될 수는 있다.

```text
외부 → 프록시 → checkout() → this.pay()
                              ↑ 프록시를 우회
```

해결할 때는 트랜잭션이 필요한 작업을 별도 빈으로 분리해 외부 호출로 만들거나, 실제 업무 단위인 외부 진입 메서드에 트랜잭션을 적용할 수 있는지 검토한다. 무조건 자기 자신을 주입하거나 프록시를 직접 조회하기보다 책임과 트랜잭션 경계를 먼저 정한다. AspectJ weaving은 프록시 방식과 동작 조건이 다르다.

---

## 전체 흐름 정리

```text
설정 클래스·컴포넌트 발견
    ↓
BeanDefinition 등록·후처리
    ↓
ApplicationContext가 내부 BeanFactory로 빈 생성 관리
    ↓
의존성 해결 → 인스턴스 생성 → 주입 → 초기화
    ↓
필요한 빈을 프록시로 감싸 호출자에 전달
    ↓
업무 메서드 실행 + 프록시를 통한 부가 동작
    ↓
컨텍스트 종료 시 관리 대상 자원 정리
```

IoC/DI는 객체를 연결하는 책임을 옮기는 원칙이고, BeanDefinition은 그 연결과 생성에 필요한 설정이다. 컨테이너는 설정을 바탕으로 빈의 생명주기를 관리하며 프록시는 호출 경로에 부가 기능을 적용한다. 이 역할을 나누면 주입 실패, 초기화 시점 문제, 트랜잭션 미적용을 서로 다른 원인으로 살펴볼 수 있다.

## 참고 자료

- [Spring Framework — IoC Container and Beans](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)
- [Spring Framework — The BeanFactory API](https://docs.spring.io/spring-framework/reference/core/beans/beanfactory.html)
- [Spring Framework — Container Extension Points](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)
- [Spring Framework — Bean Lifecycle Callbacks](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)
- [Spring Framework — Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)
