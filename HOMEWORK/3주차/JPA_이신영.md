# 탐구 1. 트랜잭션이 없는 경우에 EM이 준영속(detached)상태가 될까?

트랜잭션이 종료된다면 영속성 컨텍스트는 비활성화되어서 EM이 detached 상태로 남아는다. dirty checking과 동기화가 발생하지않지만 여전히 **존재는 한다**

여기서 트랜잭션이 아예 없는경우와 종료된 경우를 조금 헷갈릴 수 있는데 아래와 같음

### 트랜잭션 없는 경우
EntityManager는 여전히 활성 상태. 엔티티를 영속화하거나 조회할 수 있지만, 트랜잭션이 없기 때문에 변경된 엔티티는 데이터베이스에 반영되지 않는다.

### 트랜잭션 종료 후
EntityManager가 관리하던 엔티티는 준영속(detached) 상태로 전환되고 변경 사항을 데이터베이스에 반영하려면 엔티티를 다시 영속화해야 한다(merge() 사용)


# 탐구 2. 1차캐시 발생 과정

## EM 요약
```java
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();

tx.begin(); // 트랜잭션 시작

// 엔티티 생성 및 영속성 컨텍스트에 저장
Member member = new Member();
member.setId(1L);
member.setName("John");

em.persist(member); // 영속성 컨텍스트에 저장 (INSERT 준비)
System.out.println("Persist 이후: 데이터베이스에 저장되지 않음");

// 트랜잭션 커밋
tx.commit(); // 데이터베이스에 INSERT 쿼리 실행
System.out.println("Commit 이후: 데이터베이스에 저장됨");

// EntityManager 종료
em.close(); // 영속성 컨텍스트 종료, member는 준영속 상태로 전환

```

- find() : 찾기
- persist() : 영속성 컨텍스트에 저장
- commit() : EM 트랜잭션 종료 + 영속성컨텍스트와 데이터베이스를 동기화(flush 있음)
- close() : EM 종료

## 그래서 1차캐시는?

```java
// 1차 캐시 확인을 위해 동일한 EntityManager 사용
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin(); // 트랜잭션 시작

// 처음 조회: 1차 캐시에 없음 → 데이터베이스에서 조회 후 1차 캐시에 저장
Member member1 = em.find(Member.class, 1L); 
System.out.println("조회된 회원 이름: " + member1.getName()); 

// 동일 엔티티를 다시 조회: 1차 캐시에서 바로 반환 (DB 접근 없음)
Member member2 = em.find(Member.class, 1L); 
System.out.println("다시 조회된 회원 이름: " + member2.getName());

// 동일성 확인: 같은 객체임을 보장
System.out.println(member1 == member2); // true 출력됨

tx.commit(); // 트랜잭션 커밋
em.close(); // EntityManager 닫기

```

jpa의 1차 캐시는 EM을 통해 엔티티를 조회하거나 저장할 때 이루어진다. 

영속성 컨텍스트는 EM의 내부에 존재하고 EM 자체가 1차캐시를 관리하는건 아니다.

## 헷갈리던거 정리

1. 트랜잭션
- 데이터베이스 작업(INSERT, UPDATE, DELETE 등)의 논리적 작업 단위.
- 하나의 트랜잭션은 데이터베이스 작업을 일관되게 처리하고, 중간에 오류가 발생하면 작업을 롤백(취소)합니다.
- 데이터베이스와 JPA 작업을 묶는 "껍질" 같은 존재.
- JPA의 commit()과 rollback()로 트랜잭션의 상태를 변경가능

2. EntityManager (EM)
- 영속성 컨텍스트를 관리하는 JPA의 핵심 객체.
- 개발자가 직접 사용하는 인터페이스로, 엔티티 저장, 조회, 삭제 등의 작업을 수행
- EntityManager는 각각 고유한 영속성 컨텍스트를 가진다.
- JPA는 트랜잭션의 시작과 종료를 EntityManager와 연계하여 처리한다.
- 즉, EntityManager는 영속성 컨텍스트를 간접적으로 관리한다.

3. 영속성 컨텍스트
- JPA에서 엔티티를 캐시하고, 상태를 관리하며, 데이터베이스 동기화를 담당하는 메모리 공간.
- 1차 캐시, 변경 감지(dirty checking), 플러시(flush), 지연 로딩 같은 JPA의 핵심 기능은 모두 영속성 컨텍스트를 통해 동작.
- 영속성 컨텍스트는 EntityManager 내부에 존재하며, EntityManager를 통해 접근됩니다.
- 하나의 EntityManager = 하나의 영속성 컨텍스트

EntityManagerFactory > EntityManager > Persistence Context > 1차 캐시, 엔티티 상태 관리 > 트랜잭션 -> db와 소통
