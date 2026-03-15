---
title: "Design System 4패키지를 3패키지로 통합한 이유"
description: "design-tokens, design-types, design-system-vue, design-system-react 4패키지를 design-core, design-vue, design-react 3패키지로 통합하고 variant를 SSOT로 관리한 과정 기록."
author: laze
date: 2026-03-12 12:30:00 +0900
categories: [Dev, Frontend]
tags: [DesignSystem, Tailwind, SSOT]
---

## 기존 구조의 문제

기존에는 4개 패키지로 Design System을 운영했습니다.

```
design-tokens         → 디자인 토큰 (색상, 간격 등)
design-types          → TypeScript 타입 정의
design-system-vue     → Vue 컴포넌트
design-system-react   → React 컴포넌트
```

문제는 **variant 클래스의 중복 정의**였습니다. Button의 `primary`, `secondary` 등 variant에 해당하는 Tailwind 클래스 조합이 Vue와 React 패키지에 각각 정의되어 있었습니다. 하나를 수정하면 다른 쪽도 수정해야 하고, 이 동기화가 누락되면 프레임워크별로 같은 variant인데 다르게 보이는 상황이 발생했습니다.

토큰과 타입도 별도 패키지로 분리되어 있어서, 빌드 순서가 `tokens → types → vue/react`로 4단계였고, 간단한 토큰 변경에도 세 패키지를 모두 다시 빌드해야 했습니다.

## 통합 전략: SSOT를 하나의 패키지에

핵심 원칙은 **variant, 타입, 토큰을 한 곳에서 관리(Single Source of Truth)**하고, Vue/React 패키지는 이를 가져다 쓰기만 하는 것입니다.

```
design-core   → 토큰 + 타입 + variant (SSOT)
design-vue    → Vue 컴포넌트 (design-core의 variant import)
design-react  → React 컴포넌트 (design-core의 variant import)
```

### design-core의 구성

```typescript
// design-core/src/index.ts
export * from './types/common'      // Size, ButtonVariant, TabVariant 등
export * from './types/components'  // ButtonProps, InputProps, TabsProps 등
export * from './types/api'         // ApiResponse, PageResponse 등
export * from './variants'          // buttonVariants, cardVariants, tabsVariants 등
export { cn } from './utils'        // Tailwind class merge utility
```

`package.json`의 `exports`로 세분화된 진입점을 제공합니다.

```json
{
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" },
    "./css": "./dist/tokens.css",
    "./json": "./dist/tokens.json",
    "./tailwind": "./tailwind.preset.js",
    "./variants": { "types": "./dist/variants/index.d.ts", "import": "./dist/variants/index.js" }
  }
}
```

### Variant 정의 예시

variant는 Tailwind 클래스 문자열의 Record 매핑입니다.

```typescript
// design-core/src/variants/button.ts
export const buttonBase = [
  'inline-flex items-center justify-center',
  'font-medium rounded-md',
  'transition-all duration-150 ease-out',
  'focus:outline-none focus-visible:ring-2 focus-visible:ring-brand-primary',
].join(' ')

export const buttonVariants: Record<ButtonVariant, string> = {
  primary: [
    'bg-white/90 text-text-inverse',
    'hover:bg-white',
    'active:bg-white/80 active:scale-[0.98]',
    'light:bg-brand-primary light:text-white',
    'border border-transparent shadow-sm',
  ].join(' '),

  secondary: [
    'bg-transparent text-text-body',
    'hover:bg-white/5 hover:text-text-heading',
    'border border-border-default',
  ].join(' '),
  // ghost, outline, danger, error ...
}

export const buttonSizes: Record<Exclude<Size, 'xl'>, string> = {
  xs: 'h-6 px-2 text-xs gap-1',
  sm: 'h-8 px-3 text-sm gap-1.5',
  md: 'h-9 px-4 text-sm gap-2',
  lg: 'h-11 px-5 text-base gap-2',
}
```

Vue와 React 패키지에서는 이 variant를 import하여 사용합니다.

```typescript
// design-vue 또는 design-react에서
import { buttonBase, buttonVariants, buttonSizes, cn } from '@portal/design-core'

const classes = cn(buttonBase, buttonVariants[variant], buttonSizes[size])
```

`cn()`은 `clsx` + `tailwind-merge` 조합으로, 중복 클래스를 자동 병합합니다.

### Tailwind Content Scan 설정

variant 클래스는 design-core 패키지에 정의되어 있으므로, 각 앱의 `tailwind.config.js`에서 이 경로를 content scan에 포함해야 합니다. 누락하면 variant에 사용된 Tailwind 클래스가 빌드에서 제거됩니다.

```javascript
// 각 앱의 tailwind.config.js
module.exports = {
  content: [
    './src/**/*.{vue,tsx,ts}',
    '../design-core/src/variants/**/*.ts',  // 필수
    '../design-core/src/styles/**/*.css',
  ],
  presets: [require('../design-core/tailwind.preset.js')],
}
```

이 설정을 빠뜨린 적이 있는데, tabs variant의 padding이 적용되지 않아 레이아웃이 깨졌습니다. 이 사고 이후 content scan 경로를 체크리스트 항목으로 추가했습니다.

## 빌드 순서 변화

```
변경 전: tokens → types → vue → react  (4단계)
변경 후: design-core → vue + react       (2단계)
```

design-core 빌드가 끝나면 vue와 react는 병렬 빌드가 가능합니다. 실무적으로는 design-core 하나만 빌드하면 Vue/React 양쪽이 최신 variant를 즉시 사용할 수 있어, 토큰이나 variant를 수정한 뒤 확인까지의 피드백 루프가 짧아졌습니다.

## 공유 타입의 범위

design-core는 UI 컴포넌트 타입뿐 아니라, 프로젝트 전체의 공유 인터페이스 계층 역할도 합니다. Vue/React 앱, bridge 라이브러리 등에서 공통으로 사용되는 타입은 다른 마땅한 공유 패키지가 없으므로 design-core에 포함했습니다.

```typescript
// API 응답 타입
export interface ApiResponse<T> {
  success: true
  data: T
  error: null
}

// 페이지네이션
export interface PageResponse<T> {
  items: T[]
  page: number
  size: number
  totalElements: number
  totalPages: number
}

// 로거
export interface Logger {
  debug(...args: unknown[]): void
  info(...args: unknown[]): void
  warn(...args: unknown[]): void
  error(error: unknown, context?: Record<string, unknown>): void
}
```

## 교훈

**1. Tailwind content scan 누락은 조용히 실패한다**

빌드 에러 없이 클래스가 적용되지 않으므로, UI가 깨져야 비로소 알게 됩니다. variant 파일 경로를 content scan에 포함하는 것을 체크리스트에 넣어야 합니다.

**2. 패키지 수를 줄이면 빌드와 의존성 관리가 단순해진다**

4패키지에서 3패키지로의 변화가 작아 보이지만, 빌드 단계가 4에서 2로 줄고, 패키지 간 버전 동기화 포인트도 줄어듭니다. 다만 design-core가 변경되면 Vue/React 양쪽 재빌드를 강제하므로, 변경 범위가 한쪽 프레임워크에만 해당하더라도 전체가 영향받는 트레이드오프가 있습니다.
