---
title: "Module Federation에서 Vue Host와 React Remote 간 인증 상태 공유"
description: "Vue 3 Host와 React 18 Remote가 공존하는 Module Federation 환경에서 인증/테마 상태를 프레임워크 경계 없이 공유한 2-Layer Bridge 아키텍처 기록"
author: laze
date: 2026-03-12 11:30:00 +0900
categories: [Dev, MicroFrontend]
tags: [ModuleFederation, Vue, React]
---

## 문제 상황

Portal Universe의 프론트엔드는 Module Federation으로 구성됩니다. Vue 3로 된 Host(portal-shell)가 React 18로 된 Remote 앱들(shopping, prism, shopping-seller)을 로드합니다.

```
portal-shell (Host, Vue 3, :30000)
├── blog-frontend (Remote, Vue 3)
├── shopping-frontend (Remote, React 18)
├── prism-frontend (Remote, React 18)
└── shopping-seller-frontend (Remote, React 18)
```

Host에서 로그인하면 Pinia의 `useAuthStore`에 인증 상태가 저장됩니다. 문제는 React Remote에서 이 상태를 어떻게 읽느냐입니다. React는 Pinia를 사용할 수 없고, 심지어 Vue Remote도 자체 Pinia 인스턴스를 생성하면 Host의 store와 동기화되지 않습니다(Dual Pinia Instance 문제).

초기에는 `window.__PORTAL_AUTH_STATE__` 같은 전역 변수로 상태를 공유했는데, 이 방식은 상태 변경을 반응적으로 감지할 수 없고, 전역 네임스페이스를 오염시킵니다. 실제로 Host에서 로그인해도 React Remote의 헤더나 장바구니 버튼이 갱신되지 않아, 사용자가 페이지를 새로고침하기 전까지 인증 상태가 반영되지 않는 문제가 있었습니다.

## 해결: 2-Layer Bridge 아키텍처

### Layer 1: Framework-Agnostic Adapter

portal-shell에서 Pinia store를 순수 JavaScript 함수로 래핑합니다. Vue에 의존하지 않는 `getState()` + `subscribe()` 인터페이스를 제공하는 것이 핵심입니다.

```typescript
// portal-shell/src/store/storeAdapter.ts
export const authAdapter = {
  getState: (): AuthState => {
    const store = useAuthStore()
    return {
      isAuthenticated: store.isAuthenticated,
      displayName: store.displayName,
      isAdmin: store.isAdmin,
      isSeller: store.isSeller,
      roles: store.roles,
      user: store.user ? { uuid: store.user.uuid, email: store.user.email, ... } : null,
    }
  },

  subscribe: (callback: (state: AuthState) => void): UnsubscribeFn => {
    const store = useAuthStore()
    return watch(
      () => store.user,
      () => callback(authAdapter.getState()),
      { deep: true }
    )
  },

  hasRole: (role: string) => useAuthStore().hasRole(role),
  logout: () => useAuthStore().logout(),
  getAccessToken: () => useAuthStore().user?._accessToken ?? null,
  requestLogin: (path?: string) => useAuthStore().requestLogin(path),
}
```

이 adapter를 Module Federation으로 expose합니다.

```typescript
// vite.config.ts
federation({
  exposes: {
    './stores': './src/store/index.ts',  // authAdapter, themeAdapter
    './api': './src/api/index.ts',       // apiClient
  },
})
```

### Layer 2: Framework-Specific Wrapper

각 프레임워크에 맞는 래퍼 라이브러리가 adapter를 소비합니다.

#### React — `@portal/react-bridge`

React의 `useSyncExternalStore`로 adapter를 연결합니다. 이 API는 외부 store를 React의 렌더링 사이클에 통합하는 공식 방법입니다.

```typescript
// react-bridge/src/hooks/usePortalAuth.ts
export function usePortalAuth() {
  const isReady = isBridgeReady()

  const getSnapshot = useMemo(() => {
    if (!isReady) return () => defaultAuthState
    const adapter = getAdapter('auth')
    let cached: AuthState | undefined
    return () => {
      const next = adapter.getState()
      // 얕은 비교로 동일 상태면 이전 참조 반환 (불필요한 리렌더 방지)
      if (cached && cached.isAuthenticated === next.isAuthenticated
          && cached.user?.uuid === next.user?.uuid) {
        return cached
      }
      cached = next
      return next
    }
  }, [isReady])

  const subscribe = useMemo(
    () => isReady ? getAdapter('auth').subscribe : noopSubscribe,
    [isReady]
  )

  const state = useSyncExternalStore(subscribe, getSnapshot, () => defaultAuthState)

  // actions도 함께 반환
  return { ...state, hasRole, logout, getAccessToken, requestLogin }
}
```

