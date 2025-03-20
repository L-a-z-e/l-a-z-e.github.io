---
title: Vite와 Vue 프로젝트
description: Vue
author: laze
date: 2025-03-17 00:00:01 +0900
categories: [Dev, Vue]
tags: [Vue]
---
# 2장 Vite와 Vue 프로젝트

필요한 도구

- vscode
- vue-official plugin
- node.js

Vue 프로젝트 생성

`npm init vue@latest`

1. Project name?
2. Add TypeScript?  - 기본값 No(JS)
3. Add JSX Support? - JSX 자바스크립트 코드 중 HTML을 작성할 수 있는 확장 기능
4. Add Vue Router?
5. Add Pinia? - 데이터를 통합적으로 관리할 수 있는 라이브러리
6. Add Vitest? - 단위 테스트를 수행하기위한 라이브러리
7. Add End-to-End Testing Solution? - Cypress(UI 조작 포함 앱 전체 애플리케이션 테스트 라이브러리) 등의 도구 사용 여부
8. Add ESLint ? - 코드가 올바른지 여부를 정적으로 분석하는 도구(Linter)
9. Add Prettier ? - 소스 코드를 정형화하는 도구

cd → 생성한 프로젝트로 이동 후

`npm install`

ESLint 실행 → `npm run lint`

소스 코드의 문제점 지적 & Prettier → 소스코드 포매팅 실행

기존에는 Webpack을 사용해서 자바스크립트 환경에서 실행할 수 있는 파일로 변환하여 실행

→ Vite로 변경됨 ECMA스크립트의 모듈 구조를 최대한 활용하여 속도 증가됨
