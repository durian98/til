# saveAll을 호출해도 INSERT가 배치로 실행되지는 않는다

> 2026-10-15

## 배운 것

Spring Data JPA의 `save()`는 엔티티가 새 객체라고 판단하면 `persist()`, 기존 객체라고 판단하면 `merge()`를 호출한다. 기본 판정에서는 nullable한 비원시 타입 `@Version` 속성을 먼저 보고, 해당 속성이 없으면 ID의 null 여부를 본다. `Persistable`을 구현하면 `isNew()`로 판정을 제공할 수 있다. [Spring Data JPA 문서](https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html).

따라서 UUID를 미리 부여한 신규 객체는 버전 속성이나 별도 새 엔티티 판정이 없으면 merge 경로로 들어갈 수 있다. 이번 엔티티에서는 저장 전에 존재 확인 SELECT가 발생했다. 모든 merge가 언제나 SELECT 한 번을 발생시킨다는 뜻은 아니다.

`saveAllAndFlush()`는 여러 엔티티를 저장하고 flush하는 API다. 호출 한 번이라는 사실만으로 JDBC batch가 적용됐다고 판단할 수 없다. 실제 batch 설정과 실행 횟수, 저장 결과를 확인해야 한다.

## 막혔던 것 / 해결

추천 후보 200개를 저장하는 단계가 느려 SQL을 확인했다. 새 후보의 UUID가 이미 할당돼 있었고, 후보 저장에 SELECT 200회와 개별 INSERT 200회가 발생했다.

수정을 두 단계로 나눠 비교했다.

1. 새 후보만 받는 전용 저장 경로에서 `EntityManager.persist()`를 사용해 존재 조회를 제거했다.
2. 해당 Hibernate Session의 batch size를 50으로 설정하고 flush한 뒤 원래 설정으로 복원했다.

아래는 기존 트랜잭션 안에서 **새 엔티티만** 저장하는 부분의 예시다.

```java
Session session = entityManager.unwrap(Session.class);
Integer previous = session.getJdbcBatchSize();
try {
    session.setJdbcBatchSize(50);
    for (Candidate candidate : newCandidates) {
        entityManager.persist(candidate);
    }
    entityManager.flush();
} finally {
    session.setJdbcBatchSize(previous);
}
```

Session 단위 batch size 변경은 [Hibernate 문서](https://docs.hibernate.org/orm/6.6/userguide/html_single/#batch)에 설명돼 있다. 기존 엔티티 수정까지 이 경로로 옮기거나, 성능 때문에 트랜잭션·flush 순서를 임의로 바꾸면 안 된다. ID 생성 전략과 SQL 형태도 batching에 영향을 준다.

### 로컬 비교 결과

측정일은 2026-10-09다. 백엔드 2 CPU/4 GiB, PostgreSQL 1 CPU/2 GiB인 로컬 환경에서 동일 사용자·여행 다섯 쌍을 사용했다. 각 방식은 준비 세 건을 제외하고 20건씩 측정했다.

| 저장 방식 | 후보 저장 p95 | 후보 저장 JDBC 실행 |
| --- | ---: | --- |
| 기존 saveAllAndFlush | 171.59ms | SELECT 200 + 개별 INSERT 200 |
| 신규 후보 persist | 123.91ms | SELECT 0 + 개별 INSERT 200 |
| persist + batch size 50 | 52.52ms | SELECT 0 + executeBatch 4회, 저장 200행 |

`executeBatch` 네 번은 INSERT 행이 네 개가 됐거나 네트워크 왕복이 정확히 네 번이라는 뜻이 아니다.

속도와 함께 후보 값·순위·코스 내용 보존, 중복·FK 오류, 일정·작업·후보·이벤트의 전체 롤백, Session 설정 복원을 검사했다. 실험에서 첫 코스 조회 p95는 세 방식이 비슷했다. 저장 구간의 개선율을 사용자 전체 응답의 개선율로 바꿔 쓰지 않는다.

로컬 실험 변경이며 운영 적용은 별도다. 정식 반영 전에 재추천·재시도에서도 이 경로가 신규 후보만 받는지 확인해야 한다.

## 참고

- [Spring Data JPA: 엔티티 저장과 신규 판정](https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html)
- [Hibernate: JDBC batching](https://docs.hibernate.org/orm/6.6/userguide/html_single/#batch)