`getSnapshot`에서 참조 안정성(referential stability)을 유지하는 것이 중요합니다. `useSyncExternalStore`는 `Object.is`로 이전 snapshot과 비교하여 리렌더 여부를 결정하므로, 매번 새 객체를 반환하면 무한 리렌더가 발생합니다.

#### Vue — `@portal/vue-bridge`

Vue Remote에서는 module-level singleton ref를 사용합니다.

```typescript
// vue-bridge/src/composables/usePortalAuth.ts
import { ref, computed } from 'vue'
import { authAdapter } from 'portal/stores'

// 모든 컴포넌트가 같은 reactive state를 공유
const _state = ref<AuthState>(authAdapter.getState())

authAdapter.subscribe((newState) => {
  _state.value = newState
})

export function usePortalAuth() {
  return {
    isAuthenticated: computed(() => _state.value.isAuthenticated),
    displayName: computed(() => _state.value.displayName),
    isAdmin: computed(() => _state.value.isAdmin),
    roles: computed(() => _state.value.roles),
    hasRole: (role: string) => authAdapter.hasRole(role),
    logout: () => authAdapter.logout(),
  }
}
```

Vue에서는 `ref` 자체가 reactive proxy이므로, `useSyncExternalStore` 같은 별도 메커니즘이 필요 없습니다. `subscribe` 콜백이 `ref.value`를 갱신하면 `computed`가 자동으로 재계산됩니다.

React에서는 Context가 아닌 module scope에 상태를 두는 이유가 궁금할 수 있습니다. MF 환경에서 React Context Provider는 Host-Remote 경계를 넘지 못합니다. Host와 Remote가 별도 React 트리로 마운트되기 때문입니다. 따라서 module singleton이 경계를 넘는 상태 공유에 더 적합합니다. 또한 이 프로젝트는 CSR 전용이므로, SSR에서 발생할 수 있는 module singleton 상태 누수 문제(요청 간 상태 오염)를 고려할 필요가 없습니다.

### Bridge 초기화: Race Condition 해결

React Remote에서는 Module Federation의 비동기 로딩으로 인해 bridge 초기화 시점이 불확실합니다. `PortalBridgeProvider`가 초기화 완료까지 children 렌더링을 차단합니다.

```tsx
export function PortalBridgeProvider({ children, fallback }: Props) {
  const [ready, setReady] = useState(isBridgeReady())

  useEffect(() => {
    if (!window.__POWERED_BY_PORTAL_SHELL__) {
      setReady(true) // Standalone 모드
      return
    }
    Promise.all([initBridge(), initPortalApi()])
      .then(() => setReady(true))
      .catch(setError)
  }, [])

  if (!ready) return <>{fallback ?? null}</>
  return <>{children}</>
}
```

### Standalone Fallback

Remote 앱이 Host 없이 독립 실행될 때(개발 중)는 bridge가 초기화되지 않습니다. 이 경우 guest 상태를 반환하여 앱이 정상 동작하도록 합니다.

```typescript
// React: isBridgeReady() === false → defaultAuthState 반환
// Vue: import('portal/stores') 실패 → graceful degradation
```

## 실제 사용

React든 Vue든 같은 이름의 hook/composable(`usePortalAuth`)을 호출하면, 동일한 Host의 인증 상태를 반응적으로 받아옵니다.

## 아키텍처 요약

```
Pinia Store (portal-shell)
    ↓ watch()
Framework-Agnostic Adapter (getState + subscribe)
    ↓ Module Federation (portal/stores)
    ├── React: useSyncExternalStore → usePortalAuth()
    └── Vue: ref + computed → usePortalAuth()
```

## 교훈

**1. 프레임워크 경계를 넘는 상태 공유는 프레임워크 비의존 계층이 필수다**

Pinia를 직접 expose하면 React에서 사용할 수 없습니다. `getState()` + `subscribe()`라는 최소한의 인터페이스로 추상화하면, 어떤 프레임워크든 자신의 반응성 시스템에 연결할 수 있습니다.

**2. React에서 외부 store 연결은 `useSyncExternalStore`가 권장되는 방법이다**

`useEffect` + `setState` 조합으로 외부 store를 구독하면 tearing(불일치) 문제가 생깁니다. `useSyncExternalStore`는 React 18에서 이 문제를 해결하기 위해 도입된 API입니다. 특히 Concurrent Mode에서는 렌더 도중 외부 store 값이 변경될 수 있어, `useSyncExternalStore` 없이는 동일 렌더 패스 내에서 서로 다른 값을 읽는 tearing이 발생할 수 있습니다.

**3. 참조 안정성을 소홀히 하면 성능이 무너진다**

`getSnapshot`이 매번 새 객체를 반환하면 매 렌더마다 리렌더가 발생합니다. 얕은 비교로 동일 상태에 대해 같은 참조를 반환해야 합니다.
