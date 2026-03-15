---
title: "Vue + React 프론트엔드의 에러 처리 통합"
description: "Vue 3과 React 18이 공존하는 Module Federation 환경에서 에러 처리를 Design System 레벨로 통합한 과정"
author: laze
date: 2026-03-12 13:30:00 +0900
categories: [Dev, Frontend]
tags: [ErrorHandling, React, Vue]
---

## 문제 상황

프론트엔드가 6개 앱으로 구성되어 있습니다. Vue 3이 4개(portal-shell, blog, admin, drive), React 18이 3개(shopping, prism, shopping-seller)입니다. 각 앱이 에러 처리를 제각각으로 구현하고 있었습니다.

- 어떤 앱은 `try-catch`로 잡아서 `console.error`만 호출
- 어떤 앱은 `window.onerror`에 전역 핸들러를 등록
- 어떤 앱은 에러 핸들링 자체가 없음

Module Federation 환경에서 Remote 앱의 미처리 에러가 Host까지 전파되면, 전체 포털이 먹통이 됩니다. 에러 처리를 표준화할 필요가 있었습니다.

## 설계: 3계층 에러 처리

대안을 먼저 검토했습니다. 각 앱이 자체적으로 에러 처리를 구현하는 방식은 중복 코드와 일관성 부재가 문제였고, 단일 전역 핸들러(`window.onerror`)로 통합하는 방식은 프레임워크별 컴포넌트 트리 정보를 잃어서 에러 원인 추적이 어려웠습니다. 결국 프레임워크 비의존 로깅(공통) → 프레임워크별 Error Boundary(격리) → 앱별 설정(적용)으로 나누는 3계층 구조를 선택했습니다.

### 계층 1 — Framework-Agnostic Logger (design-core)

프레임워크에 의존하지 않는 로거를 design-core에 정의합니다. 이 로거는 에러 리포팅 서비스(Sentry 등)와의 연결 포인트이기도 합니다.

```typescript
// design-core/src/types/logger.ts
export type LogLevel = 'debug' | 'info' | 'warn' | 'error'

export interface ErrorReporter {
  captureError(error: unknown, context?: Record<string, unknown>): void
  captureMessage(message: string, level: LogLevel): void
}

export interface LoggerOptions {
  moduleName: string
  level?: LogLevel
  reporter?: ErrorReporter  // Sentry 등 외부 리포터 주입
}

export function createLogger(options: LoggerOptions): Logger {
  const { moduleName, level = 'info', reporter } = options
  return {
    debug: (...args) => { if (shouldLog('debug', level)) console.debug(`[${moduleName}]`, ...args) },
    info:  (...args) => { if (shouldLog('info', level))  console.info(`[${moduleName}]`, ...args) },
    warn:  (...args) => { if (shouldLog('warn', level))  console.warn(`[${moduleName}]`, ...args) },
    error: (error, context) => {
      console.error(`[${moduleName}]`, error, context)
      reporter?.captureError(error, { module: moduleName, ...context })
    },
  }
}
```

프로덕션 빌드에서는 Vite의 `esbuild.pure` 설정으로 `console.log`와 `console.debug`를 자동 제거합니다.

### 계층 2 — Framework-Specific Error Boundary

#### React: ErrorBoundary 컴포넌트

React의 Error Boundary는 클래스 컴포넌트의 `componentDidCatch` 라이프사이클을 사용합니다. design-react에서 공통 ErrorBoundary를 제공합니다.

```tsx
// design-react/src/components/ErrorBoundary.tsx
interface ErrorBoundaryProps {
  moduleName: string
  reporter?: ErrorReporter
  onError?: (error: Error, errorInfo: ErrorInfo) => void
  fallback?: ReactNode
  children: ReactNode
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  private logger: Logger

  constructor(props: ErrorBoundaryProps) {
    super(props)
    this.state = { hasError: false }
    this.logger = createLogger({
      moduleName: props.moduleName,
      reporter: props.reporter,
    })
  }

  static getDerivedStateFromError(): ErrorBoundaryState {
    return { hasError: true }
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    this.logger.error(error, { componentStack: errorInfo.componentStack })
    this.props.onError?.(error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div>
          <p>Something went wrong</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Try Again
          </button>
        </div>
      )
    }
    return this.props.children
  }
}
```

#### Vue: setupErrorHandler + ErrorBoundary

Vue에서는 두 가지를 조합합니다.

**전역 에러 핸들러**: `app.config.errorHandler`로 Vue 컴포넌트 내 에러와 `unhandledrejection`을 포착합니다. `unhandledrejection` 리스너 덕분에 `onErrorCaptured`가 잡지 못하는 비동기 에러(Promise rejection)도 보완됩니다.

