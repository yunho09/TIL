# DateTime

## 정의

- 날짜(Date)와 시간(Time)을 생성, 변환, 계산하기 위한 자바 표준 라이브러리

## 날짜 / 시간 객체

|클래스|설명|
|---|---|
|`LocalDate`|날짜만 (연, 월, 일)|
|`LocalTime`|시간만 (시, 분, 초)|
|`LocalDateTime`|날짜 + 시간|

## 변환 (Parsing & Formatting)

- 문자열 → 날짜 → `parse()`
- 날짜 → 문자열 → `format()`

```java
LocalDate.parse("2026-03-26");
date.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
```

## 계산

- 날짜 더하기 / 빼기
- 날짜 차이 계산

```java
date.plusDays(1);
date.minusMonths(1);
```