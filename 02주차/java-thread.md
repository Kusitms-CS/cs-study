# Java Thread (Backend 심화)

자바의 정석 3rd edition을 참고하였습니다.

- 하나의 스레드에 대해 `start()`를 두 번 이상 호출할 경우 `IllegalThreadStateException`이 발생함
  - `t.interrupt();`를 통해서 적절히 중단시켜 줘야 함
- 스레드는 독립적인 작업을 수행하기 위한 자신의 호출 스택을 필요로 하는데, call stack은 thread가 생성될 때마다 생성되며 종료 시 함께 소멸된다.
- 콜 스택은 스케줄러가 부여한 순서에 의해 번갈아 가며 실행되며, user thread가 없는 경우 프로그램은 종료된다.

## 스레드 생성과 사용

```java
// 1. Thread 클래스 상속
class MyThread extends Thread {
    public void run() {}
}

// 2. Runnable 인터페이스의 구현
class MyThread1 implements Runnable {
    public void run() {}
}
```

- case 2의 경우에는 재사용성이 높고 일관성 유지가 가능하다.
- `MyThread1`을 실사용할 경우에는 생성자의 매개변수로 넘겨야 한다.

```java
Thread thread = new Thread(new MyThread1());

class Main {
    public static void main(String[] args) {
        Thread threadA = new Thread(new MyThread());
        threadA.start(); // 스레드 실행
        threadA.run();   // 클래스 내부의 메서드 호출
    }
}
```

- 위처럼 `run`을 호출하게 되면 상속을 통한 오버라이딩을 피하고서도 외부의 `run()`을 제공받을 수 있게 된다.
- `run`의 내용을 채우는 것이 곧 스레드를 구현하는 것과 같다.

### 우선순위 (priority)

- 스레드는 `priority`라는 변수를 가지고 있는데, 우선순위의 값에 따라 실행 시간이 달라진다.
  - 예: 파일 전송 서비스의 경우 다운로드보다 채팅을 전송하는 스레드의 priority가 더 높아야 불편함이 없다.
- 스레드를 호출하기 전에만 우선순위를 변경할 수 있다.
- 다만 멀티 코어의 경우 OS마다 다른 방식으로 스케줄링하기 때문에 우선순위에 차등을 두지 않는 경우도 존재한다. 차라리 `PriorityQueue`에 저장해 두고 우선순위가 높은 작업이 처리되도록 하는 것이 나을 수도 있다.

## ThreadGroup

- 서로 관련된 스레드를 그룹으로 다루기 위한 것으로 폴더 개념과 유사하다.
  - 스레드 그룹 내부에 그룹을 포함시킬 수 있다.
- 보안상의 이유로 도입되었으며, 소속 스레드 그룹 및 하위 스레드 그룹은 변경 가능하나 타 스레드를 변경할 수는 없다.
- 모든 스레드는 반드시 스레드 그룹에 포함되어야 하며, 생성자를 사용하지 않은 스레드는 기본적으로 자신을 생성한 스레드와 같은 스레드 그룹에 속하게 된다.
- JVM은 `main`과 `system`이라는 스레드 그룹을 기본적으로 생성하며, `main` 스레드는 main 스레드 그룹 내에, 가비지 컬렉션을 수행하는 Finalizer 스레드는 system 그룹에 속한다.
- 따라서 생성하는 모든 스레드 그룹은 main의 하위 그룹이 되며, 지정하지 않고 생성한 스레드는 main 스레드 그룹 내에 존재하게 된다.

## Daemon Thread

- 데몬 스레드는 일반 스레드의 작업을 보조하는 스레드이다.
- 일반 스레드가 종료되면 자동 종료되며, 가비지 컬렉터 / 워드프로세서의 자동 저장 / 화면 자동 갱신을 예시로 들 수 있다.
- 무한 루프와 조건문을 활용하여 대기하다가 조건이 만족되면 작업을 수행하고 다시 대기한다.
- 데몬 스레드가 생성한 스레드는 데몬 스레드가 된다.

## 스레드의 생명 주기

`new`로 생성됨 → `start()`로 실행 대기열에 저장하여 대기(queue에 저장, RUNNABLE) → 실행

- 실행 시간이 다 되거나 `yield`를 만나는 경우 실행 대기로 돌아감
- `suspend`, `sleep`, `wait`, `join`, I/O block에 의하여 일시 정지가 된다면 WAITING / BLOCKED 상태로 존재하며, 일시 정지가 끝나면 실행 대기로 돌아감
- 일시 정지 상태에서 time-out되거나 `notify`, `resume`, `interrupt`의 호출로 일시 정지 상태를 벗어나 다시 실행 대기열에 저장된다.
- 실행을 마치거나 `stop`이 호출되는 경우에 thread는 소멸된다.

## 스레드의 동기화

싱글 스레드의 경우에는 문제가 되지 않지만, 멀티 스레드의 경우 자원을 공유해 작업하기 때문에 제어권이 다른 스레드에게 넘어갔을 때 데이터를 타 스레드가 변경하였다면, 다시 원래 스레드가 제어권을 받아 작업을 이어갈 때 원래 의도와는 다른 결과를 얻게 된다.

critical section & lock으로 해결 가능한데, 공유 데이터를 사용하는 영역을 **임계 영역**으로 지정해 두고 lock을 획득한 하나의 스레드만 영역 내의 코드를 수행할 수 있도록 한다. lock을 반납하면 다음으로 lock을 획득한 스레드가 임계 영역의 코드를 수행할 수 있게 된다. 이 개념을 **동기화(synchronization)** 라고 한다.

### synchronized를 이용한 동기화

```java
// 메서드 전체를 임계 영역으로 지정
public synchronized void calcSum() {
    // 임계 영역
}

// 특정 영역만 임계 영역으로 지정
public void calcSum() {
    synchronized (this) {
        // 임계 영역
    }
}
```

이 외에도 다음과 같은 방식이 있다.

- `wait` & `notify`를 이용한 동기화
- `lock` & `condition`을 이용한 동기화

## 자바의 외부 프로세스 실행

- `java.lang.Process` 클래스를 통해서 외부 프로세스와의 접점을 제공한다.
- `Process` 클래스는 `java.lang.Runtime` 혹은 `java.lang.ProcessBuilder`를 통해 얻을 수 있다.

참고: [네이버 D2 - Java에서 외부 프로세스 실행하기](https://d2.naver.com/helloworld/1113548)
