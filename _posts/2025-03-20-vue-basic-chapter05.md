---
title: 양방향 데이터 바인딩
description: Vue?
author: laze
date: 2025-03-20 13:00:00 +1300
categories: [Dev, Vue]
tags: [Vue]
---
# 5장 양방향 데이터 바인딩

**v-model**

- 양방향 데이터 바인딩
    - 입력 내용과 템플릿 변수간의 바인딩
        
        ```tsx
        <script setup lang="ts">
        import { ref } from "vue"
        
        const inputNameModel = ref("양방향");
        </script>
        
        <template>
          <section>
            <input type="text" v-model="inputNameModel">
            <p>{{ inputNameModel }}</p>
          </section>
        </template>
        ```
        
    - 단방향과 차이
        
        ```tsx
        const inputNameBind = ref("홍길동");
        const inputNameOn = ref("null");
        
        const onInputName = (event: Event): void => {
          const element = event.target as HTMLInputElement
          inputNameOn.value = element.value;
        }
        
        <div>
          <input type="text" v-bind:value="inputNameBind">
        </div>
        
        <div>
          <input type="text" v-on:input="onInputName">
          <p>{{ inputNameOn }}</p>
        </div>
        ```
        
    - 문자열 외 다른 입력 컨트롤에서의 v-model
        
        ```tsx
        const inputTextArea = ref("텍스트 영역 문자 입력 \n(개행추가)");
        const memberType = ref(1);
        const memberTypeSelect = ref(1);
        const isAgreed = ref(false);
        const isAgreed01 = ref(0);
        const selectedOS = ref([]);
        const selectedOSSelct = ref([]);
        
        <div>
          <textarea v-model="inputTextArea"></textarea>
          <textarea>{{ inputTextArea }}</textarea>
          <br>
          <section>
            <label><input type="radio" name="memberType" value="1" v-model="memberType">일반회원</label>
            <label><input type="radio" name="memberType" value="2" v-model="memberType">특별회원</label>
            <label><input type="radio" name="memberType" value="3" v-model="memberType">우수회원</label>
            <br>
            <p>selected radio button : {{ memberType }}</p>
          </section>
          <br>
          <section>
            <select v-model="memberTypeSelect">
              <option value="1">일반회원</option>
              <option value="2">특별회원</option>
              <option value="3">우수회원</option>
            </select>
            <br>
            <p>선택된 리스트: {{ memberTypeSelect }}</p>
          </section>
          <br>
          <section>
            <label><input type="checkbox" v-model="isAgreed">동의</label>
            <p>동의 결과 : {{ isAgreed }}</p>
          </section>
          <section>
            <label><input type="checkbox" v-model="isAgreed01" true-value="1" false-value="0">동의</label>
            <p>동의 결과 : {{ isAgreed01 }}</p>
          </section>
          <section>
            <label><input type="checkbox" v-model="selectedOS" value="1">macOS</label>
            <label><input type="checkbox" v-model="selectedOS" value="2">Windows</label>
            <label><input type="checkbox" v-model="selectedOS" value="3">Linux</label>
            <label><input type="checkbox" v-model="selectedOS" value="4">iOS</label>
            <label><input type="checkbox" v-model="selectedOS" value="5">Android</label>
            <p>선택된 OS: {{ selectedOS }}</p>
          </section>
          <section>
            <select v-model="selectedOSSelct" multiple>
              <option value="1">macOS</option>
              <option value="2">Windows</option>
              <option value="3">Linux</option>
              <option value="4">iOS</option>
              <option value="5">Android</option>
            </select>
            <p>선택된 OS: {{ selectedOSSelct }}</p>
          </section>
          
        </div>
        ```
        
- v-model 수식어
    
    
    | 수식어 | 내용 |
    | --- | --- |
    | lazy | input 대신 change 이벤트로 양방향 데이터 바인딩 수행 
    → 모두 입력을 마치고 입력 컨트롤에서 포커스가 사라질 때 |
    | number | 입력값을 숫자로 취급 |
    | trim | 입력값의 전후 공백 제거 |
    
- v-html
    - html 문자열을 그대로 표시
    
    ```tsx
    <script setup lang="ts">
    import { ref } from "vue";
    
    const htmlStr = ref(`<a href="https://vuejs.org//">Vue.js 페이지 </a>`);
    </script>
    
    <template>
      <section>{{ htmlStr }}</section>
      <section v-html="htmlStr"></section>
    </template>
    ```
    

- v-pre
    - 머스태시 구문을 포함한 하위 태그의 모든 템플릿 비활성화하고 그대로 표시
    
    ```tsx
      <section v-pre>
        <p v-on:click="showHello">{{ hello }}</p>
      </section> // 그대로 출력됨 <p v-on:click="showHello">{{ hello }}</p>
    ```
    

- v-once
    - 처음 한 번만 데이터 바인딩 수행하는 디렉티브
    
    ```tsx
        <section>
          <input type="number" v-model="price">원<br>
          <p>금액은 {{ price }}원 입니다.</p>
          <p v-once>(once)금액은 {{ price }}원 입니다.</p> // 처음 값 유지
        </section>
    ```
    

- v-cloak
    - 값이 임베디드되기 전에 머스태시 구문이 그대로 표시되는 것을 방지
    
    ```tsx
    <script>
    	const hello = ref("hello");
    </script>
    
    <template>
        <p v-cloak>{{ hello }}</p>
    </template>
        
    <style>
    	[v-cloak] {
    	  display: none;
    	}
    </style>
    ```