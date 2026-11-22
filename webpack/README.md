# 웹팩 & Module Federation 학습 로드맵

> **코드 기반 웹팩 아키텍처 분석 + Next.js App Router Module Federation 실전 구현**
>
> 이 README는 기존 `MODULE_FEDERATION_GUIDE_README.md`(안내)와 `COMPLETE_GUIDE_KR.md`(통합 요약)를 하나로 합친 문서입니다. 문서 속 `lib/...` 경로는 [webpack 저장소](https://github.com/webpack/webpack)의 소스 파일을 가리킵니다.

## 📖 개요

이 가이드는 Webpack 5의 내부 아키텍처를 **실제 소스 코드**를 기반으로 분석하고, Module Federation을 활용하여 Next.js App Router 환경에서 Micro Frontend Architecture(MFA)를 구현하는 방법을 다룹니다.

## 🎯 이 가이드가 필요한 사람

- ✅ 웹팩의 내부 작동 원리를 깊이 이해하고 싶은 개발자
- ✅ Module Federation을 실전에 적용하려는 개발자
- ✅ Next.js App Router에서 MFA를 구현하려는 개발자
- ✅ 플러그인 작성 등 웹팩을 확장하고 싶은 개발자
- ✅ 코드 레벨에서 번들러의 작동 방식을 이해하고 싶은 개발자

## 🎯 학습 목표

이 가이드를 완독하면 다음을 할 수 있습니다:

- ✅ 웹팩의 내부 작동 원리를 **코드 레벨**에서 이해
- ✅ Module Federation의 3가지 핵심 플러그인 분석
- ✅ 런타임에 생성되는 코드 해석
- ✅ Next.js App Router에서 Module Federation 수동 구현
- ✅ MFA(Micro Frontend Architecture) 실전 적용

---

## 📚 문서 구성

### 1. [웹팩 핵심 아키텍처 분석](WEBPACK_ARCHITECTURE_ANALYSIS.md)
**읽는 데 걸리는 시간**: 약 30분

웹팩의 핵심 클래스와 빌드 플로우를 실제 코드 기반으로 분석합니다.

**다루는 내용**:
- 웹팩 빌드 플로우 (webpack() → Compiler → Compilation)
- Compiler와 Compilation의 차이
- NormalModuleFactory의 역할
- ModuleGraph와 ChunkGraph
- Tapable Hook 시스템
- 플러그인 작성 방법

**핵심 코드 위치**:
- [lib/webpack.js](https://github.com/webpack/webpack/blob/main/lib/webpack.js) - 엔트리 포인트
- [lib/Compiler.js](https://github.com/webpack/webpack/blob/main/lib/Compiler.js) - 빌드 생명주기 관리
- [lib/Compilation.js](https://github.com/webpack/webpack/blob/main/lib/Compilation.js) - 실제 빌드 실행
- [lib/NormalModuleFactory.js](https://github.com/webpack/webpack/blob/main/lib/NormalModuleFactory.js) - 모듈 생성

**학습 목표**:
```javascript
// 이 코드가 무슨 일을 하는지 정확히 이해하기
const compiler = webpack(config);
compiler.run((err, stats) => {
    // 내부에서 어떤 일이 일어나는가?
});
```

---

### 2. [Module Federation 작동 원리 심층 분석](MODULE_FEDERATION_DEEP_DIVE.md)
**읽는 데 걸리는 시간**: 약 40분

Module Federation의 내부 구조와 런타임 코드 생성 과정을 분석합니다.

**다루는 내용**:
- ModuleFederationPlugin의 3가지 핵심 플러그인
- ContainerPlugin (expose) 작동 원리
- ContainerReferencePlugin (remote) 작동 원리
- SharePlugin과 Share Scope 메커니즘
- 런타임 코드 생성 과정
- 버전 호환성 처리

**핵심 코드 위치**:
- [lib/container/ModuleFederationPlugin.js](https://github.com/webpack/webpack/blob/main/lib/container/ModuleFederationPlugin.js) - 메인 플러그인
- [lib/container/ContainerPlugin.js](https://github.com/webpack/webpack/blob/main/lib/container/ContainerPlugin.js) - Expose 기능
- [lib/container/ContainerReferencePlugin.js](https://github.com/webpack/webpack/blob/main/lib/container/ContainerReferencePlugin.js) - Remote 기능
- [lib/container/ContainerEntryModule.js](https://github.com/webpack/webpack/blob/main/lib/container/ContainerEntryModule.js) - Container Entry 코드 생성
- [lib/container/RemoteRuntimeModule.js](https://github.com/webpack/webpack/blob/main/lib/container/RemoteRuntimeModule.js) - Remote 로딩 런타임

**학습 목표**:
```javascript
// remoteEntry.js에 생성되는 코드 이해하기
window.remoteApp = {
    get: function(module) { /* 어떻게 작동하는가? */ },
    init: function(shareScope) { /* Share Scope란? */ }
};
```

---

### 3. [Next.js App Router + Module Federation 구현](NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md)
**읽는 데 걸리는 시간**: 약 50분

Next.js App Router 환경에서 Module Federation을 수동으로 구현하는 방법과 실전 예제를 다룹니다.

**다루는 내용**:
- Next.js App Router의 제약사항
- Remote 앱 구현 (React + Webpack)
- Host 앱 구현 (Next.js App Router)
- 수동 Federation Loader 구현
- Share Scope 수동 관리
- 에러 처리 및 Fallback 전략
- 트러블슈팅

**제공되는 코드**:
- Remote 앱 완전한 구현 예제
- Host 앱 완전한 구현 예제
- TypeScript 타입 정의
- 재사용 가능한 FederationLoader
- RemoteWrapper 컴포넌트
- 고급 에러 처리 로직

**학습 목표**:
```typescript
// 수동으로 Remote 모듈 로드하기
const Button = await loadRemoteModule({
    url: 'http://localhost:3001/remoteEntry.js',
    scope: 'remoteApp',
    module: './Button'
});
```

---

## 🚀 빠른 시작

### 문서 읽기 순서

#### 초보자 (웹팩 경험 부족)
```
1. README.md 「핵심 개념 요약」 (개요 파악)
   ↓
2. WEBPACK_ARCHITECTURE_ANALYSIS.md (기초 다지기)
   ↓
3. MODULE_FEDERATION_DEEP_DIVE.md (심화 학습)
   ↓
4. NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md (실전 적용)
```

#### 중급자 (웹팩 기본 이해)
```
1. MODULE_FEDERATION_DEEP_DIVE.md (Module Federation 집중)
   ↓
2. NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md (실전 적용)
   ↓
3. WEBPACK_ARCHITECTURE_ANALYSIS.md (필요시 참고)
```

#### 고급자 (바로 구현하고 싶은 경우)
```
1. NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md (구현 예제)
   ↓
2. MODULE_FEDERATION_DEEP_DIVE.md (원리 이해)
```

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

### 4. Next.js 수동 구현 흐름

```typescript
// 1. 스크립트 로드
<Script src="http://localhost:3001/remoteEntry.js" />

// 2. 컨테이너 접근
const container = window.remoteApp;

// 3. Share Scope 초기화
await container.init(shareScope);

// 4. 모듈 가져오기
const factory = await container.get('./Button');
const Button = factory();
```

---

## 📁 폴더 구성

```
webpack/
├── README.md                                   # 이 파일 (로드맵 + 통합 요약)
├── WEBPACK_ARCHITECTURE_ANALYSIS.md            # 웹팩 아키텍처 분석
├── MODULE_FEDERATION_DEEP_DIVE.md              # Module Federation 심층 분석
├── NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md  # Next.js 구현 가이드
└── TESTING_DOCS.md                             # 웹팩 테스트 스위트 구조
```

분석 대상 소스(webpack 저장소 `lib/`):

```
lib/
├── webpack.js
├── Compiler.js
├── Compilation.js
├── NormalModuleFactory.js
└── container/
    ├── ModuleFederationPlugin.js
    ├── ContainerPlugin.js
    ├── ContainerReferencePlugin.js
    ├── ContainerEntryModule.js
    └── RemoteRuntimeModule.js
examples/module-federation/webpack.config.js   # 공식 예제
```

---

## 🎓 학습 로드맵

### Phase 1: 기초 다지기 (1-2일)
- [ ] 웹팩 빌드 플로우 이해
- [ ] Compiler vs Compilation 차이 파악
- [ ] Tapable Hook 시스템 이해
- [ ] 간단한 플러그인 작성해보기

### Phase 2: Module Federation 이해 (2-3일)
- [ ] ModuleFederationPlugin 구조 파악
- [ ] Container Entry 코드 분석
- [ ] Remote 로딩 메커니즘 이해
- [ ] Share Scope 작동 원리 이해

### Phase 3: 실전 구현 (3-5일)
- [ ] Remote 앱 구현 (React + Webpack)
- [ ] Host 앱 구현 (Next.js)
- [ ] Federation Loader 구현
- [ ] 에러 처리 및 최적화

### Phase 4: 심화 학습 (계속)
- [ ] 커스텀 플러그인 작성
- [ ] 동적 Remote 구성
- [ ] 성능 최적화
- [ ] 프로덕션 배포

---

## 🔧 실습 환경 설정

### 필요 사항
- Node.js 18+
- npm 또는 yarn
- 코드 에디터 (VSCode 추천)

### Remote 앱 (React + Webpack)
```bash
mkdir remote-app
cd remote-app
npm init -y
npm install react react-dom
npm install -D webpack webpack-cli webpack-dev-server
npm install -D babel-loader @babel/core @babel/preset-react
npm install -D html-webpack-plugin
```

### Host 앱 (Next.js)
```bash
npx create-next-app@latest host-app --typescript --app
cd host-app
npm install
```

---

## 📊 문서별 난이도

| 문서 | 난이도 | 사전 지식 | 예상 시간 |
|------|--------|-----------|-----------|
| README.md (핵심 개념 요약) | ⭐ 쉬움 | 기본 JavaScript | 20분 |
| WEBPACK_ARCHITECTURE_ANALYSIS.md | ⭐⭐⭐ 중간 | Webpack 기본 | 30분 |
| MODULE_FEDERATION_DEEP_DIVE.md | ⭐⭐⭐⭐ 어려움 | Webpack 구조 이해 | 40분 |
| NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md | ⭐⭐⭐ 중간 | React, Next.js | 50분 |

---

## 💻 코드 예제

각 문서에는 실제 동작하는 코드 예제가 포함되어 있습니다:

### 웹팩 플러그인 작성
```javascript
class MyPlugin {
    apply(compiler) {
        compiler.hooks.compilation.tap('MyPlugin', (compilation) => {
            compilation.hooks.buildModule.tap('MyPlugin', (module) => {
                console.log('Building:', module.identifier());
            });
        });
    }
}
```

### Remote 컴포넌트 로딩
```typescript
const Button = await loadRemoteModule({
    url: 'http://localhost:3001/remoteEntry.js',
    scope: 'remoteApp',
    module: './Button'
});
```

### Next.js 통합
```typescript
'use client';

export default function Page() {
    return (
        <RemoteWrapper
            url="http://localhost:3001/remoteEntry.js"
            scope="remoteApp"
            module="./Button"
        />
    );
}
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

## 🐛 트러블슈팅 (문서별 링크)

자주 발생하는 문제와 해결책은 각 문서의 트러블슈팅 섹션을 참고하세요:

- CORS 에러 → [NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md#61-cors-에러](NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md)
- React Hook 에러 → [NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md#62-react-버전-불일치](NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md)
- Container not found → [NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md#63-container-not-found-에러](NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md)
- SSR 에러 → [NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md#64-ssr-에러](NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md)

---

## 📚 추가 학습 자료

### 공식 문서
- [Webpack 5 Documentation](https://webpack.js.org/)
- [Module Federation Documentation](https://webpack.js.org/concepts/module-federation/)
- [Next.js Documentation](https://nextjs.org/docs)

### 참고할 만한 저장소
- [Webpack Source Code](https://github.com/webpack/webpack)
- [Module Federation Examples](https://github.com/module-federation/module-federation-examples)

### 관련 기술
- Tapable (웹팩의 플러그인 시스템)
- enhanced-resolve (웹팩의 모듈 리졸버)
- webpack-sources (소스 코드 추상화)

### 웹팩 공식 문서 (개념·API)
- [Webpack Concepts](https://webpack.js.org/concepts/)
- [Module Federation](https://webpack.js.org/concepts/module-federation/)
- [Plugin API](https://webpack.js.org/api/plugins/)

### 실제 웹팩 코드 분석
- [lib/Compiler.js](https://github.com/webpack/webpack/blob/main/lib/Compiler.js) - 빌드 생명주기
- [lib/Compilation.js](https://github.com/webpack/webpack/blob/main/lib/Compilation.js) - 실제 빌드 로직
- [lib/container/ModuleFederationPlugin.js](https://github.com/webpack/webpack/blob/main/lib/container/ModuleFederationPlugin.js) - MF 플러그인

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

## 🎯 최종 목표

이 가이드를 완료하면:

✅ 웹팩의 내부 작동 원리를 코드 레벨에서 이해
✅ Module Federation의 3가지 핵심 플러그인 작동 방식 파악
✅ Next.js App Router에서 MFA 구현 가능
✅ 커스텀 플러그인 작성 능력 습득
✅ 프로덕션 환경에서의 최적화 및 문제 해결 능력 향상

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

---

## 🤝 기여 및 피드백

이 가이드에 대한 피드백이나 개선 사항이 있다면:
1. 이슈 생성
2. 풀 리퀘스트 제출
3. 이메일로 연락

---

## 📄 라이선스

이 가이드는 MIT 라이선스를 따르며, Webpack 프로젝트의 코드 분석을 포함합니다.

Webpack 프로젝트 라이선스: MIT License
- 원저작자: Tobias Koppers (@sokra)
- 저장소: https://github.com/webpack/webpack

---

## 🚀 시작하기

지금 바로 첫 번째 문서를 읽어보세요!

👉 [웹팩 핵심 아키텍처 분석 시작하기](WEBPACK_ARCHITECTURE_ANALYSIS.md)

또는

👉 위 「핵심 개념 요약」부터 빠르게 훑어보기

Happy Learning! 📖✨
