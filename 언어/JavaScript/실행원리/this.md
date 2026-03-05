# this

## 정의
- this는 함수가 어떻게 호출되었는지에 따라 결정되는 값이다.

## 1. 기본 바인딩
```javascript
function show() {
  console.log(this);
}

show();
```
- 브라우저 환경 -> window
- strict mode -> undefined

## 2. 암시적 바인딩
```javascript
const user = {
  name: "jun",
  say() {
    console.log(this.name);
  }
};

user.say();
```
- `.`앞의 객체가 this가 된다.
- 여기서는 user

## 3. 명시적 바인딩
```javascript
function greet() {
  console.log(this.name);
}

const person = { name: "kim" };

greet.call(person);
```
- 명시적 바인딩은 개발자가 직접 this를 지정하는 것이다.
- `call` ,`apply`, `bind`로 설정할 수 있다
### call
- `call`은 **this를 지정하고 함수를 즉시 실행**한다.
### apply
- `apply`는 **call과 동일하지만 인자를 배열로 전달**한다.
### bind
- `bind`는 **this가 바인딩된 새로운 함수를 반환한다.**

## 4. new 바인딩
```javascript
function User(name) {
  this.name = name;
}

const user1 = new User("jun");

console.log(user1.name);
```
- this는 새로 생성된 객체를 가리킨다.

## 5. 화살표 함수
```js
const obj = {
  name: "kim",
  say: () => {
    console.log(this.name);
  }
};

obj.say();

// 결과 undefined
```
- 화살표 함수는 자체적인 this를 가지지 않는다.
- 대신 외부 스코프의 this를 그대로 사용한다.
