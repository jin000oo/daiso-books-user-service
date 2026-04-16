# User Service

> **Daiso Books** — MSA 기반 온라인 서점 플랫폼  
> User Service 개인 기여도: **60.6%**
>
> 🔗 [전체 프로젝트 Organization](https://github.com/nhnacademy-be12-daiso) | 📋 [API 문서 (Swagger UI)](docs/swagger.pdf)

---

## 서비스 개요

`user-service`는 Daiso Books MSA 플랫폼에서 **회원 도메인**을 담당하는 핵심 서비스입니다.  
회원 가입, 등급 관리, 포인트 정책 관리, 배송지 관리, 휴면 계정 처리, 서비스 간 내부 통신 API까지 포함합니다.

---

## 기술 스택

| 분류        | 스택                                               |
|-----------|--------------------------------------------------|
| Language  | Java 21                                          |
| Framework | Spring Boot 3.5.7                                |
| Cloud     | Spring Cloud 2025.0.0 (Eureka Client, OpenFeign) |
| ORM       | Spring Data JPA + QueryDSL 5.1.0                 |
| Database  | MySQL (운영) / H2 (테스트)                            |
| Cache     | Redis (Spring Data Redis) + Caffeine (로컬 캐시)     |
| Messaging | RabbitMQ (Spring AMQP)                           |
| Security  | Spring Security Crypto (BCrypt)                  |
| Mail      | Spring Mail (계정 복구)                              |
| Docs      | SpringDoc OpenAPI (Swagger UI)                   |
| Quality   | JaCoCo 0.8.10 + SonarQube                        |
| Build     | Maven                                            |

---

## 아키텍처 위치

```
[ API Gateway ]
      │  JWT 검증
      ▼
[ user-service ] ──OpenFeign──▶ [ 타 서비스 ]
      │
      ├── MySQL
      ├── Redis
      └── RabbitMQ
```

> JWT 검증은 **API Gateway** 레벨에서 처리됩니다.  
> `user-service`는 Gateway를 신뢰하고 순수 도메인 로직에 집중합니다.

---

## API 엔드포인트 (총 29개)

### 회원

| 메서드    | 엔드포인트                           | 설명                     |
|--------|---------------------------------|------------------------|
| `POST` | `/api/users/signup`             | 회원가입                   |
| `POST` | `/api/users/payco/login`        | PAYCO OAuth 로그인 / 회원가입 |
| `GET`  | `/api/users/check-id`           | 아이디 중복 확인              |
| `POST` | `/api/users/find-id`            | 아이디 찾기                 |
| `POST` | `/api/users/find-password`      | 임시 비밀번호 발급             |
| `GET`  | `/api/users/birthday`           | 생일 월로 사용자 조회           |
| `POST` | `/api/users/activate/send-code` | 휴면 해제 인증코드 발송          |
| `POST` | `/api/users/activate`           | 휴면 계정 활성화 (인증코드 검증)    |

### 마이페이지

| 메서드      | 엔드포인트                                 | 설명        |
|----------|---------------------------------------|-----------|
| `GET`    | `/api/users/me`                       | 내 정보 조회   |
| `PUT`    | `/api/users/me`                       | 내 정보 수정   |
| `DELETE` | `/api/users/me`                       | 회원 탈퇴     |
| `PUT`    | `/api/users/me/password`              | 비밀번호 변경   |
| `GET`    | `/api/users/me/addresses`             | 배송지 목록 조회 |
| `POST`   | `/api/users/me/addresses`             | 배송지 추가    |
| `PUT`    | `/api/users/me/addresses/{addressId}` | 배송지 수정    |
| `DELETE` | `/api/users/me/addresses/{addressId}` | 배송지 삭제    |
| `GET`    | `/api/users/me/points`                | 포인트 내역 조회 |

### 관리자

| 메서드      | 엔드포인트                                        | 설명           |
|----------|----------------------------------------------|--------------|
| `GET`    | `/api/admin/users`                           | 전체 회원 목록 조회  |
| `GET`    | `/api/admin/users/{userCreatedId}`           | 회원 상세 조회     |
| `PUT`    | `/api/admin/users/{userCreatedId}/status`    | 회원 상태 변경     |
| `PUT`    | `/api/admin/users/{userCreatedId}/grade`     | 회원 등급 변경     |
| `GET`    | `/api/admin/points/policies`                 | 포인트 정책 목록 조회 |
| `POST`   | `/api/admin/points/policies`                 | 포인트 정책 등록    |
| `PUT`    | `/api/admin/points/policies/{pointPolicyId}` | 포인트 정책 수정    |
| `DELETE` | `/api/admin/points/policies/{pointPolicyId}` | 포인트 정책 삭제    |

### 내부 통신

| 메서드    | 엔드포인트                                        | 설명              |
|--------|----------------------------------------------|-----------------|
| `GET`  | `/api/internal/users/{userCreatedId}/info`   | 주문/결제용 회원 정보 조회 |
| `GET`  | `/api/internal/users/{userCreatedId}/exists` | 회원 유효성 검증       |
| `POST` | `/api/internal/points`                       | 포인트 적립/사용 처리    |
| `POST` | `/api/internal/points/policy`                | 정책 기반 포인트 적립    |

---

## 핵심 기술적 결정

### 1. QueryDSL Projection — N+1 제거 (41쿼리 → 2쿼리)

- **문제 상황**  
  관리자 회원 목록 페이지는 회원 정보와 함께 계정 상태, 등급, 로그인 이력을 함께 조회합니다.  
  JPA 기본 페치 방식에서는 N명의 회원마다 추가 쿼리가 발생하여  
  **페이지 로드당 41개의 쿼리**가 실행되었습니다. (1 + 20 × 2)

- **원인**  
  `@OneToOne`, `@ManyToOne` 연관관계가 Lazy Loading으로 설정되어 엔티티 목록을 순회하면서 각 행마다 별도의 SELECT가 발생했습니다.

- **해결 방법**  
  엔티티 조회 대신 **QueryDSL Projections(DTO 직접 조회)**를 사용하여 필요한 필드만 단일 JOIN 쿼리로 가져오도록 변경했습니다.

#### Before: 엔티티 조회 → N+1 발생

```java
Page<UserResponse> users = userRepository.findAll(pageable)
        .map(user ->
                new UserResponse(user.getAccount().getLoginId(),
                        user.getUserName(),
                        user.getPhoneNumber(),
                        user.getEmail(),
                        user.getBirth(),
                        getGrade(user).getGradeName(),
                        pointService.getCurrentPoint(user.getUserCreatedId()).currentPoint(),
                        getStatus(user).getStatusName(),
                        user.getJoinedAt()));
```

#### After: Projection 조회 → 단일 쿼리

```java
// 데이터 조회 쿼리
List<UserResponse> content = jpaQueryFactory
                .select(Projections.constructor(UserResponse.class,
                        user.userCreatedId,
                        account.loginId,
                        user.userName,
                        user.phoneNumber,
                        user.email,
                        user.birth,
                        user.grade.gradeName,
                        user.currentPoint,
                        account.status.statusName,
                        account.joinedAt
                ))
                .from(user)
                .join(user.account, account)
                .join(user.grade)
                .join(account.status)
                .where(containsKeyword(criteria.keyword()))
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .orderBy(getOrderSpecifier(pageable.getSort()))
                .fetch();

// 카운트 쿼리
Long count = jpaQueryFactory
        .select(user.count())
        .from(user)
        .join(user.account, account)
        .where(containsKeyword(criteria.keyword()))
        .fetchOne();
```

- **결과**  
  쿼리 수 **41개 → 2개** (데이터 쿼리 1개 + count 쿼리 1개)로 감소.  
  관리자 페이지 응답 속도가 유의미하게 개선되었습니다.

---

### 2. 비관적 락(Pessimistic Lock) — 포인트 동시성 제어

- **문제 상황**  
  여러 요청이 동시에 포인트를 적립/사용할 경우 같은 잔액을 동시에 읽어 잘못된 값이 저장되는 문제가 발생할 수 있었습니다.

- **원인**  
  락 없이 두 트랜잭션이 동시에 같은 포인트 잔액을 읽고 각자 독립적으로 쓰면 한 쪽의 업데이트가 조용히 덮어쓰여집니다.

- **해결 방법**  
  DB 레벨의 **비관적 락**을 적용했습니다.  
  Redis 분산 락도 검토했으나 포인트 도메인은 DB 중심 처리라 과도한 인프라 추가를 피하기 위해 제외했습니다.

```java

@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")}) // 3초 대기 후 실패
@Query("SELECT u FROM User u WHERE u.userCreatedId = :userCreatedId")
Optional<User> findByIdForUpdate(Long userCreatedId);
```

- **결과**  
  포인트 동시 연산이 DB 레벨에서 완전히 직렬화되어 잘못된 값이 저장되거나 Phantom Read 발생 방지.

---

### 3. 휴면 계정 활성화 — Redis + 이메일 인증코드

- **문제 상황**  
  3개월 이상 미접속 회원은 `daiso-batch`에 의해 휴면 상태로 전환됩니다.  
  이메일 인증을 통해 안전하게 재활성화할 수 있어야 합니다.

- **해결 방법**
    1. 사용자가 활성화 요청 → `user-service`가 6자리 인증코드 생성
    2. 코드를 **Redis에 저장 (TTL = 5분)** → 자동 만료 처리
    3. 등록된 이메일로 Spring Mail을 통해 코드 발송
    4. 사용자가 코드 제출 → Redis 검증 → 계정 재활성화

- **결과**  
  인증 코드 검증 단계에서 DB 쓰기 없이 Stateless하게 처리.  
  TTL 기반 자동 만료로 별도 정리 배치 불필요.

---

## UI 스크린샷

| 화면        | 스크린샷                                                                    |
|-----------|-------------------------------------------------------------------------|
| 홈 (비로그인)  | ![홈-비로그인](docs/images/home.png)                                         |
| 홈 (로그인)   | ![홈-로그인](docs/images/home-login.png)                                    |
| 회원가입      | ![회원가입1](docs/images/signup1.png)<br/>![회원가입2](docs/images/signup2.png) |
| 로그인       | ![로그인](docs/images/login.png)                                           |
| 관리자 대시보드  | ![관리자-대시보드](docs/images/admin-dashboard.png)                            |
| 관리자 회원 목록 | ![관리자-회원관리](docs/images/admin-users.png)                                |
| 관리자 회원 검색 | ![관리자-회원관리-검색](docs/images/admin-users-search.png)                      |
| 관리자 회원 상세 | ![관리자-회원상세](docs/images/admin-user-details.png)                         |
| 포인트 정책 목록 | ![관리자-포인트정책관리](docs/images/admin-points.png)                            |
| 포인트 정책 수정 | ![관리자-포인트정책수정](docs/images/admin-point-update.png)                      |
| 포인트 정책 등록 | ![관리자-포인트정책등록](docs/images/admin-point-register.png)                    |
| 마이페이지     | ![마이페이지](docs/images/mypage.png)                                        |
| 내 정보 수정   | ![마이페이지-내정보수정](docs/images/mypage-update.png)                           |
| 비밀번호 변경   | ![마이페이지-비밀번호변경](docs/images/mypage-password.png)                        |
| 회원 탈퇴     | ![마이페이지-회원탈퇴](docs/images/mypage-withdraw.png)                          |
| 포인트 내역    | ![마이페이지-포인트내역](docs/images/mypage-points.png)                           |
| 배송지 관리    | ![마이페이지-배송지관리](docs/images/mypage-addresses.png)                        |
| 배송지 추가    | ![마이페이지-배송지추가](docs/images/mypage-address-register.png)                 |
| 배송지 수정    | ![마이페이지-배송지수정](docs/images/mypage-address-update.png)                   |
| 휴면 계정 활성화 | ![휴면계정활성화](docs/images/dormant.png)                                     |
| 활성화 메일 발송 | ![휴면계정활성화-메일발송](docs/images/mail-send.png)                              |
| 활성화 메일 본문 | ![휴면계정활성화-메일본문](docs/images/mail.png)                                   |

---

## 코드 품질

![sonarqube](docs/images/sonarqube.png)

| 지표                | 결과                  |
|-------------------|---------------------|
| 커버리지              | **71.2%** (1.1k 라인) |
| Security          | **A 등급**            |
| Reliability       | **A 등급**            |
| Maintainability   | **A 등급**            |
| Duplications      | **0.0%**            |
| Security Hotspots | 0                   |

> **JaCoCo 0.8.10** + **SonarQube** 기반 정적 분석 완료
