---
title: Vue 프로그래밍의 기본
description: Vue?
author: laze
date: 2025-03-17 13:00:00 +0900
categories: [Dev]
tags: [Vue]
---
# 3장 Vue 프로그래밍의 기본

Vue 프로젝트에서는 컴포넌트를 사용해 애플리케이션을 제작함

컴포넌트

- 웹 페이지를 구성하는 요소
    - HTML 태그
    - CSS 스타일시트
    - 자바스크립트 코드

SFC(단일파일컴포넌트)

- 위의 세 요소가 하나의 컴포넌트(파일)로 묶여있음

```tsx
App.vue

<script setup lang="ts"> // 스크립트 블록
...
</script>

<template> // 템플릿 블록 HTML 태그 작성
...
</template>

<style> // 스타일 블록 CSS 코드 작성
...
</style>
```

```tsx
<script setup lang="ts">
import {ref} from "vue";
const name = ref("Laze");
</script>

<template>
  <h1>hi {{ name }} !</h1>
</template>
```

- setup - 스크립트 블록 내의 코드를 단순화 할 수 있음 →  how ? 추후 학습
- {{ name }} - mastache 구문 → 스크립트 블록에서 준비한 변수를 표시하기 위한 구문
- ref - 템플릿 블록에서 사용할 변수(템플릿 변수)를 정의하는 방법
const name = ref("Laze"); 에서 ref 함수의 인자로 값을 전달하고 반환값을 name에 담음
- module export & import
    - modern javascript에서는 함수, 클래스, 객체 등을 각각 별도의 파일로 작성하고 이를 불러와서 사용할 수 있는 구조를 모듈이라 함
    - 해당 함수나 클래스를 외부에서 사용할 수 있도록 export function ref(…) {…}
    - import {…} 가져오고자하는 함수나 클래스명을 쉼표로 구분해 나열 export로 내보낸 함수명, 클래스명과 일치해야 함
    - export default function showName() {…} - 모듈 파일에 default는 하나만 정의 가능
        - 이름없음으로 내보내는 것이기 때문에 가져오는 쪽에서 이름을 지정해야 함
        - import sn from “…”;

반응형 시스템

- 변숫값의 변화에 따라 표시 내용이 자동으로 바뀌는 것
- const name = ref(value);
- name.value = new_value;

```tsx
<script setup>
import { ref } from "vue";

const now = new Date();
const nowStr = now.toLocaleTimeString();

let timeStr = nowStr;

const timeStrRef = ref(nowStr);

function changeTime() {
  const newTime = new Date();
  const newTimeStr = newTime.toLocaleTimeString();
  timeStr = newTimeStr;
  timeStrRef.value = newTimeStr;
}

setInterval(changeTime, 1000);

</script>

<template>
  <p>현재 시각 : {{ timeStr }} </p>
  <p>현재 시각(ref) : {{ timeStrRef }}</p>
</template>
```

