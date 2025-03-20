---
title: 데이터와 이벤트 디렉티브
description: Vue?
author: laze
date: 2025-03-20 00:00:01 +0900
categories: [Dev, Vue]
tags: [Vue]
---
# 4장 데이터와 이벤트 디렉티브

디렉티브란?

| 디렉티브 | 역할 |
| --- | --- |
| v-bind | 데이터 바인딩 |
| v-on | 이벤트 처리 |
| v-model | 양방향 데이터 바인딩 |
| v-html | html 문자열 표시 |
| v-pre | 정척 콘텐츠 표시 |
| v-once | 데이터 바인딩을 최초 1회로 제한 |
| v-cloak | 머스태시 구문 비표시 |
| v-if | 조건 분기 |
| v-show | 표시/비표시 제어 |
| v-for | 반복 처리 |

```tsx
<script setup lang="ts">
import { ref } from "vue";

const url = ref("https://vuejs.org/");
</script>

<template>
  <p><a v-bind:href="url" target="_blank">Vue.js사이트</a></p>
  <p><a :href="url" target="_blank">Vue.js(약어)</a></p>
  <p><a v-bind:href="url + 'guide/introduction.html'" target="_blank">Vue.js 가이드페이지</a></p>
</template>
```

v-bind

- v-bind:attribute=”variable”
    - attribute 속성 값으로 템플릿변수 variable을 사용함
    - 축약형 :attribute  :만 사용
- 템플릿변수나 수식을 사용 가능
    - url + 'guide/introduction.html' 는 가능
    - “let url = ‘guide/introduction.html’” 같은 문장은 사용 불가능
- disabled, readonly 같이 속성값이 없는 것들의 바인딩
    
    ```tsx
    
    	const isSendButtonDisabled = ref(true);
    	
      <p><button type="button" v-bind:disabled="isSendButtonDisabled">전송</button></p>
    ```
    
    - disabled 속성에 대해 boolean 값을 binding → true인 경우만 속성이 부여됨
- 속성명을 템플릿 변수로 지정 가능
    
    ```tsx
    const widthOrHeight = ref("height");
    const withOrHeightValue = ref(200);
    
    <p><img alt="VueLogo" src="./assets/logo.svg" v-bind:[widthOrHeight]="withOrHeightValue"></p>
    ```
    
    - [  ]안에 템플릿 변수 작성 → 속성명 변경 가능해짐
- 여러 속성에 일괄 바인딩
    
    ```tsx
    const imgAttributes = ref({
      src: "/images/logo.svg",
      alt: "VueLogo",
      width: 75,
      height: 75
    });
    
      <p><img v-bind:="imgAttributes"></p>
      <p><img :imgAttributes title="로고"></p> // 축약형 사용하면 예상처럼 동작하지 않음
    ```
    
