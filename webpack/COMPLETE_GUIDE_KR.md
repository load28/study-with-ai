# 웹팩과 Module Federation 완벽 가이드 (한글)

## 📚 전체 문서 구성

이 가이드는 **코드 기반**으로 웹팩의 아키텍처를 분석하고, Module Federation을 Next.js App Router에서 구현하는 방법을 다룹니다.

### 문서 읽기 순서

1. **[웹팩 핵심 아키텍처 분석](WEBPACK_ARCHITECTURE_ANALYSIS.md)** ⭐ 시작점
   - Webpack의 내부 작동 원리
   - Compiler, Compilation, Module 시스템
   - 플러그인 아키텍처 (Tapable)
   - 전체 빌드 플로우

2. **[Module Federation 작동 원리](MODULE_FEDERATION_DEEP_DIVE.md)**
   - ModuleFederationPlugin 내부 구조
   - Container, Remote, Share 메커니즘
   - 런타임 코드 생성 과정
   - Share Scope 버전 관리

3. **[Next.js + Module Federation 구현](NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md)**
   - Remote 앱 구현 (React + Webpack)
   - Host 앱 구현 (Next.js App Router)
   - 수동 스크립트 로딩 전략
   - 실전 예제 및 트러블슈팅

---

## 🎯 학습 목표

이 가이드를 완독하면 다음을 할 수 있습니다:

- ✅ 웹팩의 내부 작동 원리를 **코드 레벨**에서 이해
- ✅ Module Federation의 3가지 핵심 플러그인 분석
- ✅ 런타임에 생성되는 코드 해석
- ✅ Next.js App Router에서 Module Federation 수동 구현
- ✅ MFA(Micro Frontend Architecture) 실전 적용

---

## 📖 핵심 개념 요약

### 1. 웹팩 아키텍처 핵심

```
webpack(config)
    ↓
Compiler (빌드 생명주기 관리)
    ↓
Compilation (개별 빌드 실행)
    ├─ NormalModuleFactory (모듈 생성)
    ├─ ModuleGraph (의존성 그래프)
    ├─ ChunkGraph (청크 그래프)
    └─ CodeGeneration (코드 생성)
    ↓
Output (파일 생성)
```

**핵심 클래스**:
- **Compiler**: 전체 빌드 프로세스의 싱글톤 관리자
- **Compilation**: 개별 빌드 실행 (watch 시 여러 번 생성)
- **NormalModuleFactory**: 파일 모듈 생성 팩토리
- **ModuleGraph**: 모듈 간 의존성 추적
- **Tapable**: 플러그인 시스템의 기반 (Hook 기반)

### 2. Module Federation 핵심

```javascript
// ModuleFederationPlugin = 3개 플러그인의 조합
new ModuleFederationPlugin({
    name: 'app',
    exposes: {...},   // → ContainerPlugin
    remotes: {...},   // → ContainerReferencePlugin
    shared: {...}     // → SharePlugin
})
```

**3가지 핵심 플러그인**:

| 플러그인 | 역할 | 생성 코드 |
|---------|------|-----------|
| **ContainerPlugin** | 모듈 노출 (expose) | `get()`, `init()` 함수 |
| **ContainerReferencePlugin** | 외부 모듈 소비 | RemoteModule, 동적 로딩 코드 |
| **SharePlugin** | 의존성 공유 | Share Scope 관리 코드 |

**런타임 흐름**:
```
1. Host 앱 시작
   ↓
2. Share Scope 초기화 (자신의 shared 모듈 등록)
   ↓
3. Remote 스크립트 로드 (remoteEntry.js)
   ↓
4. Remote.init(shareScope) → Share Scope 병합
   ↓
5. Remote.get(moduleName) → 모듈 팩토리 반환
   ↓
6. factory() 실행 → 실제 모듈 (React 컴포넌트 등)
```

### 3. Next.js App Router 제약사항

**문제**:
- Next.js는 SSR 지원
- Module Federation은 브라우저 전용
- App Router는 Server Components 기본

**해결**:
```typescript
// ❌ 작동 안 함
import RemoteComponent from 'remoteApp/Component'

// ✅ 수동 로딩 필요
'use client';
const Component = await loadRemoteModule({
    url: 'http://localhost:3001/remoteEntry.js',
    scope: 'remoteApp',
    module: './Component'
});
```

---

## 🔧 실전 구현 단계

### Step 1: Remote 앱 구성 (React + Webpack)