- {{ timeStr }} 과 {{ timeStrRef }} 가 왜 같이 변하는지?
    
    Q : timeStrRef만 반응형인데 실제로는 둘 다 같이 변하는 현상이 일어나는 이유는?
    
    - **변수 값 변경과 자동 렌더링:** 변수 값이 변한다고 해서 자동으로 렌더링이 되지 않음
    - **`now`, `nowStr`, `timeStr`:** `now`는 `Date` 객체, `nowStr`은 문자열, `timeStr`은 초기 시간을 담는 일반 변수
    - **`timeStrRef`와 반응형:** `timeStrRef`가 `ref` 객체(반응형 객체)이고, `timeStrRef.value`가 변경되면 Vue가 이를 감지하여 렌더링을 트리거
    - **`changeTime` 함수:** 새로운 시간을 가져와서 문자열로 변환하고, `timeStr`과 `timeStrRef.value`를 모두 업데이트
    - **`{{ timeStr }}`의 동작:** `timeStr`이 일반 변수이기 때문에 *자동으로* 렌더링되지 않지만 `changeTime` 함수 내에서 `timeStr = newTimeStr;` 코드를 통해 *명시적으로* `timeStr`의 값을 *변경*하고 있음 
    그리고 템플릿에서 `{{ timeStr }}`를 사용하고 있기 때문에, 
    `timeStr`의 *변경된 값*이 화면에 표시됨
    
    **정리(단계별 설명)**
    
    1. **초기 렌더링:**
        - 컴포넌트가 처음 렌더링될 때, `timeStr`과 `timeStrRef.value`는 모두 `nowStr`의 값 (초기 시간)을 가짐
        - 템플릿은 `timeStr`과 `timeStrRef.value`의 값을 모두 표시하므로 두 부분 모두 초기 시간을 보여줍니다.
    2. **`setInterval`과 `changeTime`:**
        - `setInterval`은 1초마다 `changeTime` 함수를 *호출*
        - `changeTime` 함수는:
            - 새로운 시간을 가져와서 문자열(`newTimeStr`)로 만듦
            - `timeStr = newTimeStr;` <-- `timeStr`의 값을 *명시적으로 변경*
            - `timeStrRef.value = newTimeStr;` <-- `timeStrRef.value`를 변경하고, Vue는 이를 감지하여 렌더링을 트리거
    3. **템플릿 업데이트:**
        - `{{ timeStr }}`:
            - Vue가 자동으로 업데이트하는 것이 아님.
            - `changeTime` 함수에서 `timeStr`의 값을 *명시적으로 변경*했기 때문에, 변경된 값이 표시됨
        - `{{ timeStrRef }}`:
            - Vue의 반응형 시스템이 `timeStrRef.value`의 변경을 감지하고, 이 부분을 *자동으로* 업데이트합니다.
    
    **핵심은 `timeStr`이 *자동으로* 렌더링되는 것이 아니라, `changeTime` 함수에서 *명시적으로 값을 변경*하고 있기 때문에 템플릿에 반영된다는 것**
    
    **만약 `timeStr`이 변하지 않게 하려면?**
    
    `changeTime` 함수에서 `timeStr = newTimeStr;` 이 부분을 제거 
    이렇게 하면 `timeStr`은 초기 값으로 고정되고, `timeStrRef.value`만 변경되어 `{{ timeStrRef }}` 부분만 업데이트 됨
    
    **정리 (최종)**
    
    - `ref`는 Vue의 반응형 시스템을 위한 것
     `ref` 객체의 `.value`를 변경해야 Vue가 렌더링을 트리거
    - 일반 변수(`timeStr`)는 값이 변경되어도 자동으로 렌더링되지 않음
    - 현재 코드에서는 `changeTime` 함수가 `timeStr`의 값을 *명시적으로 변경*하고 있기 때문에, `{{ timeStr }}` 부분도 업데이트되는 것처럼 보이는 것
    - `timeStr`도 바뀌는 이유는 `setInterval`에서 콜백함수를 호출하면서 `timeStr`의 값을 바꾸기 때문입니다. `timeStr = newTimeStr`를 지우면 `timeStr`는 바뀌지 않음
- computed()
    - 계산 결과를 반응형으로 만듦
    
    ```tsx
    <script setup lang="ts">
    import { ref, computed } from "vue"
    
    const radiusInit = Math.round(Math.random() * 10);
    const PI = ref(3.14);
    const radius = ref(radiusInit);
    
    const area = computed(
      (): number => {
        return radius.value * radius.value * PI.value;
      }
    );
    
    setInterval(
      (): void => {
        radius.value = Math.round(Math.random() * 10);
      },
      1000
    )
    </script>
    
    <template>
      <p>반지름이 {{  radius }} 이고 원주율이 {{  PI  }}인 원의 면적은 {{ area }}</p>
    </template>
    ```
    