- 템플릿 블록과 스크립트 블록에서의 경로 변환 차이
    - assets 폴더 내의 경로를 기술하면 build 이후 → 자동으로 변환
    - 스크립트 블록에서는 자동으로 변환되지 않으므로 public 폴더 안에 넣어야함
    - 상세 설명
        
        네, 템플릿 블록과 스크립트 블록에서의 경로 변환 차이
        
        이 차이는 Vue CLI (또는 Vite)와 Webpack이 정적 자산(static assets)을 처리하는 방식 때문에 발생 함
        
        **핵심:**
        
        - **템플릿 블록 (`<template>`):** Vue Loader (또는 Vite의 해당 플러그인)가 HTML을 파싱하면서 `src` 속성과 같이 URL이 사용되는 부분을 찾아 자동으로 경로를 변환. 이 과정에서 Webpack(또는 Vite의 빌드 시스템)이 개입
        
        **상세 설명**
        
        1. **템플릿 블록 (`<template>`)에서의 경로 변환**
            
            ```
            <template>
              <img src="./assets/image.png" alt="My Image">
              <div :style="{ backgroundImage: 'url(./assets/background.jpg)' }"></div>
            </template>
            
            ```
            
            - `./assets/image.png`와 `url(./assets/background.jpg)`는 상대 경로입니다.
            - Vue Loader는 템플릿을 파싱하면서 이러한 상대 경로를 만납니다.
            - Vue Loader는 Webpack (또는 Vite의 빌드 시스템)에게 이 파일들을 처리하도록 요청합니다.
            - Webpack은 `file-loader` 또는 `url-loader`와 같은 로더를 사용하여 이 파일들을 처리합니다.
                - **`file-loader`**: 파일을 복사하고, 파일 이름에 해시를 추가하여 캐싱을 최적화합니다. 최종 경로가 생성됩니다 (예: `/img/image.d9e5f6.png`).
                - **`url-loader`**: 작은 파일 (설정된 크기 이하)의 경우, 파일을 복사하지 않고 Base64로 인코딩된 데이터 URL로 변환하여 HTML에 직접 포함시킵니다. 이렇게 하면 HTTP 요청 횟수를 줄일 수 있습니다.
            - Webpack은 변환된 경로를 템플릿에 다시 삽입합니다. 예를 들어, `<img src="./assets/image.png">`는 `<img src="/img/image.d9e5f6.png">`와 같이 변경될 수 있습니다.
            
            **장점:**
            
            - **자동 경로 관리:** 개발자는 상대 경로를 사용하여 편리하게 파일을 참조할 수 있습니다. Webpack이 빌드 시점에 올바른 경로로 변환해 줍니다.
            - **캐싱 최적화:** 파일 이름에 해시가 추가되므로, 파일 내용이 변경되지 않으면 브라우저는 캐시된 버전을 사용합니다.
            - **데이터 URL (선택 사항):** 작은 파일의 경우 데이터 URL로 변환하여 HTTP 요청을 줄일 수 있습니다.
        2. **스크립트 블록 (`<script>`)에서의 경로**
            
            ```
            <script setup>
            import { ref } from 'vue';
            
            const imageUrl = ref('./assets/image.png'); // 상대 경로 (Webpack에 의해 처리되지 않음)
            const imageUrl2 = ref('/images/image.png'); // 절대 경로, public 폴더 기준
            import imgUrl from './assets/image.png'; // import를 사용해야 Webpack이 처리
            
            const style = ref({
              backgroundImage: `url(${imageUrl.value})`,  // 이 경우 Webpack 처리가 안됨
              backgroundImage2: `url(${imgUrl})`, // import한 것은 Webpack처리됨.
            });
            
            </script>
            
            ```
            
            - `imageUrl` 변수에 할당된 `'./assets/image.png'`는 단순한 문자열입니다. Webpack은 이 문자열을 자동으로 처리하지 않습니다.
            - `imageUrl2`에 할당된 `'/images/image.png'`는 `public` 폴더를 기준으로 하는 절대 경로입니다. `public` 폴더의 내용은 빌드 시점에 변경되지 않고 그대로 복사되므로, 이 경로는 개발 환경과 배포 환경에서 모두 유효합니다.
            - `import imgUrl from './assets/image.png';`와 같이 `import` 문을 사용하면 Webpack이 이 파일을 처리하고, 변환된 경로를 `imgUrl` 변수에 할당합니다. 이렇게 해야 스크립트 블록에서도 Webpack의 경로 변환 혜택을 받을 수 있습니다.
        
        **`public` 폴더를 사용하는 이유 (스크립트 블록에서)**
        
        스크립트 블록에서 Webpack의 경로 변환을 사용하지 *않고* 파일을 참조하려면 `public` 폴더를 사용해야 하는 이유는 다음과 같습니다.
        
        - **Webpack 처리 없음:** `public` 폴더의 내용은 Webpack에 의해 처리되지 *않습니다*. 파일 이름이 변경되거나 해시가 추가되지 않습니다.
        - **루트 경로 기준:** `public` 폴더는 빌드된 결과물 (일반적으로 `dist` 폴더)의 루트에 그대로 복사됩니다. 따라서 `/images/my-image.png`와 같이 `public` 폴더를 기준으로 하는 절대 경로를 사용하면 개발 환경과 배포 환경에서 모두 동일하게 파일을 참조할 수 있습니다.
        - **정적 자산:** `public` 폴더는 일반적으로 Webpack에 의해 처리될 필요가 *없는* 정적 자산 (예: robots.txt, favicon.ico, manifest.json, 또는 자주 변경되지 않는 이미지)을 저장하는 데 사용됩니다.
        
        **정리**
        
        - **템플릿 블록:** 상대 경로 (`./assets/...`)를 사용하면 Vue Loader와 Webpack이 자동으로 경로를 변환해 줍니다. 캐싱 최적화 등의 이점이 있습니다.
        - **스크립트 블록:**
            - 상대 경로 (`./assets/...`)를 그냥 문자열로 사용하면 Webpack이 처리하지 *않습니다*.
            - Webpack의 경로 변환을 사용하려면 `import`를 사용해야 합니다 (`import imgUrl from './assets/image.png'`).
            - `public` 폴더를 기준으로 하는 절대 경로 (`/images/...`)를 사용하면 Webpack 처리 없이도 파일을 참조할 수 있습니다. `public` 폴더의 내용은 빌드 시점에 그대로 복사됩니다.
        
        이러한 차이점을 이해하면 Vue 프로젝트에서 정적 자산을 효과적으로 관리하고, 개발 환경과 배포 환경에서 경로 문제를 방지할 수 있습니다.
        
