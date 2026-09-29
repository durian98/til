# `@Async` 작업은 DB 커밋을 기다리지 않는다

> 2026-09-29

## 배운 것

작업 행을 저장한 뒤 `@Async` 메서드를 호출해도, **호출자의 트랜잭션이 아직 열려 있으면** 별도 스레드가 미커밋 행을 먼저 조회할 수 있다. `save()` 호출과 커밋 완료는 같은 뜻이 아니다.

## 막혔던 것 / 해결

트랜잭션이 끝난 뒤 실행기에 넘기려면 저장을 담당한 별도 서비스의 트랜잭션이 반환된 다음 호출하거나, 트랜잭션 이벤트를 `AFTER_COMMIT`에 처리할 수 있다. 후자는 다음처럼 연결한다.

```java
@Transactional
public UUID submit() {
    Job job = jobs.save(new Job());
    events.publishEvent(new JobReady(job.getId()));
    return job.getId();
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void dispatch(JobReady event) {
    worker.run(event.id()); // 다른 Spring 빈의 @Async 메서드
}
```

`@Async`는 기본적으로 프록시를 거친 호출에 적용된다. 같은 객체 안에서 `this.run()`을 부르면 비동기 실행이 되지 않으므로 작업자는 별도 빈으로 둔다. 실행기가 요청을 거부하거나 작업 중 예외가 나면 작업 상태를 실패로 남겨, 조회 화면이 계속 기다리지 않게 해야 한다.

## 참고

- [Spring Framework: 트랜잭션에 묶인 이벤트](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
- [Spring Framework: 비동기 메서드와 프록시](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