- 기타 궁금한점
    
    const 는 상수인데 상수는 값 할당하면 변경할 수 없음
    
    근데 computed를 const에?
    
    area는 값이 변한다로 이해할게 아니라 의존하는 값이 변경되면 자동으로 다시 계산된다 정도로 이해하는게 좋음
    
    computed 함수에는 getter와 setter가 있음
    
    getter는 필수 여기서는 (): number ⇒ { return radius~~~ } 여기서 number는 return type
    
    setter를 사용할꺼면
    
    ```tsx
    // getter만 있는 경우 (읽기 전용)
    const area = computed((): number => {
      return radius.value * radius.value * PI.value;
    });
    
    // getter와 setter가 모두 있는 경우
    const fullName = computed({
      get: (): string => {
        return firstName.value + ' ' + lastName.value;
      },
      set: (newValue: string) => {
        [firstName.value, lastName.value] = newValue.split(' ');
      }
    });
    
    // fullName에 값을 할당 (setter 호출)
    fullName.value = 'John Doe'; // 이 코드가 set 함수를 호출합니다.
    
    // firstName과 lastName 확인
    console.log(firstName.value); // 'John'
    console.log(lastName.value);   // 'Doe'
    ```
    

- reactive
    - 객체를 한꺼번에 반응형으로 만듦
    - 여러 개의 데이터를 묶어 하나의 객체로 취급하여 반응형으로 만듦
    
    ```tsx
    <script setup lang="ts">
    import {reactive, computed} from "vue"
    
    const data = reactive({
      PI: 3.14,
      radius: Math.round(Math.random() * 10)
    });
    
    const area = computed(
      (): number => data.radius * data.radius * data.PI
    )
    
    setInterval(
      (): void => {
        data.radius = Math.round(Math.random() * 10);
      },
      1000
    )
    </script>
    
    <template>
      <p> 반지름이 {{ data.radius }} 이고 원주율이 {{ data.PI }}인 원의 면적은 {{ area }}입니다.</p>
    </template>
    ```
    
- 화살표 함수 생략
    
    화살표 함수(arrow function)에서는 단일 표현식을 반환하는 경우 중괄호(`{}`)와 `return` 키워드를 생략할 수 있음
    
    ```tsx
    const area = computed(
      (): number => {
        return data.radius * data.radius * data.PI;
      }
    );
    
    ```
    
    이 코드는 다음과 같이 중괄호와 `return`을 생략하여 더 간결하게 작성할 수 있음
    
    ```tsx
    const area = computed(
      (): number => data.radius * data.radius * data.PI
    );
    
    ```
    
    **화살표 함수에서 중괄호와 `return`을 생략하는 규칙:**
    
    - **단일 표현식(single expression):** 반환 값이 *단일 표현식*인 경우에만 중괄호와 `return`을 생략할 수 있음
    - **여러 문(multiple statements):** 함수 본문에 여러 개의 문(statement)이 있거나, 부수 효과(side effect)를 발생시키는 코드가 있는 경우에는 중괄호를 생략할 수 *없음*
    
    **예시:**
    
    ```jsx
    // 단일 표현식 (중괄호, return 생략 가능)
    const add = (a: number, b: number): number => a + b;
    
    // 여러 문 (중괄호, return 생략 불가)
    const logAndAdd = (a: number, b: number): number => {
      console.log(`Adding ${a} and ${b}`);
      return a + b;
    };
    
    // 부수 효과 (중괄호, return 생략 불가)
    const incrementCounter = (): void => {
      counter++; // counter 변수를 증가시킴 (부수 효과)
    };
    
    ```
    
    ```tsx
    // 정상
    (): number => data.radius * data.radius * data.PI
    
    // 오류 (중괄호 X, return O)
    (): number => return data.radius * data.radius * data.PI
    
    // 오류 (중괄호 X, 세미콜론 O)
    (): number => data.radius * data.radius * data.PI;
    
    ```
    
    - 첫 번째 코드는 중괄호, `return`, 세미콜론을 모두 생략한 올바른 형태
    - 두 번째 코드는 중괄호 없이 `return`을 사용했으므로 문법 오류
    - 세 번째 코드는 중괄호 없이 세미콜론을 사용했으므로 문법 오류
    
    화살표 함수에서 중괄호(`{}`)를 *사용하는 경우*와 *사용하지 않는 경우*는 서로 다른 의미를 가집니다.
    
    1. **중괄호 `{}`를 사용하는 경우:**
        - 함수의 *본문(body)*을 정의하는 것으로 간주
        - 본문에는 여러 개의 문(statement)이 올 수 있음
        - 값을 반환하려면 반드시 `return` 키워드를 명시해야함
        - 각 문(statement)의 끝에는 세미콜론(`;`)이 있어야 함. (일반적인 JavaScript 문법 규칙)
    2. **중괄호 `{}`를 사용하지 않는 경우:**
        - 단일 표현식(expression)*을 함수의 반환 값으로 간주
        - `return` 키워드를 사용하면 문법 오류
        - 표현식 뒤에 세미콜론(`;`)을 붙이면  문법 오류
