---
title: TypeScript 핵심 개념
description: 
author: laze
date: 2025-06-22 00:00:04 +0900
categories: [Dev, TypeScript]
tags: [TypeScript]
---
### 등장 배경

1. JS → 코드가 길 수록 버그가 많이 발생
2. JS는 동적 타입 언어 ↔ Java, C 등은 정적 타입 언어
3. 컴파일 시점에서 타입 체크

### 개발환경 설정

1. Node.js설치
2. npm 으로 typescript 설치
  1. npm install -g typescript
3. TypeScript 파일 생성 .ts
  1. tsc app.ts → 컴파일
4. 자동컴파일 설정
  1. tsc -w 파일경로 (w = watch)
5. 에러처리
  1. about_Execution_Policies
    1. 관리자 권한 → Treminal → Get-ExecutionPolicy
    2. Set-ExecutionPolicy RemoteSigned
  2. File ‘app.ts’ not found
    1. 컴파일시 경로 설정 제대로
  3. Duplicate Function Implementation
    1. tsc —init
    2. tsconfig.json 파일 생성 → 다양한 설정값 존재 → 문제 해결
  4. cannot find module
    1. node app.js → cannot find module 발생
    2. node .\경로￦app.js 제대로 설정

### Type 종류

1. 기본 타입

| Type | JavaScript | TypeScript |
| --- | --- | --- |
| 수 | Number | number |
| 불리언 | Boolean | boolean |
| 문자열 | String | string |
| 객체 | Object | object |
1. typescript 타입
  1. boolean
  2. number
  3. string
  4. object
  5. array
  6. tuple
  7. enum
  8. any // 모든 타입 허용, 정해지지 않은 변수 지정 가능
  9. void // 값 없음
  10. null
  11. undefined
  12. unkown
  13. never // 도달 불가능한 코드

### 변수 타입 선언

:type (type annotation)

```tsx
let name: string;
let age: number;
```

### 타입 추론

JS와 호환성을 위해 타입 표기를 생략할 수 있으며, 이때 컴파일러가 타입을 추론함 (우측 값에 따라)

### any

모든 타입 허용

타입체크를 위해서 any는 지양해야함

### never

절대 반환을 하지 않는 함수에 사용

도달되지 않는 코드를 나타내며, 실행이 종료되지 않고 무한으로 반복되는 함수나 오류를 발생시키기위해 존재하는 함수 등에 쓰임

```tsx
const neverTest = () => {
	while(true){
		console.log("the function is running");
	}
}
```

neverTest 함수의 타입은 never

```tsx
function sayName(value: string): string {
	if(typeoff value === "string) {
		return value;
	} else {
		return value;
	}
}
```

else 절의 value 를 보면 value: never 로 나옴

### 유니온 타입

```tsx
let a: string| number;
```

두 개 이상의 타입을 가지는 변수

### 커스텀 타입

```tsx
type Centimeter = number;
type Kilogram = number;

type Student = {
	name?: string; // ? -> 선택사항
	height: Centimeter;
	weight: Kilogram;
}
	
```

Property 누락

Property ‘누락된프로퍼티’ is missing in type ‘{ height: number, weight: number; }’ but required in type Student.

### 배열(Array)

```tsx
let list: number[] = [1,2,3,4,5];
```

number[] : 배열 타입

number: 요소 타입

[1,2,3,4,5]: 배열의 요소

```tsx
let member: string[] = ["홍길동","유관순"]
member.push("아무개");
```

요소 타입이 정해져있지 않은 경우 → 유니온타입

```tsx
let member: any[] = ["홍길동", 10, true, null];
let member: (string|number|boolean|null)[] = ["홍길동", 10, true, null];
```

### 제네릭 배열

```tsx
let str: Array<string> = ["홍길동"];
```

Array<Type> 형태로 선언

```tsx
let str: Array<string | number> = ["홍길동", 10]; //유니온 타입
let str2: typeof str = ["홍길동", 10]; // type queries
```

type queries → typeof 연산자 사용, 참조할 변수의 타입을 얻어와 타입 지정

```tsx
let arr: Array<() => string> = [() => "홍길동", () => "유관순"];
console.log(arr[0]()); // result: 홍길동
```

배열 요소를 익명 함수로도 받을 수 있음 () ⇒ string

