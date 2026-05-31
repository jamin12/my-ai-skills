---
name: domain-model
description: 도메인 모델 규칙(Private constructor, Factory method, 불변성)과 Value Object(@JvmInline value class) 작성 규칙. 도메인 모델 생성/수정 시 참조한다.
---

# 도메인 모델 & Value Object 규칙

## 규칙 1: Private Constructor + Factory Method

도메인 모델은 반드시 private constructor를 사용하고, companion object에 factory method를 제공한다.

- `create()`: 새로운 엔티티 생성 (id 없음)
- `restore()`: DB에서 복원 (id 있음)

### 생성 factory 이름은 도메인 언어를 따른다 (`create` 남발 금지)

생성 factory의 이름은 "현실에서 그 객체가 **어떻게 생겨나는가**"를 표현한다. 단순 `create`를 기본 반사로 쓰지 말고, 같은 의미 계열이면 같은 동사를 재사용해 일관성을 지킨다. (`restore()`는 항상 그대로 — 영속화 복원 전용)

| 동사 | 의미 계열 | 현재 적용 |
|------|----------|-----------|
| `register` | 시스템에 가입/등록되는 주체 | Member |
| `issue` | 발급자가 있고 만료·회수·소진 라이프사이클을 갖는 발급물(토큰·코드) | RefreshToken, PairingInvite |
| `link` | 두 대상을 잇는 관계 엔티티 | SocialIdentity |
| `create` | 발급/가입/연결 의미가 없는 평범한 엔티티 (**기본값**) | Couple, Profile |
| `of` | 이미 결정된 값들을 담기만 하는 단순 래퍼/캐리어 | AccessToken, IssuedRefreshToken, VerifiedSocialUser |
| `generate` | 무에서 값을 만들어내는 VO (id·코드·랜덤 닉네임) | 모든 *Id, InviteCode, Nickname |

**기본값(fallback)**: 마땅한 도메인 동사가 없으면 — 엔티티는 `create`, 값 래퍼는 `of`, 무에서 만드는 VO는 `generate`.

**확장 규칙**: 새 모델을 추가할 때 위 의미 계열에 들어맞으면 **새 동사를 만들지 말고 기존 동사를 재사용**한다(예: 새 토큰류 = `issue`). 어느 계열에도 안 맞는 새 동사가 정말 필요하면, 그때 이 표에 한 줄 추가하고 의미를 적는다 — 임의 작명은 금지.

// WRONG - public constructor 직접 노출
```kotlin
class Alert(
    val id: AlertId? = null,
    val name: String,
    val price: BigDecimal,
)
```

// CORRECT - private constructor + factory
```kotlin
class Alert private constructor(
    val id: AlertId? = null,
    val name: String,
    val price: BigDecimal,
) {
    companion object {
        fun create(name: String, price: BigDecimal): Alert =
            Alert(name = name, price = price)

        fun restore(id: AlertId, name: String, price: BigDecimal): Alert =
            Alert(id = id, name = name, price = price)
    }
}
```

## 규칙 2: 불변 식별자는 `val`, 변경되는 상태는 `var` + `private set` (in-place 갱신)

- 절대 안 바뀌는 필드(식별자·생성 시각 등)는 `val`.
- 도메인 로직으로 **변경되는 필드**는 constructor에 `val`/`var` 없이 일반 파라미터로 받고, 클래스 body에 **`var ... private set`** 으로 선언한다.
- 상태 변경 메서드는 **새 인스턴스를 반환하지 않고 자기 필드를 직접 갱신**한다. 반환 타입은 보통 `Unit`.
- 상태 변경 메서드가 아예 없는 모델(생성 후 불변)은 모든 필드를 `val`로 둔다.

```kotlin
// 상태 전이가 있는 애그리거트 — 변경 필드만 var + private set, in-place 갱신
class PairingInvite private constructor(
    val id: InviteId,
    val code: InviteCode,
    val inviterId: MemberId,
    status: InviteStatus,          // val/var 없이 일반 파라미터로 받음
    val createdAt: Instant,
    acceptedBy: MemberId?,
    acceptedAt: Instant?,
) {
    var status: InviteStatus = status
        private set                // 외부 직접 변경 금지
    var acceptedBy: MemberId? = acceptedBy
        private set
    var acceptedAt: Instant? = acceptedAt
        private set

    fun accept(joinerId: MemberId, now: Instant) {   // 새 객체 반환 X — 자기 상태를 직접 바꾼다
        require(usable(now)) { "사용할 수 없는 초대입니다" }
        status = InviteStatus.ACCEPTED
        acceptedBy = joinerId
        acceptedAt = now
    }

    companion object { /* issue(), restore() */ }
}
```

```kotlin
// 상태 전이가 없는 모델 — 전부 val (조회 메서드만)
class Couple private constructor(
    val id: CoupleId,
    val memberAId: MemberId,
    val memberBId: MemberId,
    val pairedAt: Instant,
)
```

// WRONG - constructor에 var로 직접 선언 (외부에서 마음대로 변경 가능)
```kotlin
class PairingInvite private constructor(
    var status: InviteStatus,  // 금지! private set 없이 외부 변경 가능
)
```