- 축약형 관련 의문
    
    맞습니다. `v-bind:`의 축약형이 `:`인 것은 맞습니다. 하지만, 이 경우에는 단순한 축약형 이상의 차이가 있습니다.  `v-bind="imgAttributes"`와 `:imgAttributes`는 *전혀 다른* 동작을 합니다.  혼란을 드려 죄송합니다. 제가 이전 답변에서 spread operator (`...`) 이야기를 잠깐 했는데, 그게 핵심입니다.  자세히 설명드리겠습니다.
    
    **`v-bind="imgAttributes"` (또는 `v-bind="{ ...imgAttributes }"`): 객체의 속성을 개별 속성으로 전달**
    
    - `v-bind`에 객체를 *직접* 전달하면, Vue는 이 객체의 *각 속성(property)*을 HTML 엘리먼트의 *개별 속성(attribute)*으로 바인딩합니다. 이것이 바로 원하는 동작입니다.
    - 내부적으로 Vue는 spread operator(`...`)를 사용하는 것과 유사하게 동작합니다. `v-bind="imgAttributes"`는 마치 `<img src="..." alt="..." width="..." height="...">`와 같이 각 속성을 풀어서 쓴 것과 같습니다. (정확히는 spread operator와 약간의 차이가 있지만, 개념적으로는 그렇습니다.)
    - 이 방식이 Vue 3에서 객체의 속성을 HTML 엘리먼트의 속성으로 전달하는 *올바른* 방법입니다.
    
    **`:imgAttributes` (또는 `v-bind:imgAttributes`):  `imgAttributes`라는 이름의 속성에 객체를 문자열로 전달 (잘못됨)**
    
    - `:imgAttributes`는 `imgAttributes`라는 *이름의* 속성을 만들고, 여기에 `imgAttributes` *객체 자체*를 값으로 할당합니다. 이때 객체는 문자열로 변환됩니다(일반적으로 `"[object Object]"`).
    - HTML 엘리먼트는 `imgAttributes`라는 속성을 갖게 되지만, 이 속성은 `src`, `alt`, `width`, `height`와 같은 유효한 이미지 속성이 아닙니다. 브라우저는 이 속성을 무시합니다.
    - 결과적으로 이미지가 표시되지 않습니다.
    
    **왜 `v-bind`를 생략하면 안 되는가?**
    
    `v-bind`를 생략하고 `:`만 사용하면, Vue는 해당 표현식을 *문자열 리터럴*로 해석하지 않고, *JavaScript 표현식*으로 해석합니다.  그리고 그 표현식의 *결과값*을 해당 속성에 할당합니다.  `v-bind`가 없으면, Vue는 `imgAttributes`라는 이름을 가진 속성에 JavaScript 표현식 `imgAttributes`의 평가 결과 (즉, 객체 자체)를 할당하려고 시도하는 것입니다.
    
    **예시로 다시 보기**
    
    ```
    <script setup>
    const imgAttributes = {
      src: '/images/logo.svg',
      alt: 'Vue Logo',
      width: 100,
      height: 100
    };
    </script>
    
    <template>
      <!-- 1. 올바른 방법: imgAttributes 객체의 각 속성을 개별 속성으로 전달 -->
      <img v-bind="imgAttributes">
    
      <!-- 2. 내부적으로 1번과 거의 동일 (spread operator 사용) -->
      <img :src="imgAttributes.src" :alt="imgAttributes.alt" :width="imgAttributes.width" :height="imgAttributes.height">
    
      <!-- 3. 잘못된 방법: imgAttributes라는 이름의 속성에 객체(문자열)를 할당 -->
      <img :imgAttributes="imgAttributes">  <!-- imgAttributes="[object Object]" -->
      <img v-bind:imgAttributes="imgAttributes"> <!-- imgAttributes="[object Object]" -->
    
        <!-- 4.  잘못된 방법: imgAttributes 라는 속성을 동적으로 바인딩 -->
        <!-- imgAttributes라는 attribute가 없으므로, 무시된다. -->
        <!-- <img :[imgAttributes]="'someValue'"> -->
    </template>
    
    ```
    
    **결론**
    
    - 객체의 속성을 HTML 엘리먼트의 개별 속성으로 전달하려면 반드시 `v-bind="objectName"` (또는 `v-bind="{ ...objectName }"`) 형식을 사용해야 합니다.
    - `:attributeName="objectName"` 또는 `v-bind:attributeName="objectName"` 형식은 `attributeName`이라는 이름의 속성에 객체를 문자열로 할당하는 것이므로, 이미지 속성을 설정하는 데는 *잘못된* 방법입니다.
    
    이제 `v-bind`를 생략했을 때 왜 다르게 동작하는지 명확하게 이해하셨기를 바랍니다. Vue의 데이터 바인딩은 강력하지만, 이런 미묘한 차이점을 알아두는 것이 중요합니다.
    