### 튜플(Tuple)

```tsx
let member: [string, number] = ["홍길동", 10];
console.log(member);
```

배열은 요소의 개수에 제한이 없고, 특정 타입으로 배열 요소의 타입을 강제할 수 있다.

튜플은 n개의 요소에 대응하는 타입이다.

let member: [튜플 타입] = [배열]; 구조로 이루어짐

```tsx
let member: [string, number];  // 'member'는 문자열과 숫자를 요소로 가진 튜플입니다.

member = ['James', 10];  // 이는 허용됩니다.
member = [10, 'James'];  // 이는 허용되지 않습니다. 타입 순서가 맞지 않습니다.
member = ['James', 10, 20];  // 이는 허용되지 않습니다. 튜플의 길이가 맞지 않습니다.
```

### 객체 타입 (Object Types)

TypeScript는 객체를 선언할 때 어떤 타입인지 명확하게 정의해야함

```tsx
const student1: object = { };
const student2 = {}; // any
```

object 는 any보다 좁은 범위의 타입

object는 값 자체가 JavaScript의 객체라고 말해주는 정도 선

타입을 별도로 지정해주지 않으면 any 타입으로 지정해준다.

타입추론

```tsx
const student = {
	name: '홍길동', // string 추론
	grade: 3, // number 추론
}
```

옵션속성

```tsx
const cat: { type: string, age?: number } ={
	type: 'Persian'
};
cat.age = 2;
```

선택적 속성을 사용한다면 객체 생성시 지정하지않고 나중에 값을 추가할 수 있으며,

필드명 뒤에 ? 를 붙임

인덱스 시그니처

```tsx
const student: { [index: string]: number } = {};
student.id = 222123;
student.id = '홍길동'; //error
```

빈 객체를 만들 때 필드의 자료형을 지정하지 않은 채 인덱스를 사용하면 됨

student라는 객체에 id(string)로 지정하고 해당 값은 222123(number)로 추가할 수 있다.

**여기서 `student`는 객체를 선언하고 있습니다. 이 객체의 타입은 `{ [index: string]: number }`으로, 이는 "문자열 키와 숫자 값"을 가진 객체를 나타냅니다. 이는 TypeScript의 인덱스 시그니처(Index Signature)라는 문법을 사용한 것입니다.**

**인덱스 시그니처는 객체의 인덱스를 나타내는 부분에 타입을 설정하는 문법입니다. `{ [index: string]: number }`는 "이 객체는 문자열 키를 가지며, 그에 대응하는 값은 숫자이다"라는 의미입니다.**

**따라서 `student.id = 222123;`는 `student` 객체에 `id`라는 키로 `222123`이라는 숫자 값을 할당하는 것입니다. 이는 인덱스 시그니처 `{ [index: string]: number }`에 부합하므로 문제 없습니다.**

**그러나 `student.id = '홍길동';`은 `id` 키에 문자열 값을 할당하려고 하므로, 인덱스 시그니처 `{ [index: string]: number }`에 부합하지 않습니다. 따라서 TypeScript 컴파일러는 이를 오류로 간주하고, 이 코드는 컴파일되지 않습니다.**

**즉, `{ [index: string]: number }` 타입의 객체는 어떤 문자열 키를 사용하여 값을 할당하더라도, 그 값은 반드시 숫자여야 합니다.**

```tsx
let obj1: { [index: string]: number } = {};
let obj2: { [key: string]: number } = {};
let obj3: { [x: string]: number } = {};
```

### 열거형(Enums)

비슷한 종류의 아이템들을 함께 묶어서 표현할 수 있는 수단

```tsx
enum Class {
	Rock,
	Scissors,
	Paper
}
```

첫글자 대문자

키의 첫 문자도 대문자

숫자 열거형 (Numeric enums)

```tsx
enum Class{
	Rock = 0, //0
	Scissors = 100+1, //101
	Paper // 102
```

값을 작성하지 않으면 0부터 1씩 증가

값을 쓰다가 없으면 이전값 + 1

문자열 열거형 (String Enum)

```tsx
enum Game{
	Rock = 'ROCK',
	Scissors = 'SCISSORS',
	Paper = 'PAPER'
}
```

숫자 열거형처럼 값이 증가하지는 않음

이종 열거형

