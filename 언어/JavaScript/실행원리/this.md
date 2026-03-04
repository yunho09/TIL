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