- style 속성 바인딩
    - `style=”color: white; background-color: black;”`
    
    ```tsx
    const msg = ref("style속성 테스트");
    const msgTextRed = ref("red");
    const msgTextColor = ref("white");
    const msgBoxColor = ref("black");
    const msgStyles = ref({
      color: "white",
      backgroundColor: "black"
    });
    const msgStyle2 = ref ({
      fontSize: "24pt"
    });
    const msgStyle3 = ref ({
      color: "pink",
      fontSize: "24pt"
    });
    const textSize = computed(
      (): string => {
        const size = Math.round(Math.random() * 25) + 10;
        return `$(size)pt`;
      }
    )
    
    <p v-bind:style="{ color: msgTextRed }">{{ msg }}</p> // {속성: 템플릿변수}
    <p v-bind:style="{ color: 'pink' }">{{ msg }}</p> // {속성: '리터럴'}
    <p v-bind:style="{ fontSize: textSize }">{{ msg }}</p> // 계산형속성
    <p v-bind:style="{ color: msgTextColor, backgroundColor: msgBoxColor }">{{ msg }}</p> // 여러개 지정시 ,로 구분
    <p v-bind:style="{ color: msgTextColor, 'background-color': msgBoxColor }">{{ msg }}</p> // 케밥표기법 사용시 속성 ''로 감싸기
    <p v-bind:style="msgStyles">{{ msg }}</p> // 객체리터럴 그대로 사용
    <p v-bind:style="[msgStyles, msgStyle2]">{{ msg }}</p> // 여러 객체 리터럴을 배열로 사용
    <p v-bind:style="[msgStyles, msgStyle3]">{{ msg }}</p> // 중복된 속성은 나중 것을 덮어 씀
    <p v-bind:style="[msgStyle2, msgStyle3]">{{ msg }}</p>
    
    ```
    
    - 복수의 템플릿 변수에 바인딩하는 구조가 필요함
    - { } 를 사용해서 style속성과 설정 값을 콜론으로 구분하여 작성
    - 캐멀표기법 backgroundColor → 케밥표기법 background-color
        - 원래는 케밥표기법이지만 style 속성이 객체 리터럴 형태이고 객체 속성명에는 -를 사용할 수 없어서 속성명으로 사용함 → ‘ ‘ 로 둘러서 케밥표기법으로도 사용 가능
- class 속성 바인딩
    
    ```tsx
    <script setup lang="ts">
    import { ref, computed } from "vue";
    
    const msg = ref("class binding");
    const isTextColorRed = ref(true);
    const isBgColorBlue = ref(false);
    const styles = ref({
      textColorRed: false,
      bgColorBlue: true
    });
    const computedStyles = computed(
      (): { textColorRed: boolean; bgColorBlue: boolean; } => {
        const randText = Math.round(Math.random());
        let textColorFlag = true;
        if (randText == 0) {
          textColorFlag = false;
        }
    
        const randBg = Math.round(Math.random());
        let bgColorFlag = true;
        if (randBg == 0) {
          bgColorFlag = false;
        }
    
        return {
          textColorRed: textColorFlag,
          bgColorBlue: bgColorFlag
        };
      }
    )
    
    </script>
    
    <template>
      <p v-bind:class="{textColorRed: true, bgColorBlue: true}">{{ msg }}</p>
      <p v-bind:class="{textColorRed: isTextColorRed, bgColorBlue: isBgColorBlue}">{{ msg }}</p>
      <p v-bind:class="{textColorPink: true}">{{ msg }}</p>
      <p v-bind:class="{'text-color-pink': true}">{{ msg }}</p>
      <p class="textSize24" v-bind:class="styles">{{ msg }}</p>
      <p v-bind:class="computedStyles">{{ msg }}</p>
    </template>
    
    <style>
      .textColorRed {
        color: red;
      }
    
      .text-color-pink {
        color: pink;
      }
    
      .bgColorBlue {
        background-color: blue;
      }
    
      .textSize24 {
        font-size: 24px;
      }
    </style>
    ```
    
    - 스타일 클래스 적용 → true / false 로 지정
        - `v-bind:class=”{style class name: true/false, …}”`
        - true/false 값을 템플릿 변수로 사용 가능
        - style과 다르게 케밥 ↔ 캐멀 변환이 안됨 따라서 실제 지정한 style class 속성대로 선언하되 케밥은 ‘’를 사용할 것
        - computed를 통해 동적으로 속성 바인딩 사용 가능