- ref 와 value / reactive는 왜 안쓰는지?
    
    **핵심 차이점: 반응형으로 만들 대상**
    
    - **`ref`:** *원시 타입(primitive type)* 값 (숫자, 문자열, 불리언 등) 또는 *객체*를 반응형으로 만들 때 사용
    - **`reactive`:** *객체*만을 반응형으로 만들 때 사용
    
    **`.value`를 사용하는 이유 (핵심)**
    
    `ref`를 사용하는 주된 이유는 JavaScript의 *원시 타입 값*이 *참조(reference)*가 아닌 *값 복사(value copy)* 방식으로 전달되기 때문
    
    **1. 원시 타입 값 (값 복사):**
    
    ```jsx
    let a = 10;
    let b = a; // a의 값(10)이 b에 복사됨
    b = 20;
    
    console.log(a); // 10 (b를 변경해도 a는 변경되지 않음)
    console.log(b); // 20
    
    ```
    
    - `b = a`에서 `a`의 *값*이 `b`에 *복사*
    - `b`를 변경해도 `a`는 영향을 받지 않음
    
    **2. 객체 (참조):**
    
    ```jsx
    let obj1 = { value: 10 };
    let obj2 = obj1; // obj1의 참조가 obj2에 복사됨
    obj2.value = 20;
    
    console.log(obj1.value); // 20 (obj2를 변경하면 obj1도 변경됨)
    console.log(obj2.value); // 20
    
    ```
    
    - `obj2 = obj1`에서 `obj1`이 가리키는 *객체의 참조*가 `obj2`에 *복사*
    - `obj2`를 통해 객체의 속성을 변경하면 `obj1`도 같은 객체를 가리키고 있으므로 변경된 내용이 반영
    
    **`ref`와 `.value`의 필요성 (원시 타입)**
    
    만약 Vue에서 원시 타입 값을 `ref` 없이 일반 변수로 사용한다면, 위에서 설명한 "값 복사" 문제 때문에 반응성을 잃게 됨
    
    ```tsx
    <script setup>
    let count = 0; // 일반 변수
    
    function increment() {
      count++; // count를 증가시켜도 템플릿은 업데이트되지 않음
    }
    </script>
    
    <template>
      <p>Count: {{ count }}</p>
      <button @click="increment">Increment</button>
    </template>
    
    ```
    
    `count`는 일반 변수이므로, `increment` 함수에서 `count`를 증가시켜도 Vue는 이 변화를 감지하지 못하고 템플릿을 업데이트하지 않음
    
    `ref`는 이 문제를 해결하기 위해 *객체*를 사용.
     `ref`는 내부적으로 `.value` 속성을 가진 객체를 생성하고, 이 `.value` 속성에 실제 값을 저장
    
    ```tsx
    <script setup>
    import { ref } from 'vue';
    
    const count = ref(0); // ref 객체 생성
    
    function increment() {
      count.value++; // .value를 통해 값을 변경해야 Vue가 감지
    }
    </script>
    
    <template>
      <p>Count: {{ count }}</p>
      <button @click="increment">Increment</button>
    </template>
    
    ```
    
    - `count`는 이제 `ref` 객체
    - `count.value`를 통해 값을 변경하면 Vue는 이 변화를 감지하고 템플릿을 업데이트. (객체의 속성 변경은 "참조"를 통해 이루어지므로)
    
    **`reactive`와 `.value` 불필요 (객체)**
    
    `reactive`는 *객체*를 반응형으로 만듦. 객체는 이미 "참조" 방식으로 동작하므로, `ref`처럼 `.value`를 사용할 필요가 없음
    
    ```tsx
    <script setup>
    import { reactive } from 'vue';
    
    const state = reactive({
      count: 0,
      message: 'Hello'
    });
    
    function increment() {
      state.count++; // .value 없이 직접 속성에 접근하여 변경
    }
    </script>
    
    <template>
      <p>Count: {{ state.count }}</p>
      <p>Message: {{ state.message }}</p>
      <button @click="increment">Increment</button>
    </template>
    
    ```
    
    - `state`는 `reactive`로 만들어진 반응형 객체
    - `state.count`와 같이 `.value` 없이 직접 객체의 속성에 접근하여 값을 변경할 수 있음
     Vue는 객체의 속성 변경을 감지하고 템플릿을 업데이트
    
    **정리**
    
    - `ref`는 원시 타입 값(또는 객체)을 반응형으로 만들 때 사용하며, `.value`를 통해 값에 접근해야 함 (JavaScript의 값 복사 방식 때문)
    - `reactive`는 객체를 반응형으로 만들 때 사용하며, `.value` 없이 직접 객체의 속성에 접근할 수 있음. (객체의 참조 방식 때문)
    - Vue 3 Composition API에서는 `ref`와 `reactive` 중 어떤 것을 사용해도 상관없지만, 일관성을 유지하는 것이 좋음 (예: 모든 반응형 데이터를 `ref`로 통일하거나, 객체는 `reactive`, 원시 타입은 `ref`로 사용하는 등)