```tsx
enum Game{
	Rock = 'ROCK',
	Scissors = 2,
	Paper
}
```

const enums

```tsx
const enum Game{
	Rock,
	Scissors,
	Papaer
}
```

값으로는 문자열 리터럴로만 지정할 수 있고, 숫자형 열거형은 안전성을 해치는 문제 발생할 수 있음

→ 값이나 키로 열거형에 접근할 때 존재하지 않는 키에도 접근할 수 있기 때문

→ 이를 해결하기위해 const enum 을 사용하는 것을 추천

### Type Aliases

타입에 대해 별칭 제공, 재사용 가능

정의한 타입을 참고할 수 있게 이름을 짓는 것이지 새로운 타입을 생성하는 것이 아님

```tsx
type 별칭 = 타입;

const food: string = "banana";
const food: string = 1; // Error

//문자열 타입
type Food = string;
const myFood: Food = "banana";
const myFood2: Food = 1; // Error

// 문자열 리터럴
type Name = "Laze"
const userName: Name = "laze" // Error

// 숫자 타입
type MyNumber = number;
const myNumber: MyNumber = 1;

// 유니온 타입
type MyText = string | number;

// 문자열 유니온
type YourText = "hello" | "hi";

// 숫자 리터럴 유니온
type MyNumber = 1 | 2 | 3;

// 객체 리터럴 유니온
type MyObject = {first:1} | {second: 2};

// 함수 유니온
type MyFunction = ( () => string) | ( () => void);

// 인터페이스 유니온
interface A {a: number;}
interface B {b: number;}
type MyInterface = A|B;

// 튜플 타입
type MyTuple = [string, number];

// 제네릭 타입
type MyGenerics<t> = {
	name: T;
}

```

```tsx
// 객체타입
type User = {
	name: string,
	age: number,
}

function getUser(user: User){}
let newUser: User = {
	name: "철수",
	age: 20,
}

getUser(newUser);

//함수 타입
type AddNumber = (x: number, y:number) => number; // return해주는 값이 없을땐 void
let addNumber: AddNumber = (x, y) => {
	return x+y;
}
```

### Interface

새로운 타입을 정의하는 방법

객체, 함수, 함수의 파라미터, 클래스 등에 사용할 수 있으며

상호간에 정의한 약속 혹은 규칙 의미

interface 인터페이스이름 { 속성: 타입; }

```tsx
// 객체의 스펙(속성과 속성의 타입)
interface User {
	name: string;
	age: number;
}

let user1: User = {
	name: "철수",
	age: 20,
}

let user2 = {} as User;
user2.name = "영희";
user2.age = 21;

// 함수의 스펙(파라미터, 반환타입 등)
interface AddNumber {
	(x: number, y: number): number; // return 해주는 값이 없을 땐 void
}

let addNumber = (x, y) => {
	return x+y;
}
addNumber(10,10);

// 함수의 파라미터
interface User {
	name: string;
	age: number;
}

function getAge(obj: User) {
	console.log(obj.age);
}

let person = {
	name: "철수",
	age: 20,
	gender: "male"
};

getAge(person);

```

인터페이스를 인자로 사용할 때, 정의한 프로퍼티와 타입의 조건을 만족한다면

인자로 받는 객체의 프로퍼티 개수나 순서는 같지 않아도 된다.

옵션 속성

인터페이스를 정의할 때 옵션 속성을 사용하면 그 속성을 사용하지 않아도 된다.

> 속성?: 타입;
>

```tsx
interface User {
	name: string;
	age: number;
	gender?: "male" | "female";
}

function getAge(obj: User){
	console.log(obj.name);
	console.log(obj.age);
}

let person = {
	name: "철수",
	age: 20,
};

getAge(person);
```

readonly 속성: 타입;

객체를 생성할 때 값을 할당 한 후 변경할 수 없는 속성 의미

```tsx
interface User {
	readonly name: string;
}

let person: User {
	name: "laze"
};

person.name = "철수"; // Error
```

ReadonlyArray<T> 타입을 사용하여 읽기 전용 배열을 생성할 수 있다.

```tsx
let Users: ReadonlyArray<string> = ["철수", "영희"];

Users.splice(0,1); // Error
Users.push("현지"); // Error
Users[0] = "현지" // Error
```

클래스 타입