v-on 이벤트 디렉티브

- `v-on:이벤트명=”메서드명”` / v-on 약어 : @
    
    ```tsx
    <script setup lang="ts">
    import { ref } from "vue";
    
    const randValue = ref("시작 전");
    const onButtonClick = (): void => {
      const rand = Math.round(Math.random() * 10);
      randValue.value = String(rand);
    }
    
    </script>
    
    <template>
      <section>
        <button v-on:click="onButtonClick">Click</button>
        <p>result : {{  randValue }}</p>
      </section>
    </template>
    ```
    
- 메서드 정의
    
    ```tsx
    const 메서드명 = (): void => {
    	처리내용
    };
    ```
    
- 이벤트 (자바스크립트에서 제공하는 이벤트)

| 분류 | 이벤트명 | 발생 시점 |
| --- | --- | --- |
| 포커스 | blur | 포커스를 잃었을 때 |
| 포커스 | focus | 포커스를 받았을 때 |
| 포커스 | focusin | 포커스를 받았을 때(부모 요소에서도 이벤트 감지 가능) |
| 포커스 | focusout | 포커스를 잃었을 때(부모 요소에서도 이벤트 감지 가능) |
| 마우스 | click | 클릭했을 때 |
| 마우스 | contextmenu | 컨텍스트 메뉴가 표시될 때 |
| 마우스 | dbclick | 더블클릭했을 때 |
| 마우스 | mousedown | 마우스 버튼을 눌렀을 때 |
| 마우스 | mouseenter | 마우스 포인터가 요소에 들어갔을 때(자체 요소만 해당) |
| 마우스 | mouseleave | 마우스 포인터가 요소를 벗어났을 때(자체 요소만 해당) |
| 마우스 | mousemove | 마우스 포인터가 움직일 때 |
| 마우스 | mouseout | 마우스 포인터가 요소를 벗어났을 때(자식 요소 포함) |
| 마우스 | mouseover | 마우스 포인터가 요소에 들어갔을 때(자식 요소 포함) |
| 마우스 | mouseup | 마우스 버튼을 놓았을 때 |
| 마우스 | wheel | 마우스 휠이 회전할 때 |
| 입력 | change | 드롭다운 등으로 입력 내용이 변경되었을 때 |
| 입력 | compositionend | IME를 사용하여 입력을 종료할 때 |
| 입력 | compositionstart | IME를 사용하여 입력을 시작할 때 |
| 입력 | compositionupdate | IME를 사용하여 입력 중일 때 |
| 입력 | input | 입력  내용이 업데이트될 때 |
| 입력 | keydown | 키를 눌렀을 때 |
| 입력 | keypress | 키를 눌러 문자가 입력될 때 |
| 입력 | keyup | 키를 놓았을 때 |
| 입력 | select | 텍스트가 선택되었을 때 |
| 기타 | resize | 요소의 크기가 변경되었을 때 |
| 기타 | scroll | 스크롤할 때 |
- 이벤트 객체를 메서드 인자로 받기
    
    ```tsx
    <script setup lang="ts">
    import { ref } from "vue";
    
    const mousePointerX = ref(0);
    const mousePointerY = ref(0);
    const onImgMousemove = (event: MouseEvent): void => {
      mousePointerX.value = event.offsetX;
      mousePointerY.value = event.offsetY;
    }
    </script>
    
    <template>
      <section>
        <img src="./assets/logo.svg" alt="Vue logo" width="200" @mousemove="onImgMousemove">
        <p>Pointer : x = {{ mousePointerX }} / y = {{ mousePointerY }}</p>
      </section>
    </template>
    ```
    
