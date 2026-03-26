# Collection 프레임워크

## 정의

- 여러 개의 데이터를 효율적으로 저장하고 관리하기 위한 클래스와 인터페이스 집합

## 핵심

| 구분    | 특징           | 대표 구현                 |
| ----- | ------------ | --------------------- |
| List  | 순서 있음, 중복 허용 | ArrayList, LinkedList |
| Set   | 순서 없음, 중복 불가 | HashSet               |
| Map   | key-value 구조 | HashMap               |
| Stack | LIFO (후입선출)  | Stack, Deque          |
| Queue | FIFO (선입선출)  | Queue, Deque          |
## 인터페이스

| 인터페이스 | 설명            |
| ----- | ------------- |
| List  | 순서 있는 데이터     |
| Set   | 중복 없는 데이터     |
| Map   | key-value 데이터 |
## 예제

```java
List<String> list = new ArrayList<>();
Set<String> set = new HashSet<>();
Map<String, Integer> map = new HashMap<>();
```

### 주요 메서드

- add() : 데이터 추가
- remove() : 데이터 삭제
- get() : 데이터 조회 (List)
- put() : 데이터 추가 (Map)

---
예외
map은 collection이 아님