implements 로 미리 정의된 인터페이스를 채택하여 클래스를 정의할 수 있다.

인터페이스로 정의한 메서드, 프로퍼티는 해당 클래스에 필수적으로 들어가야한다.

```tsx
interface User {
	name: string;
	age: number;
	getAge(age: number): void;
}

class newUser implements User {
	name: string = "철수";
	age: number = 20;
	getAge(age: number) {
		console.log(this.age);
	}
	constructor(){}
}

let user1 = new newUser();
user1;
```

인터페이스와 타입 별칭 차이

→ 타입의 확장 가능, 불가능 여부

```tsx
interface User {
	name: string;
}

interface User{
	age: number;
} // 같은 이름으로 정의하면 자동으로 확장된다.
```

```tsx
type User = {
	name: string;
}

type User = {
	age: number;
} // Error
```

인터페이스 확장

```tsx
interface User {
	name: string;
}

interface Gender {
	gender: string;
}

interface UserInfo extends User,Gender{
	age: number;
}
```

인터페이스는 타입도 상속받을 수 있는데

리터럴 타입은 불가능하다

```tsx
interface UserInfo2 extends string {} // Error
```

유니온 연산자를 사용한 타입 또한 extends나 implements 할 수 없다.

타입의 확장은 extends가 아닌 & 로 확장가능하다.

```tsx
type User = { // interface also possible
	name: string;
}

type UserInfo = User & {
	age: number;
};

```

가능하면 interface를 사용하고

유니온 , 튜플 등을 사용해야하는 상황에서는 타입 별칭을 사용하도록 권장

(공식문서)

### 유니온 타입

```tsx
let text: string | number = 22;
text = '22';
```

유니온 타입의 인터페이스를 연결시 모든 타입의 공통 속성만 접근 가능

```tsx
interface User {
	name: string;
	age: number;
}

interface Info {
	name: string;
	description: string;
}

function combine(obj: User | Info){
	obj.name; // 정상동작
	obj.age; // Type Error
	obj.description; // Type Error
}
```

유니온타입 가드

유니온 타입 내에 무엇이 있는지 분석할 수 없기때문에

런타임 타입 검사를 추가하여 어느쪽에 해당하는지 판정해줘야함

```tsx
function combine(input1: number | string, input2: number | string){
	let result;
	if(typeof input === 'number' && typeof input2 === 'number'){
		result = input1 + input2;
	}else {
		result = input1.toString() + input2.toString();
	}
	return result;
}
console.log('hello','world');
console.log(10,10);
```

클래스 객체 식별

```tsx
interface Cat {
	meow(): string
}

interface Dog {
	bowwow(): string
}

function checkType(pet: Cat | Dog) {
	if("meow" in pet){
		(pet as Cat).meow()
	}
	else{
		(pet as Dog).bowwow()
	}
};
```

if 문을 통해 pet 타입을 체크 했지만 내부에서 as 문을 통해 한번 더 어떤 객체인지 알려줘야 함

null 체크

문자열의 값이 존재하지 않는 경우

string | null 이라고 유니온타입 지정하면 값이 null이어서 string으로 사용할 수 없거나

null이 아닌 경우 예외처리 할 때 사용

```tsx
function nullCheck(val: string | null): number {
	if(val != null){
		return val;
	} else {
		return 0;
	}
}
```

### Type Casting

Java, C++ 과 다르게 특정 타입을 단언해야하는 상황에서만 사용하고

특별한 검사나 , 데이터 재구성을 하지 않음

### Type Assertion

시스템이 추론한 타입의 내용을 변경하는 것

<> 표기법, as 연산자

```tsx
var val:any;
var foo = <string>val

var val2:any;
var foo2 = val as number
```

문자열로 선언된 변수를 as number[] 로 타입을 무시하고 숫자형을 넣으면

런타임 에러 발생

### Function

JavaScript → 매개변수 적고 인자 넘기지 않으면 undefined

TypeScript → 매개변수 적고 인자 넘기지 않으면 Error

매개변수 선언하면 필수적으로 인자를 입력해야한다.

```tsx
function sum(number1: number, number2: number): number{
	return number1 + number2;
}

console.log(sum(10,20));
```

매개변수 뒤에 타입 지정 ( 매개변수 : 타입 ) :타입 ← 반환형의 타입, 없는경우 void 기재