```javascript
// webpack.config.js
new ModuleFederationPlugin({
    name: 'remoteApp',
    filename: 'remoteEntry.js',
    exposes: {
        './Button': './src/components/Button',
    },
    shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
    }
})
```

**생성 결과**: `http://localhost:3001/remoteEntry.js`
```javascript
window.remoteApp = {
    get: (module) => Promise<factory>,
    init: (shareScope) => Promise<void>
}
```

### Step 2: Host 앱 구성 (Next.js)

```typescript
// lib/federationLoader.ts
export async function loadRemoteModule(config) {
    // 1. 스크립트 로드
    await loadScript(config.url);

    // 2. 컨테이너 가져오기
    const container = window[config.scope];

    // 3. Share Scope 초기화
    await container.init(createShareScope());

    // 4. 모듈 가져오기
    const factory = await container.get(config.module);
    return factory();
}
```

```typescript
// app/page.tsx
'use client';

export default function Page() {
    return (
        <>
            <Script src="http://localhost:3001/remoteEntry.js" />
            <RemoteWrapper
                scope="remoteApp"
                module="./Button"
            />
        </>
    );
}
```

### Step 3: Share Scope 수동 관리

```typescript
function createShareScope() {
    return {
        default: {
            react: {
                '18.2.0': {
                    get: () => Promise.resolve(() => require('react')),
                    loaded: true
                }
            }
        }
    };
}
```

---

## 🎨 아키텍처 다이어그램

### 전체 시스템 구조

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────┐           ┌──────────────────┐       │
│  │   Host App       │           │   Remote App     │       │
│  │   (Next.js)      │           │   (React)        │       │
│  │                  │           │                  │       │
│  │  ┌────────────┐  │           │  ┌────────────┐  │       │
│  │  │ Federation │◀─┼───HTTP────┼─▶│ Container  │  │       │
│  │  │   Loader   │  │           │  │   Entry    │  │       │
│  │  └────────────┘  │           │  └────────────┘  │       │
│  │        ↓         │           │                  │       │
│  │  ┌────────────┐  │           │  remoteEntry.js  │       │
│  │  │   Share    │◀─┼───────────┼─────────────────▶│       │
│  │  │   Scope    │  │  Merge    │                  │       │
│  │  └────────────┘  │           │                  │       │
│  │                  │           │                  │       │
│  │  react: 18.2.0   │           │  react: 18.2.0   │       │
│  │  react-dom       │           │  date-fns        │       │
│  └──────────────────┘           └──────────────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Module Federation 런타임 흐름

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 초기화 단계                                               │
└─────────────────────────────────────────────────────────────┘
                          ↓
    Host App 시작 → Share Scope 생성
         __webpack_require__.S = { default: {} }
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Remote 로딩 단계                                          │
└─────────────────────────────────────────────────────────────┘
                          ↓
    <Script src="http://remote.com/remoteEntry.js" />
                          ↓
    window.remoteApp = { get, init }
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Share Scope 초기화                                        │
└─────────────────────────────────────────────────────────────┘
                          ↓
    await remoteApp.init(__webpack_require__.S.default)
                          ↓
    Share Scope 병합:
    {
      react: {
        '18.2.0': { get: ..., loaded: true, from: 'host' },
        '18.1.0': { get: ..., loaded: true, from: 'remote' }
      }
    }
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 모듈 가져오기                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
    const factory = await remoteApp.get('./Button')
                          ↓
    모듈 팩토리 반환:
    () => {
      const React = __webpack_require__('react'); // Share Scope에서
      return ButtonComponent;
    }
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. 컴포넌트 실행                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
    const Button = factory()
                          ↓
    <Button label="Click" />
```

---

## 🚀 실행 가이드

### 1. Remote 앱 실행

```bash
cd remote-app
npm install
npm start
```

**확인**:
- http://localhost:3001 → Remote 앱 UI
- http://localhost:3001/remoteEntry.js → Container Entry

### 2. Host 앱 실행

```bash
cd host-app
npm install
npm run dev
```

**확인**:
- http://localhost:3000/remote-components
- 개발자 도구 → Network 탭에서 remoteEntry.js 로딩 확인
- Console에서 `window.remoteApp` 확인

### 3. 디버깅

```javascript
// 브라우저 콘솔에서
console.log(window.remoteApp);
// { get: ƒ, init: ƒ }

console.log(__webpack_require__.S);
// { default: { react: {...}, react-dom: {...} } }

