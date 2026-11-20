# restAPI_Mission

Spring Boot REST API 연습 프로젝트입니다.
[p-14172-2](https://github.com/jhs512/p-14172-2)의 `068` 커밋(글/댓글 CRUD + Swagger 문서화)을 기반으로,
[강의 4부 연습문제(69강 ~ 79강)](https://www.slog.gg/p/14172)의 **회원 기능과 작성자 권한 체크**를 TDD로 구현했습니다.

## 기술 스택

| 구분 | 사용 기술 |
|---|---|
| Language | Java 25 |
| Framework | Spring Boot 4.1.2 (Web MVC, Data JPA, Validation) |
| Database | H2 (dev: 파일 DB, test: 인메모리 DB) |
| API 문서 | springdoc-openapi (Swagger UI) |
| Test | JUnit 5, MockMvc, AssertJ |
| Build | Gradle (Kotlin DSL) |

## 실행 방법

```bash
# 애플리케이션 실행 (dev 프로필, http://localhost:8080)
./gradlew bootRun

# 전체 테스트 실행
./gradlew test
```

- Swagger UI : http://localhost:8080/swagger-ui/index.html
- H2 Console : http://localhost:8080/h2-console
  - JDBC URL : `jdbc:h2:./db_dev` / User Name : `sa` / Password : (없음)

## 구현 요구사항

- 로그인/로그아웃은 구현하지 않으며, 요청자를 `?actorId={회원번호}` 쿼리 파라미터로 식별합니다.
- `actorId`가 필요한 액션 : **내 정보, 글 작성·수정·삭제, 댓글 작성·수정·삭제**
- `actorId`가 없거나 존재하지 않는 회원이면 **401**, 글/댓글의 작성자가 아니면 **403** 을 응답합니다.

| 문제 | 내용 | 상태 |
|---|---|---|
| 1 | 회원가입 API (username 중복 시 409) | ✅ |
| 2 | 내 정보 API (응답에 비밀번호 미포함) | ✅ |
| 3 | 샘플 데이터 생성 시 회원 6명 생성 | ✅ |
| 4 | 모든 글과 댓글은 작성자(`author`)를 가진다 | ✅ |
| 5 | 글 작성 시 작성자 정보 필요 | ✅ |
| 6 | 댓글 작성 시 작성자 정보 필요 | ✅ |
| 7 | 글 삭제 시 작성자 본인이어야 함 | ✅ |
| 8 | 글 수정 시 작성자 본인이어야 함 | ✅ |
| 9 | 댓글 삭제 시 작성자 본인이어야 함 | ✅ |
| 10 | 댓글 수정 시 작성자 본인이어야 함 | ✅ |

## API 명세

### 회원

| Method | URL | 설명 | actorId |
|---|---|---|:---:|
| POST | `/api/v1/members/join` | 회원가입 | |
| GET | `/api/v1/members/me` | 내 정보 | 필요 |

**회원가입 요청 본문**

```json
{ "username": "user5", "password": "12345678", "name": "유저5" }
```

- 제약 : `username` 4~30자(유니크), `password` 8~30자, `name` 2~30자
- 성공 : `201` `{"resultCode": "201-1", "msg": "유저5님 환영합니다. 회원가입이 완료되었습니다.", "data": null}`

**내 정보 응답** (`MemberDto`, 비밀번호 제외)

```json
{ "id": 3, "createDate": "...", "modifyDate": "...", "username": "user1", "name": "유저1" }
```

### 글

| Method | URL | 설명 | actorId | 권한 |
|---|---|---|:---:|---|
| GET | `/api/v1/posts` | 다건 조회 | | |
| GET | `/api/v1/posts/{id}` | 단건 조회 | | |
| POST | `/api/v1/posts` | 작성 | 필요 | |
| PUT | `/api/v1/posts/{id}` | 수정 | 필요 | 작성자 본인 |
| DELETE | `/api/v1/posts/{id}` | 삭제 | 필요 | 작성자 본인 |

### 댓글

| Method | URL | 설명 | actorId | 권한 |
|---|---|---|:---:|---|
| GET | `/api/v1/posts/{postId}/comments` | 다건 조회 | | |
| GET | `/api/v1/posts/{postId}/comments/{id}` | 단건 조회 | | |
| POST | `/api/v1/posts/{postId}/comments` | 작성 | 필요 | |
| PUT | `/api/v1/posts/{postId}/comments/{id}` | 수정 | 필요 | 작성자 본인 |
| DELETE | `/api/v1/posts/{postId}/comments/{id}` | 삭제 | 필요 | 작성자 본인 |

### 공통 에러 응답

| 상태 | resultCode | msg | 발생 조건 |
|---|---|---|---|
| 400 | `400-1` | 필드별 검증 메시지 / `요청 본문이 올바르지 않습니다.` | 요청 본문 검증 실패, JSON 형식 오류 |
| 401 | `401-1` | `로그인 후 이용해주세요.` | `actorId` 누락 또는 존재하지 않는 회원 |
| 403 | `403-1` | `권한이 없습니다.` | 글/댓글 수정·삭제 시 작성자가 아님 |
| 404 | `404-1` | `해당 데이터가 존재하지 않습니다.` | 존재하지 않는 글/댓글 |
| 409 | `409-1` | `{username}(은)는 이미 사용중인 username 입니다.` | 회원가입 시 username 중복 |

예외 클래스는 `GlobalExceptionHandler` 내부의 정적 중첩 클래스로 정의되어 있습니다.

- `GlobalExceptionHandler.MemberDuplicateUsernameException`
- `GlobalExceptionHandler.UnauthenticatedException`
- `GlobalExceptionHandler.AccessDeniedException`

## 샘플 데이터

애플리케이션 시작 시 `BaseInitData`가 데이터가 없을 때만 생성합니다. 모든 회원의 비밀번호는 `12345678` 입니다.

| id | username | name |
|:---:|---|---|
| 1 | system | 시스템 |
| 2 | admin | 관리자 |
| 3 | user1 | 유저1 |
| 4 | user2 | 유저2 |
| 5 | user3 | 유저3 |
| 6 | user4 | 유저4 |

| 글 | 작성자 | 댓글 (작성자) |
|---|---|---|
| 1번 글 | user1 | 1번 (user1), 2번 (user2), 3번 (user3) |
| 2번 글 | user2 | 4번 (user1), 5번 (user2) |
| 3번 글 | user3 | - |

**사용 예시** : 1번 글은 user1(`actorId=3`)만 수정·삭제할 수 있습니다.

```bash
# 성공 (200)
curl -X DELETE "http://localhost:8080/api/v1/posts/1?actorId=3"

# 작성자가 아님 (403)
curl -X DELETE "http://localhost:8080/api/v1/posts/1?actorId=4"

# actorId 누락 (401)
curl -X DELETE "http://localhost:8080/api/v1/posts/1"
```

## 설계 포인트

- **인증 → 조회 → 권한 → 실행** 순서로 처리합니다.
  - 인증 : `MemberService.getActor(actorId)` 가 회원을 찾지 못하면 `UnauthenticatedException`(401)
  - 권한 : `Post` / `PostComment` 엔티티의 `checkActorCanModify` · `checkActorCanDelete` 가 작성자가 아니면 `AccessDeniedException`(403)
- `@RequestParam(required = false) Integer actorId` 로 받아, 파라미터 누락 시 400이 아닌 401을 직접 응답합니다.
- `author`는 `@ManyToOne(fetch = LAZY)` 이므로 Hibernate 프록시와의 `equals` 비교 문제를 피하기 위해 **id로 작성자를 비교**합니다.
- 회원 응답은 `MemberDto` 를 사용해 비밀번호가 API에 노출되지 않도록 했습니다.
- username 중복 검사는 `MemberService.join` 에 두어, 컨트롤러뿐 아니라 샘플 데이터 생성 시에도 동일한 규칙이 적용됩니다.

## 프로젝트 구조

```
src/main/java/com/back
├── domain
│   ├── member/member        # 회원 (controller, dto, entity, repository, service)
│   └── post
│       ├── post             # 글 (controller, dto, entity, repository, service)
│       └── postComment      # 댓글 (controller, dto, entity)
└── global
    ├── aspect               # RsData resultCode → HTTP 상태코드 변환
    ├── globalExceptionHandler
    ├── initData             # 샘플 데이터 생성
    ├── jpa                  # BaseEntity (id, createDate, modifyDate)
    ├── rsData               # 공통 응답 객체
    ├── springDoc
    └── webMvc
```

## 테스트

`./gradlew test` 기준 **27개 테스트 통과**

| 테스트 클래스 | 개수 | 주요 검증 |
|---|:---:|---|
| `ApiV1MemberControllerTest` | 6 | 회원가입, username 중복(409), 내 정보, 401, 샘플 회원 |
| `ApiV1PostControllerTest` | 13 | 글 CRUD, 검증(400), 404, 샘플 작성자, 401, 403 |
| `ApiV1PostCommentControllerTest` | 8 | 댓글 CRUD, 401, 403 |

## 커밋 이력

문제 단위로 커밋을 나누고 `001`, `002` … 형식의 태그를 붙였습니다.

| 커밋 | 내용 |
|---|---|
| 000 | 068 커밋 기반 프로젝트 세팅 |
| 001 | ApiV1MemberController, MemberService, MemberRepository, Member 추가 |
| 002 | 회원가입 테스트케이스 추가(실패) |
| 003 | 회원가입 기능 구현 |
| 004 | 회원가입 username 중복 체크(409) |
| 005 | 내 정보 API 구현(MemberDto, UnauthenticatedException) |
| 006 | 샘플 데이터 생성 시 회원 6명 생성 |
| 007 | 글/댓글에 작성자(author) 추가 |
| 008 | 글 작성 시 actorId로 작성자 지정(401 처리) |
| 009 | 댓글 작성 시 actorId로 작성자 지정(401 처리) |
| 010 | 글 삭제 시 작성자 본인 확인(403 처리) |
| 011 | 글 수정 시 작성자 본인 확인(403 처리) |
| 012 | 댓글 삭제 시 작성자 본인 확인(403 처리) |
| 013 | 댓글 수정 시 작성자 본인 확인(403 처리) |