> **호출부**: 새 인스턴스를 반환하지 않으므로 `obj.accept(...); outPort.save(obj)` 형태로 쓴다(`save(obj.accept(...))` 아님). 저장은 merge가 통째로 덮는다(`jpa-entity` 스킬).
> **테스트**: 변경 메서드를 호출하는 케이스마다 새 인스턴스를 만든다 — 공유 fixture를 여러 케이스가 변형하면 실행 순서에 의존하게 된다.

## 규칙 3: 비즈니스 로직은 도메인 모델 안에

도메인 관련 검증, 계산, 상태 전이 로직은 도메인 모델 내부에 위치한다.

```kotlin
class Alert private constructor(...) {
    init {
        require(price > BigDecimal.ZERO) { "가격은 0보다 커야 합니다" }
    }
    companion object {
        fun create(name: String, price: BigDecimal): Alert =
            Alert(name = name, price = price)  // init 블록에서 검증
    }
}
```

## 규칙 4: Spring/JPA 어노테이션 금지

도메인 모델은 순수 Kotlin 클래스여야 한다. `@Entity`, `@Component` 등 금지.

## 규칙 5: 도메인 검증 실패 — 코드 예외 vs `require`

도메인 검증이 실패할 때 두 가지로 나눠 던진다.

| 상황 | 방식 | 결과 |
|------|------|------|
| 사용자에게 의미 있는 에러로 보여줘야 함 (예: 탈퇴 회원이 변경 시도) | 코드 예외 — `BaseRuntimeException` + 도메인 `ResponseCode` | 전역 핸들러가 그 코드/상태로 응답 |
| 정상 흐름에선 도달 불가능한 불변식 가드 | `require(...)` (= `IllegalArgumentException`) | 전역 핸들러에서 일반 오류(500급→4xx)로 떨어짐 |

- **코드 예외**: `MemberNotActiveException`처럼 `BaseRuntimeException`을 상속하고 도메인 `ResponseCode`를 들고 던진다. `core`는 Spring 비의존이라 `ResponseCode` enum을 도메인 패키지에 둬도 도메인 순수성이 깨지지 않는다.
- **`require`(백스톱)**: 전용 핸들러가 없어 일반 오류로 떨어진다(친절한 코드/메시지 안 나감). 그래서 **사용자가 도달할 수 있는 조건은 앞단(UseCase)에서 코드 예외로 선검증**하고, 도메인 `require`는 "여기까지 오면 버그/레이스"인 최후 방어선으로만 남긴다. 같은 검증이 UseCase·도메인 양쪽에 있는 건 중복이 아니라 역할 분담 — 앞단은 UX, 도메인은 불변식 보호.

```kotlin
// 코드 예외 — 도메인이 직접 깔끔한 코드를 던짐
fun changeEmail(newEmail: Email) {
    if (status != MemberStatus.ACTIVE) throw MemberNotActiveException(id, status)  // 403
    if (email == newEmail) return
    email = newEmail
}

// require 백스톱 — 앞단(UseCase)에서 InviteNotUsableException(409) 등으로 선검증되므로 보통 도달 X
fun accept(joinerId: MemberId, now: Instant) {
    require(usable(now)) { "사용할 수 없는 초대입니다" }       // 도달 = 버그/레이스
    require(joinerId != inviterId) { "자신의 초대 코드로는 연결할 수 없습니다" }
    ...
}
```

> 앞단(UseCase)에서의 선검증 책임은 `usecase` 스킬 참조.

---

## Value Object 규칙

### 도메인 개념은 Value Object로 래핑

원시 타입(Long, String) 대신 도메인 의미를 가진 VO를 사용한다.

```kotlin
// domain/alert/vo/AlertId.kt
@JvmInline
value class AlertId(val value: Long)

// domain/alert/vo/StockCode.kt
@JvmInline
value class StockCode(val value: String) {
    init {
        require(value.matches(Regex("^[0-9]{6}$"))) {
            "종목코드는 6자리 숫자여야 합니다"
        }
    }
}
```

### VO 규칙 요약

- **위치**: `domain/{도메인}/vo/` 패키지
- **단일 값 VO**: 반드시 `@JvmInline value class` 사용 (data class 금지)
- **검증**: VO 생성 시점에 `init` 블록으로 유효성 보장

## 체크리스트
- [ ] private constructor를 사용하는가?
- [ ] create()와 restore() factory method가 있는가?
- [ ] 생성 factory 이름이 도메인 언어를 따르는가? (발급물=issue, 가입=register, 관계=link, 그 외 엔티티=create / 래퍼=of / VO=generate)
- [ ] 변경되는 필드만 `var` + `private set`이고, 나머지는 `val`인가?
- [ ] 상태 변경 메서드가 새 객체 대신 자기 상태를 갱신하는가? (변경 메서드가 없으면 전부 `val`)
- [ ] 사용자 도달 가능 검증은 코드 예외(또는 앞단 선검증), `require`는 불변식 백스톱으로만 쓰는가?
- [ ] 비즈니스 검증 로직이 도메인 모델 안에 있는가?
- [ ] Spring/JPA 어노테이션이 없는가?
- [ ] ID 타입이 @JvmInline value class VO로 래핑되어 있는가?
- [ ] VO가 domain/{도메인}/vo/ 패키지에 있는가?