// 모듈 수동 로드 테스트
await window.remoteApp.init(__webpack_require__.S.default);
const factory = await window.remoteApp.get('./Button');
const Button = factory();
console.log(Button);
```

---

## 🔍 심화 학습

### 1. 웹팩 플러그인 작성하기

```javascript
class MyModuleFederationPlugin {
    apply(compiler) {
        // Compiler 훅
        compiler.hooks.compilation.tap('MyPlugin', (compilation) => {
            // Compilation 훅
            compilation.hooks.buildModule.tap('MyPlugin', (module) => {
                console.log('Building:', module.identifier());
            });

            // NormalModuleFactory 훅 (핵심!)
            const nmf = compilation.moduleGraph._normalModuleFactory;
            nmf.hooks.factorize.tap('MyPlugin', (resolveData) => {
                // 여기서 RemoteModule로 교체 가능!
                if (resolveData.request.startsWith('remote/')) {
                    return new RemoteModule(...);
                }
            });
        });
    }
}
```

### 2. 커스텀 Share Scope 구현

```typescript
class CustomShareScopeManager {
    private static scopes = new Map<string, any>();

    static registerScope(name: string, modules: Record<string, any>) {
        const scope = this.scopes.get(name) || {};

        for (const [moduleName, version] of Object.entries(modules)) {
            if (!scope[moduleName]) {
                scope[moduleName] = {};
            }

            scope[moduleName][version] = {
                get: () => Promise.resolve(() => require(moduleName)),
                loaded: true,
                version
            };
        }

        this.scopes.set(name, scope);
        return scope;
    }

    static resolveModule(
        scopeName: string,
        moduleName: string,
        requiredVersion: string
    ) {
        const scope = this.scopes.get(scopeName);
        if (!scope || !scope[moduleName]) {
            throw new Error(`Module ${moduleName} not found in scope ${scopeName}`);
        }

        const versions = Object.keys(scope[moduleName]);
        const compatibleVersion = versions.find(v =>
            this.satisfiesVersion(v, requiredVersion)
        );

        if (!compatibleVersion) {
            throw new Error(
                `No compatible version for ${moduleName}@${requiredVersion}`
            );
        }

        return scope[moduleName][compatibleVersion].get();
    }

    private static satisfiesVersion(version: string, required: string): boolean {
        // 간단한 버전 비교 (실제로는 semver 라이브러리 사용)
        return version === required || required.startsWith('^');
    }
}
```

### 3. 동적 Remote 구성

```typescript
interface DynamicRemoteConfig {
    name: string;
    url: string;
    scope: string;
    modules: string[];
}

class DynamicRemoteRegistry {
    private static remotes = new Map<string, DynamicRemoteConfig>();

    static async register(config: DynamicRemoteConfig) {
        // 1. 런타임에 Remote 등록
        await loadScript(config.url);

        // 2. 컨테이너 확인
        const container = (window as any)[config.scope];
        if (!container) {
            throw new Error(`Container ${config.scope} not found`);
        }

        // 3. 레지스트리에 저장
        this.remotes.set(config.name, config);

        console.log(`✅ Registered remote: ${config.name}`);
    }

    static async load(remoteName: string, moduleName: string) {
        const config = this.remotes.get(remoteName);
        if (!config) {
            throw new Error(`Remote ${remoteName} not registered`);
        }

        return loadRemoteModule({
            url: config.url,
            scope: config.scope,
            module: moduleName
        });
    }

    static list() {
        return Array.from(this.remotes.entries()).map(([name, config]) => ({
            name,
            url: config.url,
            modules: config.modules
        }));
    }
}

// 사용 예시
await DynamicRemoteRegistry.register({
    name: 'dashboard',
    url: 'https://dashboard.example.com/remoteEntry.js',
    scope: 'dashboard',
    modules: ['./Chart', './Table']
});

const Chart = await DynamicRemoteRegistry.load('dashboard', './Chart');
```

---

## 📊 성능 최적화

### 1. Preloading 전략

```typescript
'use client';

import Script from 'next/script';
import { FederationLoader } from '@/lib/federationLoader';

export function RemotePreloader({ remotes }: { remotes: string[] }) {
    React.useEffect(() => {
        // Idle 시간에 미리 로드
        if ('requestIdleCallback' in window) {
            requestIdleCallback(() => {
                remotes.forEach(url => {
                    FederationLoader.preload(url);
                });
            });
        }
    }, [remotes]);

    return (
        <>
            {remotes.map(url => (
                <link key={url} rel="preload" as="script" href={url} />
            ))}
        </>
    );
}
```

### 2. 캐싱 전략

```typescript
class ModuleCache {
    private static cache = new Map<string, Promise<any>>();
    private static TTL = 5 * 60 * 1000; // 5분

