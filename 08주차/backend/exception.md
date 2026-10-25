# Exception

## Throwable 계층 구조

```text
Throwable
 ├─ Error                      : 시스템 수준의 심각한 문제, 애플리케이션에서 복구 불가
 │   ├─ OutOfMemoryError
 │   └─ StackOverflowError
 └─ Exception                  : 애플리케이션에서 처리할 수 있는 문제
     ├─ IOException, SQLException ...   ← Checked Exception
     └─ RuntimeException                ← Unchecked Exception
         ├─ NullPointerException
         ├─ IllegalArgumentException
         ├─ IllegalStateException
         └─ IndexOutOfBoundsException
```

- **Error**: JVM 메모리 부족, 스택 오버플로처럼 애플리케이션이 잡아서 해결할 수 없는 문제다. `catch`하지 않는다.
- **Exception**: 프로그램 로직에서 발생하며 처리할 수 있는 문제다.

## Checked와 Unchecked Exception

| | Checked Exception | Unchecked Exception |
| --- | --- | --- |
| 대상 | `Exception`의 하위 중 `RuntimeException`이 아닌 것 | `RuntimeException`과 그 하위 |
| 처리 강제 | **컴파일러가 강제** (`try-catch` 또는 `throws` 필수) | 강제하지 않음 |
| 확인 시점 | 컴파일 타임 | 런타임 |
| 예시 | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |
| 의도 | 호출자가 **복구할 수 있는** 상황 | 대부분 **프로그래밍 오류** |

Checked Exception의 문제

- 처리할 수 없는 예외도 `throws`로 계속 올려야 해서, 상위 계층(Service, Controller)이 하위 기술(JDBC의 `SQLException`)에 **의존**하게 된다.
- 억지로 처리하려다 `catch (Exception e) {}`처럼 **예외를 삼키는** 코드가 생기기 쉽다.

그래서 최근에는 **Unchecked Exception을 기본으로 사용**하는 추세다. Spring도 `SQLException`을 `DataAccessException`(Unchecked) 계층으로 변환해서 던진다.

## 예외 처리 방법

**try-catch-finally**

```java
try {
    // 예외가 발생할 수 있는 코드
} catch (IllegalArgumentException | IllegalStateException e) {   // 멀티 catch (Java 7+)
    // 처리
} finally {
    // 예외 발생 여부와 관계없이 실행 (자원 해제 등)
}
```

- `catch`는 위에서부터 순서대로 확인하므로 **하위 타입을 먼저** 써야 한다.
- `finally` 안에서 `return`하면 `try`의 예외가 사라지므로 하지 않는다.

**try-with-resources (Java 7+)**

```java
try (BufferedReader br = new BufferedReader(new FileReader("a.txt"))) {
    return br.readLine();
}   // 블록이 끝나면 자동으로 br.close() 호출
```

- `AutoCloseable`을 구현한 자원은 자동으로 닫힌다. 여러 자원은 선언 역순으로 닫힌다.
- `finally`에서 `close()`하다 예외가 나면 원래 예외가 묻히는 문제가 있었는데, try-with-resources는 `close()` 중 발생한 예외를 **suppressed exception**으로 원래 예외에 붙여 보존한다.

**예외 전환과 cause**

```java
try {
    repository.save(member);
} catch (SQLException e) {
    throw new MemberSaveException("회원 저장 실패", e);   // 원인 예외(cause)를 꼭 넘긴다
}
```

- 예외를 다른 예외로 감쌀 때 원인을 넘기지 않으면 **스택 트레이스가 끊겨** 실제 원인을 찾을 수 없다.

## 예외 처리 시 주의할 점

- **예외를 삼키지 않는다.** `catch (Exception e) {}`나 `e.printStackTrace()`만 하고 넘어가지 않는다. 최소한 로그를 남기고, 처리할 수 없다면 다시 던진다.
- **너무 넓게 잡지 않는다.** `catch (Exception e)`는 예상하지 못한 예외까지 잡아버린다.
- **흐름 제어에 예외를 쓰지 않는다.** 예외 생성은 스택 트레이스를 채우는 비용이 크다. 값이 없을 수 있는 정상 상황이면 `Optional`이나 반환값으로 표현한다.
- **의미 있는 예외를 쓴다.** 표준 예외(`IllegalArgumentException`, `IllegalStateException`)로 충분하면 재사용하고, 도메인 의미가 필요하면 커스텀 예외를 만든다.

## Spring에서의 예외 처리

**@Transactional 롤백 규칙**

- 기본적으로 **Unchecked Exception(`RuntimeException`)과 `Error`는 롤백**, **Checked Exception은 커밋**한다.
- Checked Exception은 "비즈니스적으로 의미 있는 결과"로 보고 복구 가능하다고 가정하기 때문이다.
- Checked Exception에도 롤백하려면 `@Transactional(rollbackFor = Exception.class)`를 지정한다.
- 메서드 안에서 예외를 `catch`하고 던지지 않으면 트랜잭션은 예외를 모르므로 롤백되지 않는다.

**전역 예외 처리**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException e) {
        return ResponseEntity.status(e.getErrorCode().getStatus())
                .body(ErrorResponse.of(e.getErrorCode()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.internalServerError().body(ErrorResponse.of(ErrorCode.INTERNAL_ERROR));
    }
}
```

- 컨트롤러마다 `try-catch`를 쓰지 않고, 예외 종류별로 HTTP 상태 코드와 응답 형식을 한곳에서 통일한다.
- 커스텀 예외에 `ErrorCode` enum(상태 코드, 에러 코드, 메시지)을 담는 패턴을 많이 쓴다.