선택적 매개변수 ?

Optional한 값 → 인자를 넘기지 않아도 에러발생 x

단 선택적 매개변수는 필수 매개변수 뒤에만 올 수 있음

매개변수 초기화

매개변수를 선언할 때 값을 미리 초기화 > 따로 타입을 지정해주지 않아도 된다.

```tsx
function sum(a: number, b = 2022): number {
	return a+b;
}
```

REST 문법이 적용된 매개변수 (전개문법, 배열타입)

```tsx
function sum(...numbers: number[]): number{
	return numbers.reduce((result, number) => result + number, 0)
}

console.log(sum(10,20,30));
```

this > this의 타입을 정할 때는 함수의 첫 번째 매개변수 자리에 this를 쓰고 타입을 입력한다.

실제로 인자값을 받는 매개변수는 this:Type 을 제외한 나머지임

```tsx
interface User {
	name: string,
	age: number,
	init(this: User): {};
}

let user1: User = {
	name: 'laze',
	age: 20,
	init: function(this: User) {
		return () => {
			return this.age;
		}
	}
}

let getAge = user1.init();
let age = getAge();
console.log(age);
```

**`this` 키워드는 현재 실행 중인 함수가 속한 객체를 참조합니다. 따라서 `init` 함수에서 `this.age`로 접근하면, `init` 함수를 호출하는 객체의 `age` 속성에 접근하게 됩니다.**

**`user1.init()`를 호출하면, `init` 함수 내부의 `this`는 `user1` 객체를 가리킵니다. 따라서 `this.age`는 `user1.age`를 의미하게 됩니다.**

**여기서 `this: User`는 `this`가 `User` 인터페이스를 따르는 객체를 가리키도록 강제하는 역할을 합니다. 즉, `this`를 통해 접근할 수 있는 속성과 메서드는 `User` 인터페이스에 정의된 것들로 제한됩니다.**

**따라서 `this.age`는 `User` 인터페이스의 `age` 속성이 아니라, `init` 함수를 호출하는 객체(`user1`)의 `age` 속성에 접근하게 됩니다. 이 때 `init` 함수 내부에서 `this`를 통해 접근할 수 있는 속성과 메서드는 `User` 인터페이스에 정의된 것들로 제한되는 것입니다.**

콜백에서 this

```tsx
interface BrowserEL {
	addClickListener(onclick: (this: void, e:Event) => void): void;
}

class Handler {
	info: string;
	onClick(this: Handler, e: Event) {
	//BrowseEL 의 this 타입은 void -> Handler 로 타입선언시 에러
		this.info = e.message;
	}
}
let handler = new Handler();
browserEl.addClickListener(handler.onClick); // error
```

```tsx
class Handler {
	info: string;
	onClick(this: void, e: Event) {
	// this Type = void -> this.변수명 사용불가
		console.log('clicked');
	}
}
let handler = new Handler();
browserEl.addClickListener(handler.onClick);
```

궁금해서 추가로 찾아본 내용

- this: void

  this: void로 선언한다는 건 함수가 this를 사용하지 않겠다고 명시적으로 선언한 것

  함수가 특정 객체에 바인딩 되지 않고 독립적으로 실행되어야할때, this context에 의존하지 않아야 할 때 사용함


함수 오버로드

매개변수의 개수는 동일하지만 타입이 다른경우 any

```tsx
function sum(a: string, b: string): string;
function sum(a: string, b: number): number;

function sum(a: any, b:any): any{
	return a+b);
}
```

매개변수의 개수는 다르지만, 타입은 같은 경우

```tsx
function sum(a: number): number;
function sum(a: number, b: number): number;
function sum(a: number, b: number, c: number);

function sum(a: number, b?: number, c?: number): number{
	return a + (b||0) + (c||0);
}
```

**`||` 연산자는 첫 번째 truthy 값을 반환하고, `&&` 연산자는 마지막 값을 반환합니다.**

매개변수의 개수와 타입이 다른경우

```tsx
interface NicknameMaker{
	name: string,
	num: number,
		init(this: NicknameMaker): () => {};
}

function makeNickname(name: string, num: number | string): NicknameMaker | string {
	if(typeof num === "number"){
		return{
			name,
			num,
				init: function(this: NicknameMaker){
					return () => {
						return this.name + this.num;
					}
				}
		};
	}else{
		return "no string just number";
	}
}
const getNickname = makeNickname("laze",24).init();
```