vue 프로젝트 파일 구성

- root
- .vscode - vscode용 설정 파일
- dist - 배포용 파일 세트가 저장된 폴더
- node_modules - 라이브러리가 저장된 폴더
- public - 웹에 공개하는 파일을 저장하는 폴더 (ex favicon.ico 파비콘)
- src - 소스 코드 파일을 저장하는 폴더
    - assets - 이미지 등 asset 종류를 저장하는 폴더
    - components - 컴포넌트 파일을 저장하는 폴더
    - App.vue - 메인 단일 파일 컴포넌트
    - main.ts - 메인 스크립트 파일
- .eslintrc.js - ESLint 관한 파일
- .gitignore
- env.d.ts - 환경 함수를 설정하는 파일
- index.html - 메인페이지
- package-lock.json - npm 의존 관계에 대한 설정 파일
- package.json - npm에 관한 설정 파일
- tsconfig.json - 타입스크립트에 관한 설정 파일
- tsconfig.config.json - 타입스크립트에 관한 추가설정 파일
- vite.config.ts - Vite에 관한 설정 파일

배포용 파일 세트 - dist 폴더

- npm run build 를 실행해야 dist 폴더가 생성됨
- assets, index.html, favicon, js 파일, css 파일 등 웹 애플리케이션에 필요한 파일 생성됨
- yarn dev → Vue 프로젝트 폴더 구성 그대로 동작하는 내부 서버를 구동하여 사용하는 명령어
- 개발용 파일 → 배포용 파일 생성해주는 도구가 Vite

public폴더

- 그대로 dist 폴더에 저장됨
- vue.js의 처리를 통하지 않고 외부에 공개하고싶은 파일 저장

Vue 프로젝트의 작동 원리

- id 가 app인 div 태그는 vue 애플리케이션의 출발점

```tsx
import { createApp } from 'vue' // 모듈을 가져오는 부분
import App from './App.vue' // App.vue 파일을 App으로 import
import './assets/main.css' // main.css 파일을 가져옴
createApp(App).mount('#app') 
// createApp의 인자로 시작점인 단일 파일 컴포넌트 전달
// mount는 Vue 애플리케이션을 표시하는 메서드
// index.html의 어느태그에 표시할지를 인자로 지정하는데 그 태그가 app
```