    static async get<T>(key: string, loader: () => Promise<T>): Promise<T> {
        const cached = this.cache.get(key);
        if (cached) {
            return cached;
        }

        const promise = loader();
        this.cache.set(key, promise);

        // TTL 후 자동 삭제
        setTimeout(() => {
            this.cache.delete(key);
        }, this.TTL);

        return promise;
    }

    static clear() {
        this.cache.clear();
    }
}

// 사용
const Button = await ModuleCache.get(
    'remoteApp/Button',
    () => loadRemoteModule({ ... })
);
```

### 3. 에러 처리 및 Fallback

```typescript
function RemoteComponentWithFallback({
    url,
    scope,
    module,
    fallback: FallbackComponent
}) {
    const [Component, setComponent] = React.useState(null);
    const [error, setError] = React.useState(null);

    React.useEffect(() => {
        loadRemoteModule({ url, scope, module })
            .then(setComponent)
            .catch(err => {
                console.error('Failed to load remote:', err);
                setError(err);
                // 에러 로깅 서비스로 전송
                logErrorToService(err);
            });
    }, [url, scope, module]);

    if (error && FallbackComponent) {
        return <FallbackComponent error={error} />;
    }

    if (!Component) {
        return <Skeleton />;
    }

    return <Component />;
}
```

---

## 🐛 일반적인 문제 해결

### 문제 1: "Shared module is not available for eager consumption"

**원인**: Bootstrap 패턴 미사용

**해결**:
```javascript
// ❌ 잘못된 방법
import React from 'react';
import App from './App';

// ✅ 올바른 방법
// index.js
import('./bootstrap');

// bootstrap.js
import React from 'react';
import App from './App';
```

### 문제 2: React Hook 에러

**원인**: 여러 React 인스턴스 존재

**해결**:
```javascript
// 모든 앱의 webpack.config.js
shared: {
    react: {
        singleton: true,  // ✅ 필수!
        strictVersion: true
    }
}
```

### 문제 3: publicPath 관련 청크 로딩 실패

**원인**: 상대 경로 사용

**해결**:
```javascript
// ❌ 잘못된 방법
output: {
    publicPath: '/dist/'  // 상대 경로
}

// ✅ 올바른 방법
output: {
    publicPath: 'http://localhost:3001/'  // 절대 경로
}
```

---

## 📚 추가 학습 자료

### 웹팩 공식 문서
- [Webpack Concepts](https://webpack.js.org/concepts/)
- [Module Federation](https://webpack.js.org/concepts/module-federation/)
- [Plugin API](https://webpack.js.org/api/plugins/)

### 실제 웹팩 코드 분석
- [lib/Compiler.js](lib/Compiler.js) - 빌드 생명주기
- [lib/Compilation.js](lib/Compilation.js) - 실제 빌드 로직
- [lib/container/ModuleFederationPlugin.js](lib/container/ModuleFederationPlugin.js) - MF 플러그인

### 고급 주제
- Webpack 5의 새로운 기능
- Federated Types (타입 공유)
- Version Mismatch 처리
- 프로덕션 최적화

---

## 🎓 학습 체크리스트

- [ ] 웹팩의 Compiler와 Compilation 차이 이해
- [ ] Tapable Hook 시스템 이해
- [ ] ModuleGraph와 ChunkGraph 차이 이해
- [ ] Module Federation의 3가지 플러그인 역할 파악
- [ ] RemoteModule 생성 과정 이해
- [ ] Share Scope 작동 원리 이해
- [ ] Next.js에서 수동 로딩 구현 가능
- [ ] 에러 처리 및 Fallback 전략 수립
- [ ] 프로덕션 배포 고려사항 이해

---

## 📝 마무리

이 가이드를 통해 웹팩의 내부 작동 원리부터 Module Federation을 활용한 Micro Frontend Architecture 구현까지 모두 다뤘습니다.

**핵심 인사이트**:
1. **웹팩은 플러그인의 집합**: Compiler, Compilation, NormalModuleFactory 모두 Tapable Hook 제공
2. **Module Federation은 웹팩 시스템의 확장**: NormalModuleFactory.factorize 훅을 활용한 영리한 설계
3. **Next.js 제약은 수동 구현으로 해결**: 브라우저 환경에서 스크립트 로딩 + Share Scope 수동 관리

**다음 단계**:
- 실제 프로젝트에 적용해보기
- 성능 측정 및 최적화
- 더 복잡한 시나리오 도전 (다중 Remote, 동적 구성 등)

Happy Coding! 🚀