이경우 makeNickName 의 return type > string인 경우 getNickname = undefined

### 클래스

프로퍼티 & 메서드 선언

생성자 함수를 통해 초기화가 되는 프로퍼티는 초기화 전 프로퍼티 정의가 꼭 필요함

타입 미표기시에도 타입 추론이 있어 에러발생은 하지 않음

JavaScript에서는 프로퍼티를 미리 선언해도 되고 안 해도 문제되지 않는다.

일반적인 프로퍼티 선언

```tsx
class Test {
	month: string,
	day: number,

	constructor(month, day){
		this.month = month;
		this.day = day;
	}
}
```

생성자 매개변수를 이용한 프로퍼티 선언

```tsx
class Test {
	constructor(private readonly month: string) {}
	
	getMonth(): string {
		return this.month;
	}
}
```

**`month`라는 이름의 private 멤버 변수를 선언하고, 생성자의 매개변수로 받은 값을 이 변수에 할당합니다. `readonly` 키워드는 해당 변수가 오직 생성자 내에서만 값을 할당받을 수 있으며, 그 이후에는 값을 변경할 수 없다는 것을 의미**

접근제한자

- public 클래스 내부/외부, default값
- private 해당 클래스 내부
- protected 해당 클래스 내부 + 상속받은 클래스 내부

readonly

→ interface와 type 정의 등에서 사용할 수 있음 의도치 않게 값이 변경됐을 때 발생할 수 있는 오류 방지

extends

→ 상속받고자하는 부모클래스 명시

interface

- 새로운 타입을 정의하는 방식중 하나 ( interface <> type )
- 구현 없이 이름 + 타입만 정의한 추상화된 프로퍼티, 메서드만 가짐
- 다중 상속 가능
- 인터페이스로 정의한 프로퍼티, 메서드는 해당 클래스에 필수로 들어가야하며
  오버라이딩을 통해 구현

abstract

- 상속용으로만 가능, 객체 인스턴스 생성 불가
- 상속받은 클래스는 추상메서드를 반드시 구현(오버라이딩)

interface ↔ abstract

추상메서드를 갖는 점은 공통점이며,

차이는 일반메서드를 포함할 수 있는지이다. abstract에서만 가능함

interface ↔ type

**Step 1: 기본적인 차이점 이해하기**

**TypeScript에서 `interface`와 `type`은 모두 사용자 정의 타입을 생성하는 데 사용됩니다.**

**Interface:**

**인터페이스는 주로 객체의 형태(shape)를 정의하는 데 사용됩니다.인터페이스는 클래스에 특정 속성이나 메서드가 있음을 보장하는 데 사용될 수 있습니다.인터페이스는 확장(extends)하거나 구현(implements)할 수 있습니다.**

```tsx
typescript

interface User {
  name: string;
  age: number;
}

```

**Type:**

**`type`은 인터페이스뿐만 아니라 프리미티브 타입, 유니온, 인터섹션 등 보다 광범위한 타입을 표현할 수 있습니다.`type`은 다른 타입을 참조하여 새로운 타입을 생성할 수 있습니다.**

```tsx
typescript

type User = {
  name: string;
  age: number;
}

```

**Step 2: 인터페이스 확장 및 구현**

**인터페이스는 다른 인터페이스를 확장(extends)하거나 클래스에서 구현(implements)할 수 있습니다.**

```tsx
typescript

interface User {
  name: string;
}

interface Admin extends User {
  permissions: string[];
}

class AdminUser implements Admin {
  name = 'admin';
  permissions = ['read', 'write'];
}

```

**Step 3: 타입 별칭과 유니온, 인터섹션**

**`type`은 인터페이스뿐만 아니라 프리미티브 타입, 유니온, 인터섹션 등 보다 광범위한 타입을 표현할 수 있습니다.**

```tsx
typescript

type StringOrNumber = string | number;// 유니온 타입

type UserAndAdmin = User & Admin;// 인터섹션 타입
```

**Step 4: 타입 별칭의 복제**

**`type`은 다른 타입으로부터 새로운 타입을 생성(복제)할 수 있습니다.**