```typescript
// design-vue/src/composables/useErrorHandler.ts
export function setupErrorHandler(app: App, options: ErrorHandlerOptions): void {
  const logger = createLogger(options)

  app.config.errorHandler = (err, instance, info) => {
    logger.error(err, {
      component: instance?.$options?.name,
      info,
    })
  }

  window.addEventListener('unhandledrejection', (event) => {
    logger.error(event.reason, { type: 'unhandledrejection' })
  })
}
```

**컴포넌트 레벨 ErrorBoundary**: `onErrorCaptured` 훅으로 특정 컴포넌트 트리의 에러를 잡습니다. React ErrorBoundary와 달리 `createLogger`를 사용하지 않는데, 이는 의도적인 역할 분리입니다. 컴포넌트 레벨 ErrorBoundary는 UI 격리만 담당하고, 구조화된 로깅과 외부 리포팅은 `setupErrorHandler`의 `createLogger`가 전담합니다.

```vue
<!-- design-vue/src/components/ErrorBoundary/ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const props = withDefaults(defineProps<{
  fallbackMessage?: string
}>(), {
  fallbackMessage: 'Something went wrong',
})

const hasError = ref(false)

onErrorCaptured((err) => {
  console.error('[ErrorBoundary]', err)
  hasError.value = true
  return false  // 상위 전파 중단
})

function retry() {
  hasError.value = false
}
</script>

<template>
  <div v-if="hasError">
    <p>{{ fallbackMessage }}</p>
    <button @click="retry">Try Again</button>
  </div>
  <slot v-else />
</template>
```

### 계층 3 — 앱별 적용

각 앱의 진입점에서 한 줄로 에러 처리를 설정합니다.

```typescript
// Vue 앱 (portal-shell, blog, admin, drive)
import { setupErrorHandler } from '@portal/design-vue'

const app = createApp(App)
setupErrorHandler(app, { moduleName: 'Portal' })
```

```tsx
// React 앱 (shopping, prism, shopping-seller)
import { ErrorBoundary } from '@portal/design-react'

function App() {
  return (
    <ErrorBoundary moduleName="Shopping">
      <ShoppingRouter />
    </ErrorBoundary>
  )
}
```

## 에러 전파 흐름

```
컴포넌트 에러 발생
    ↓
ErrorBoundary 포착 (React: componentDidCatch / Vue: onErrorCaptured)
    ↓
createLogger()로 로깅 ([Shopping] Error: ...)
    ↓
reporter?.captureError() → 외부 모니터링 (Sentry 등)
    ↓
Fallback UI 표시 ("Something went wrong" + "Try Again")
```

Remote 앱에서 에러가 발생해도 ErrorBoundary가 포착하므로, Host(portal-shell)까지 에러가 전파되지 않습니다. 해당 Remote 영역만 fallback UI로 대체되고 나머지 포털은 정상 동작합니다.

## useLogger Hook

컴포넌트 내부에서 로깅이 필요할 때는 framework별 hook을 사용합니다.

```typescript
// React
import { useLogger } from '@portal/design-react'

function CartPage() {
  const logger = useLogger({ moduleName: 'Cart' })
  // logger.info('Cart loaded', { itemCount: 3 })
  // logger.error(err, { action: 'addToCart' })
}

// Vue
import { useLogger } from '@portal/design-vue'

const logger = useLogger({ moduleName: 'Blog' })
```

## 교훈

**1. 에러 처리의 추상화 레벨을 맞춰라**

Logger는 프레임워크 비의존(design-core), Error Boundary는 프레임워크 의존(design-vue/react), 앱별 설정은 진입점에서 한 줄. 각 레벨에서 담당하는 역할이 명확합니다.

**2. Module Federation에서 ErrorBoundary는 방화벽이다**

일반 SPA와 달리 MF 환경에서는 Remote 앱이 독립 빌드·배포됩니다. Host가 Remote의 코드 품질을 통제할 수 없으므로, Host 보호를 위해 각 Remote 마운트 지점에 ErrorBoundary를 두는 것이 필수입니다. Remote 하나가 죽더라도 나머지 포털 기능은 유지됩니다.

**3. 프로덕션에서 console.log는 제거하라**

Vite의 `esbuild.pure: ['console.log', 'console.debug']` 설정으로 빌드 타임에 자동 제거됩니다. `console.error`와 `console.warn`은 유지하여 프로덕션 디버깅에 활용합니다.