- 이벤트 객체 이외의 인자를 받는 이벤트 핸들러 메서드
    
    ```tsx
    const pBgColor = ref("white");
    const onPClick = (bgColor: string): void => {
      pBgColor.value = bgColor;
    }
    
    <p @click="onPClick('red')" :style="{ backgroundColor: pBgColor }">
      클릭-> 색상변경
    </p>
    ```
    
- 이벤트객체 + 이외 인자 같이 사용
    
    ```tsx
    const pMsg = ref("이벤트 전 (Click!)");
    const pBgColorEvent = ref("white");
    const onPClickWithEvent = (bgColor: string, event: MouseEvent): void => {
      pBgColorEvent.value = bgColor;
      pMsg.value = event.timeStamp.toString();
    };
    
    <p @click="onPClickWithEvent('green', $event)" :style="{ backgroundColor: pBgColorEvent }">{{ pMsg }}</p>
    ```
    
    - $event 인자 ⇒ 이벤트 객체를 인자로 자동으로 전달해주는 것은 인자를 생략하고 메서드명만 작성한 경우 (@mousemove="onImgMousemove")
    - 다른 인자와 같이 쓸 때에는 이벤트 객체로 $event를 명시해야함, $event의 순서는 상관없음
- v-on 수식어
    - 수식어 → 인자 뒤에 .로 연결하여 작성하는 키워드
    
    | 수식어 | 내용 |
    | --- | --- |
    | stop | 이벤트 버블링 취소(event.stopPropagation()와 동일) |
    | capture | 이벤트 리스너를 캡처 모드로 설정 |
    | self | 이 요소에서 이벤트가 발생했을 때만 이벤트를 실행 |
    | prevent | 이벤트의 기본 처리 취소(event.preventDefault()와 동일) |
    | passive | 해당 이벤트의 기본 동작을 즉시 실행 |
    | once | 이벤트 실행을 1회로 제한 |
    - DOM 이벤트 단계
        1. 캡처 단계
            1. button 태그를 클릭하면 window 객체가 제일 먼저 이벤트를 감지
            2. 차례대로 요소들 내부로 들어가서 이벤트 발생지까지 추적
                1. window → body → section → div → button
        2. 타깃 단계
            1. 최종적으로 이벤트의 발생지를 파악
            2. 특정된 발생지에서 해당 태그에 등록된 이벤트 핸들러 호출됨
        3. 버블링 단계
            1. 타깃 단계 이후 캡처 단계의 역순으로 부모 요소로 거슬러 올라감(이벤트의 전파)
            2. 마지막으로 window 객체에 도달하게 됨
            3. 기본 설정은 버블링 단계로 되어있음 → 이걸 캡쳐단계에서 실행하기위해 옵션을 설정하기도 함
        - stop - 버블링 취소, 부모 요소에 설정된 이벤트 핸들러가 실행되는 것을 막음
        - capture - 캡쳐단계에서 해당 이벤트 핸들러 실행 → 버블링 단계에서는 실행 X
        - self - 해당 요소 자체가 이벤트 발생지일 때만 이벤트 핸들러 실행
        - prevent - onFormSubmit 실행 이후 action도 실행되어 URL /# 이됨 ⇒ prevent를 통해 원래 이벤트(action) 전송을 취소
            
            ```tsx
            const msg = ref("미전송");
            const onFormSubmit = (): void => {
              msg.value = "전송";
            }
            
            <form action="#" @submit.prevent="onFormSubmit">
              <input type="text" required>
              <button type="submit">전송</button>
            </form>
            <p>{{ msg }}</p>
            ```
            
        - passive - 이벤트 처리 함수 내에서 preventDefault() 메서드를 실행하지 않았음을 브라우저에 알림 ⇒ 브라우저가 즉시 이벤트 처리 할 수 있게됨 (scroll 이벤트에 적용하면 스크롤 처리가 즉시되어 끊김을 줄일 수 있음)
    - 키 이벤트 @keydown.q=”메서드” ⇒ q 키 누를때 이벤트 처리
        - 특수키 enter, tab, delete, esc, space, up, down, left, right
    - 클릭 이벤트 수식어
        - left, right, middle
    - 시스템 수식어
        - ctrl, alt, shift, meta(맥 - cmd, 윈도우 - win)
    - exact
        - 여러키를 눌러도 한 키가 포함되는 경우 이벤트가 원래 발생하지만 단독으로 누를때로 제한하고 싶을 떄 `@keydown.enter.exact`