```tsx
typescript

type User = {
  name: string;
  age: number;
}

type DuplicateUser = User;// User 타입을 복제
```

**Step 5: 인터페이스와 타입의 재정의**

**인터페이스는 동일한 이름으로 다시 선언하면 원래의 인터페이스를 확장한 것으로 간주되지만, `type`은 동일한 이름으로 다시 선언할 수 없습니다.**

```tsx
typescript

interface User {
  name: string;
}

interface User {
  age: number;// User 인터페이스를 확장
}

type UserType = {
  name: string;
}

type UserType = {
  age: number;// 오류: Duplicate identifier 'UserType'.
}

```

**이처럼, `interface`와 `type`은 비슷한 용도로 사용되지만, 사용할 수 있는 기능과 사용법에는 차이가 있습니다. 어떤 것을 사용할지는 주로 개발자의 선호나 특정 상황에 따라 결정됩니다.**

### 제네릭

```tsx
function testGeneric<T>(message: T): T{
	return message;
}

console.log(testGeneric("test"));
```

message라고 받은 인자를 T라고 지정

함수내에서 T라고 지정됨 T에 string이 들어간다면 함수 내 T는 string인 것

제네릭 장점은 T를 변수처럼 사용해서 Type을 지정해 줄 수 있음

any는 모든 것, 제네릭은 지정해서 사용하는 차이점이 있음

### 유틸리티 타입

정의해놓은 타입을 변환할 때 사용

Partial<Type>

모든 프로퍼티를 선택적으로 타입을 변환하는 유틸리티

```tsx
interface Book {
	title?: string;
	description?: string;
}

let typescriptBook: Partial<Book> = {
	title: "TypeScript",
};
```

Required<Type>

모든 프로퍼티를 필수로 설정한 타입을 생성하는 유틸리티 타입

```tsx
interface Book {
	title: string;
	description: string;
}
let typescriptBook: Partial<Book> = {
	title: "TypeScript",
	description: "설명",
};
```

Readonly<Type>

Type의 모든 프로퍼티를 재할당이 불가능한 읽기 전용으로 설정한 타입생성

```tsx
interface Book {
	readonly title: string;
	readonly description: string;
}

let typescriptBook: Readonly<Book> = {
	title: "TypeScript",
	description: "설명",
};

typescriptBook.title = "tsBook" // 재할당 불가능
```

Record<Keys, Type>

프로퍼티 타입을 Key, value타입을 Type으로 지정해 생성

프로퍼티를 다른 타입으로 매핑하고싶을 때 사용

```tsx
interface Food {
	"1": "pizza" | "chicken" | "hamburger";
	"2": "pizza" | "chicken" | "hamburger";
	"3": "pizza" | "chicken" | "hamburger";
}

const food: Food = {
	"1": "pizza",
	"2": "chicken",
	"3": "hamburger",
};

// Record<Keys, Type>
const food: Record<
	"1"|"2"|"3",
	"pizza"|"chicken"|"hamburger"
> = {
	"1": "pizza",
	"2": "chicken",
	"3": "hamburger",
};

// type > 정리
type Id = "1"|"2"|"3";
type Food = "pizza"|"chicken"|"hamburger";

const food: Record<Team, Food> ={ 
	"1": "pizza",
	"2": "chicken",
	"3": "hamburger",
}
```

장점 : 가독성, 유지보수

Pick<Type, Keys>

```tsx
interface Person{
	name: string;
	age: number;
	location: string;
	gender: "M" | "W";
}

const kim: Pick<Person, "name" | "age"> = {
	name: "kim",
	age: 27,
};
	
```

name, age가 아닌 다른 key선택시 에러 발생

Omit<Type, Keys>

pick 과 반대로 생략할때 사용

```tsx
interface Person{
	name: string;
	age: number;
	location: string;
	gender: "M" | "W";
}

const kim: Omit<Person, "name" | "age"> = {
	location: "kim",
	gender: 27,
};
```

Exclude<Type, ExcludeUnion>

```tsx
type T1 = Exclude<"kim"|"lee"|"park", "park">;
```

type T1 = kim | lee 가 됨 park제외

NonNullable<Type>

```tsx
type T1 = NonNullable<string | number | boolean | null | undefined>;
```

T1 = string, number, boolean

null 과 undefined 제외됨
