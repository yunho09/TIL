---
tags:
  - spring
  - di
  - bean
updated: 2026-09-04
---

# 빈과 DI

> 빈(Bean) = 스프링이 만들어서 [[스프링-컨테이너|컨테이너]]에 보관하는 객체.
> DI(의존성 주입) = 필요한 객체를 직접 `new` 하지 않고 생성자로 받는 것.

## 기본 패턴 — 인터페이스 + 구현체 + 생성자 주입

학습용으로 만들었던 예시(발송 수단을 규격만 알고 쓰는 서비스, 이후 CRUD로 넘어가며 삭제함):

```java
public interface MessageSender {
    void send(String to, String message);
}

@Component
public class EmailSender implements MessageSender {
    @Override
    public void send(String to, String message) {
        System.out.println("이메일 → " + to + " : " + message);
    }
}

@Service
public class UserService {
    private final MessageSender sender;   // 규격만 앎, 구현체는 모름

    public UserService(MessageSender sender) {
        this.sender = sender;
    }
}
```

- `@Component`/`@Service` — 컨테이너에 등록해달라는 표시. 없으면 `NoSuchBeanDefinitionException`
- `implements` — 컴파일러가 메서드 구현을 강제 + 이 타입으로도 취급되게 함(빈 검색의 근거)
- 필드가 구현체 타입이 아니라 **인터페이스 타입**인 것이 DI의 핵심. `= new EmailSender()`를 안 쓰는 게 전부

## 같은 타입 구현체가 여럿일 때

`private final MessageSender sender;`처럼 칸이 하나인데 구현체 후보가 둘(`EmailSender`, `KakaoSender`) 이상이면 스프링이 못 고르고 `NoUniqueBeanDefinitionException`을 던진다. 해결 3가지:

| 원하는 것 | 방법 |
| --- | --- |
| 하나만 (기본값 지정) | `@Primary` — 여러 후보 중 기본값 표시 |
| 하나만 (받는 쪽에서 지정) | `@Qualifier("빈이름")` |
| **전부 다** | `List<MessageSender>` |

```java
private final List<MessageSender> senders;

public UserService(List<MessageSender> senders) {
    this.senders = senders;   // 등록된 MessageSender 구현체 전부가 담김
}

public void join(String email) {
    for (MessageSender s : senders) {
        s.send(email, "가입을 환영합니다");   // 이메일도, 카톡도 나감
    }
}
```

`NoUniqueBeanDefinitionException`은 "구현체가 둘이면 안 된다"가 아니라 **"칸은 하나인데 후보가 둘이라 못 고른다"**는 뜻. `List<T>`로 받으면 고를 필요가 없어져 에러가 안 난다. 새 구현체(`SlackSender` 등)를 추가해도 `UserService`는 안 고쳐도 자동으로 목록에 합류한다.

## 실무에서 실제로 쓰는 빈도

| 패턴 | 빈도 | 실무 예 |
| --- | --- | --- |
| 생성자 주입 (`private final X x;`) | 매일 | 스프링 코드의 기본 문형 |
| `List<T>` 전체 주입 | 꽤 자주 | 알림 이메일+푸시+슬랙 동시 발송, 결제수단별 처리기, 검증기 체인 |
| `@Qualifier` | 가끔 | DataSource 두 개, 설정 다른 HTTP 클라이언트 두 개 |
| `@Primary` | 드물게 | 테스트에서 가짜 구현체를 기본값으로 |

> [!important] 인터페이스가 필요 없는 경우가 더 흔하다
> 실무 코드 대부분은 인터페이스 없이 구현체가 하나뿐이다 (`PostRepository postRepository` 처럼). 인터페이스+다중 구현체 구조는 "교체 가능성이 실제로 있을 때"만 만든다 — 구현체가 하나인데 습관적으로 인터페이스부터 만드는 건 과설계로 본다.

## 싱글톤

스프링 빈은 **기본이 싱글톤**이다. `@Service`, `@Component`를 붙이는 순간 컨테이너 안에 객체가 딱 하나만 만들어지고, 요청이 몇 번이든 그 하나를 모두가 공유한다. 별도로 설정할 것이 없다.

지켜야 할 규칙은 문법이 아니라: **빈에 요청별 상태를 두지 마라.** `private final X x;`(의존성)만 있으면 무해하지만, `private int count = 0;` 같은 가변 상태를 두면 모든 요청이 그 값을 공유해 꼬인다.

확인법: 컨트롤러에서 `postService.toString()`을 몇 번 찍어봐도 `PostService@1a2b3c4d`처럼 **같은 객체 식별자**가 나온다.

## 출처
- Claude Code 세션 자동 캡처 (/data/project/spring-practice)
