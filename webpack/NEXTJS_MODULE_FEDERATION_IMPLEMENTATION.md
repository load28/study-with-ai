# Next.js App Router + Module Federation 구현 가이드

## 목차
1. [문제 상황 및 해결 전략](#1-문제-상황-및-해결-전략)
2. [Remote 앱 구현 (React + Webpack)](#2-remote-앱-구현-react--webpack)
3. [Host 앱 구현 (Next.js App Router)](#3-host-앱-구현-nextjs-app-router)
4. [수동 Module Federation 로더](#4-수동-module-federation-로더)
5. [전체 통합 예제](#5-전체-통합-예제)
6. [트러블슈팅](#6-트러블슈팅)

---

## 1. 문제 상황 및 해결 전략

### 1.1 Next.js App Router의 제약사항

**문제점**:
```javascript
// ❌ Next.js App Router에서 작동하지 않음
// next.config.js
module.exports = {
    webpack: (config) => {
        config.plugins.push(
            new ModuleFederationPlugin({  // Error!
                remotes: {
                    remote: "remote@http://localhost:3001/remoteEntry.js"
                }
            })
        );
        return config;
    }
};
```

**이유**:
1. Next.js는 **서버 사이드 렌더링(SSR)**을 지원
2. Module Federation은 **브라우저 전용** 기술
3. Next.js 빌드 프로세스가 복잡함 (서버 + 클라이언트 별도 번들)
4. App Router는 Server Components 기본 사용

### 1.2 해결 전략

```
┌─────────────────────────────────────────────────────────────┐
│                      해결 전략                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Remote: 표준 Webpack + ModuleFederationPlugin         │
│     → remoteEntry.js 생성                                  │
│                                                             │
│  2. Host (Next.js): 수동 스크립트 로딩                      │
│     → <Script> 태그로 remoteEntry.js 로드                   │
│     → 전역 변수로 컨테이너 접근                             │
│     → init() + get() 수동 호출                             │
│                                                             │
│  3. Client Component 필수                                  │
│     → 'use client' 지시어 사용                              │
│     → Dynamic import로 lazy loading                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Remote 앱 구현 (React + Webpack)

### 2.1 프로젝트 구조

```
remote-app/
├── src/
│   ├── index.js
│   ├── bootstrap.js
│   ├── components/
│   │   ├── Button.tsx
│   │   └── Card.tsx
│   └── App.tsx
├── public/
│   └── index.html
├── webpack.config.js
├── package.json
└── tsconfig.json
```

### 2.2 package.json

```json
{
  "name": "remote-app",
  "version": "1.0.0",
  "scripts": {
    "start": "webpack serve --mode development",
    "build": "webpack --mode production"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@babel/core": "^7.23.0",
    "@babel/preset-react": "^7.22.0",
    "@babel/preset-typescript": "^7.23.0",
    "babel-loader": "^9.1.3",
    "html-webpack-plugin": "^5.5.3",
    "typescript": "^5.2.2",
    "webpack": "^5.88.0",
    "webpack-cli": "^5.1.4",
    "webpack-dev-server": "^4.15.1"
  }
}
```

### 2.3 webpack.config.js (핵심)

```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  entry: './src/index.js',
  mode: 'development',
  devServer: {
    port: 3001,
    hot: true,
    headers: {
      'Access-Control-Allow-Origin': '*',  // CORS 허용 (중요!)
    },
  },
  output: {
    publicPath: 'http://localhost:3001/',  // 절대 경로 필수
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
    clean: true,
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx|ts|tsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              '@babel/preset-react',
              '@babel/preset-typescript',
            ],
          },
        },
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'remoteApp',  // 컨테이너 이름
      filename: 'remoteEntry.js',  // 엔트리 파일명

      // 노출할 모듈
      exposes: {
        './Button': './src/components/Button',
        './Card': './src/components/Card',
      },

      // 공유 의존성
      shared: {
        react: {
          singleton: true,  // 단일 인스턴스만 사용
          requiredVersion: '^18.0.0',
          eager: false,  // lazy loading
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
          eager: false,
        },
      },
    }),
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx'],
  },
};
```

### 2.4 src/index.js (Bootstrap 패턴)

```javascript
// index.js - 비동기 초기화 (shared 모듈 로딩을 위해 필수)
import('./bootstrap');
```

```javascript
// bootstrap.js - 실제 앱 시작점
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**왜 bootstrap 패턴?**
- Module Federation의 shared 모듈은 **비동기로 로드**됨
- 동기적으로 `import React from 'react'`를 하면 에러 발생
- `import('./bootstrap')`로 비동기 경계 생성

### 2.5 노출할 컴포넌트

```typescript
// src/components/Button.tsx
import React from 'react';

export interface ButtonProps {
  label: string;
  onClick?: () => void;
  variant?: 'primary' | 'secondary';
}

const Button: React.FC<ButtonProps> = ({ label, onClick, variant = 'primary' }) => {
  return (
    <button
      onClick={onClick}
      style={{
        padding: '10px 20px',
        backgroundColor: variant === 'primary' ? '#0070f3' : '#666',
        color: 'white',
        border: 'none',
        borderRadius: '5px',
        cursor: 'pointer',
      }}
    >
      {label}
    </button>
  );
};

export default Button;
```

```typescript
// src/components/Card.tsx
import React from 'react';

export interface CardProps {
  title: string;
  content: string;
}

const Card: React.FC<CardProps> = ({ title, content }) => {
  return (
    <div
      style={{
        border: '1px solid #ddd',
        borderRadius: '8px',
        padding: '20px',
        margin: '10px',
        boxShadow: '0 2px 4px rgba(0,0,0,0.1)',
      }}
    >
      <h3>{title}</h3>
      <p>{content}</p>
    </div>
  );
};

export default Card;
```

### 2.6 생성되는 remoteEntry.js 분석

```javascript
// http://localhost:3001/remoteEntry.js (간소화)
var remoteApp;
(() => {
  var __webpack_modules__ = { /* ... */ };
  var __webpack_module_cache__ = {};

  function __webpack_require__(moduleId) {
    // 모듈 로딩 로직
  }

  // Module Federation 런타임
  __webpack_require__.f = {};
  __webpack_require__.e = function(chunkId) { /* chunk 로딩 */ };
  __webpack_require__.I = function(name) { /* share scope 초기화 */ };
  __webpack_require__.S = {}; // Share Scope 전역 객체

  // Container Entry Module
  var moduleMap = {
    "./Button": function() {
      return __webpack_require__.e("src_components_Button_tsx")
        .then(() => () => __webpack_require__("./src/components/Button.tsx"));
    },
    "./Card": function() {
      return __webpack_require__.e("src_components_Card_tsx")
        .then(() => () => __webpack_require__("./src/components/Card.tsx"));
    }
  };

  var get = function(module, getScope) {
    __webpack_require__.currentRemoteGetScope = getScope;
    var result = moduleMap[module]
      ? moduleMap[module]()
      : Promise.reject(new Error('Module "' + module + '" does not exist'));
    __webpack_require__.currentRemoteGetScope = undefined;
    return result;
  };

  var init = function(shareScope, initScope) {
    if (!__webpack_require__.S) __webpack_require__.S = {};
    var name = "default";
    var oldScope = __webpack_require__.S[name];
    if (oldScope && oldScope !== shareScope) {
      throw new Error("Container initialization failed");
    }
    __webpack_require__.S[name] = shareScope;
    return __webpack_require__.I(name, initScope);
  };

  // 전역 변수로 노출
  remoteApp = {
    get: get,
    init: init
  };
})();
```

---

## 3. Host 앱 구현 (Next.js App Router)

### 3.1 프로젝트 구조

```
host-app/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   └── remote-components/
│       └── page.tsx
├── lib/
│   ├── federationLoader.ts
│   └── types.d.ts
├── components/
│   └── RemoteWrapper.tsx
├── next.config.js
├── package.json
└── tsconfig.json
```

### 3.2 package.json

```json
{
  "name": "host-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "^14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/react": "^18.2.0",
    "typescript": "^5.2.2"
  }
}
```

### 3.3 lib/federationLoader.ts (핵심 로더)

```typescript
// lib/federationLoader.ts
export interface RemoteContainer {
  get: (module: string) => Promise<() => any>;
  init: (shareScope: any) => Promise<void>;
}

interface FederationConfig {
  url: string;
  scope: string;
  module: string;
}

/**
 * Share Scope 생성
 * Host의 shared 모듈을 등록
 */
function createShareScope(): any {
  // Next.js 환경에서는 이미 로드된 모듈 사용
  const shareScope: any = {
    react: {
      '18.2.0': {
        get: () => Promise.resolve(() => require('react')),
        loaded: true,
        from: 'host'
      }
    },
    'react-dom': {
      '18.2.0': {
        get: () => Promise.resolve(() => require('react-dom')),
        loaded: true,
        from: 'host'
      }
    }
  };

  return { default: shareScope };
}

/**
 * 스크립트 동적 로드
 */
function loadScript(url: string): Promise<void> {
  return new Promise((resolve, reject) => {
    // 이미 로드된 경우
    const existingScript = document.querySelector(`script[src="${url}"]`);
    if (existingScript) {
      resolve();
      return;
    }

    const script = document.createElement('script');
    script.src = url;
    script.type = 'text/javascript';
    script.async = true;

    script.onload = () => {
      console.log(`✅ Script loaded: ${url}`);
      resolve();
    };

    script.onerror = () => {
      console.error(`❌ Failed to load script: ${url}`);
      reject(new Error(`Failed to load script: ${url}`));
    };

    document.head.appendChild(script);
  });
}

/**
 * Remote 컨테이너 가져오기
 */
async function getContainer(scope: string): Promise<RemoteContainer> {
  // @ts-ignore - 전역 변수로 노출된 컨테이너
  const container = window[scope];

  if (!container) {
    throw new Error(
      `Container "${scope}" not found. Make sure the remote entry script is loaded.`
    );
  }

  return container;
}

/**
 * Remote 모듈 로드
 */
export async function loadRemoteModule<T = any>(
  config: FederationConfig
): Promise<T> {
  const { url, scope, module } = config;

  // 1. 스크립트 로드
  await loadScript(url);

  // 2. 컨테이너 가져오기
  const container = await getContainer(scope);

  // 3. Share Scope 초기화
  const shareScope = createShareScope();
  await container.init(shareScope);

  // 4. 모듈 가져오기
  const factory = await container.get(module);
  const Module = factory();

  return Module;
}

/**
 * 캐싱을 지원하는 로더
 */
const moduleCache = new Map<string, Promise<any>>();

export function loadRemoteModuleCached<T = any>(
  config: FederationConfig
): Promise<T> {
  const cacheKey = `${config.scope}:${config.module}`;

  if (!moduleCache.has(cacheKey)) {
    moduleCache.set(cacheKey, loadRemoteModule<T>(config));
  }

  return moduleCache.get(cacheKey)!;
}

/**
 * React Suspense와 함께 사용
 */
export function createRemoteComponent<T = any>(config: FederationConfig) {
  let modulePromise: Promise<T> | null = null;
  let module: T | null = null;
  let error: Error | null = null;

  return {
    read(): T {
      if (error) throw error;
      if (module) return module;

      if (!modulePromise) {
        modulePromise = loadRemoteModuleCached<T>(config)
          .then((m) => {
            module = m;
            return m;
          })
          .catch((err) => {
            error = err;
            throw err;
          });
      }

      throw modulePromise; // Suspense가 catch
    }
  };
}
```

### 3.4 lib/types.d.ts

```typescript
// lib/types.d.ts
declare module 'remoteApp/Button' {
  export interface ButtonProps {
    label: string;
    onClick?: () => void;
    variant?: 'primary' | 'secondary';
  }

  const Button: React.FC<ButtonProps>;
  export default Button;
}

declare module 'remoteApp/Card' {
  export interface CardProps {
    title: string;
    content: string;
  }

  const Card: React.FC<CardProps>;
  export default Card;
}

// 전역 타입
declare global {
  interface Window {
    remoteApp?: {
      get: (module: string) => Promise<() => any>;
      init: (shareScope: any) => Promise<void>;
    };
  }
}

export {};
```

### 3.5 components/RemoteWrapper.tsx

```typescript
'use client';

import React, { Suspense, lazy } from 'react';
import { loadRemoteModule } from '@/lib/federationLoader';

interface RemoteWrapperProps {
  url: string;
  scope: string;
  module: string;
  fallback?: React.ReactNode;
  errorFallback?: React.ReactNode;
  children?: (Component: React.ComponentType<any>) => React.ReactNode;
}

/**
 * Remote 컴포넌트를 안전하게 로드하는 래퍼
 */
export function RemoteWrapper({
  url,
  scope,
  module,
  fallback = <div>Loading...</div>,
  errorFallback = <div>Failed to load component</div>,
  children
}: RemoteWrapperProps) {
  const [Component, setComponent] = React.useState<React.ComponentType | null>(null);
  const [error, setError] = React.useState<Error | null>(null);
  const [loading, setLoading] = React.useState(true);

  React.useEffect(() => {
    loadRemoteModule({ url, scope, module })
      .then((mod) => {
        setComponent(() => mod.default || mod);
        setLoading(false);
      })
      .catch((err) => {
        console.error('Failed to load remote module:', err);
        setError(err);
        setLoading(false);
      });
  }, [url, scope, module]);

  if (loading) return <>{fallback}</>;
  if (error) return <>{errorFallback}</>;
  if (!Component) return <>{errorFallback}</>;

  if (children) {
    return <>{children(Component)}</>;
  }

  return <Component />;
}

/**
 * 타입 안전한 Remote 컴포넌트 로더
 */
export function createRemoteComponent<P = any>(
  url: string,
  scope: string,
  module: string
) {
  return function RemoteComponent(props: P) {
    return (
      <RemoteWrapper url={url} scope={scope} module={module}>
        {(Component) => <Component {...props} />}
      </RemoteWrapper>
    );
  };
}
```

### 3.6 app/remote-components/page.tsx

```typescript
'use client';

import React from 'react';
import Script from 'next/script';
import { RemoteWrapper, createRemoteComponent } from '@/components/RemoteWrapper';
import type { ButtonProps } from 'remoteApp/Button';
import type { CardProps } from 'remoteApp/Card';

const REMOTE_URL = 'http://localhost:3001/remoteEntry.js';

// 방법 1: RemoteWrapper 직접 사용
function DirectUsage() {
  return (
    <div>
      <h2>방법 1: RemoteWrapper 직접 사용</h2>
      <RemoteWrapper
        url={REMOTE_URL}
        scope="remoteApp"
        module="./Button"
        fallback={<div>Loading Button...</div>}
      >
        {(Button) => (
          <Button
            label="Click Me"
            variant="primary"
            onClick={() => alert('Button clicked!')}
          />
        )}
      </RemoteWrapper>
    </div>
  );
}

// 방법 2: 미리 정의된 컴포넌트
const RemoteButton = createRemoteComponent<ButtonProps>(
  REMOTE_URL,
  'remoteApp',
  './Button'
);

const RemoteCard = createRemoteComponent<CardProps>(
  REMOTE_URL,
  'remoteApp',
  './Card'
);

function PredefinedComponents() {
  return (
    <div>
      <h2>방법 2: 미리 정의된 컴포넌트</h2>
      <RemoteButton
        label="Predefined Button"
        variant="secondary"
        onClick={() => console.log('Clicked!')}
      />
      <RemoteCard
        title="Remote Card"
        content="This is a card loaded from remote app"
      />
    </div>
  );
}

// 메인 페이지
export default function RemoteComponentsPage() {
  const [scriptLoaded, setScriptLoaded] = React.useState(false);

  return (
    <div style={{ padding: '20px' }}>
      {/* 1. Remote Entry 스크립트 로드 */}
      <Script
        src={REMOTE_URL}
        onLoad={() => {
          console.log('✅ Remote entry loaded');
          setScriptLoaded(true);
        }}
        onError={(e) => {
          console.error('❌ Failed to load remote entry:', e);
        }}
      />

      <h1>Next.js App Router + Module Federation</h1>

      {!scriptLoaded && <div>Loading remote entry...</div>}

      {scriptLoaded && (
        <>
          <DirectUsage />
          <hr />
          <PredefinedComponents />
        </>
      )}
    </div>
  );
}
```

### 3.7 app/layout.tsx

```typescript
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Host App - Module Federation',
  description: 'Next.js App Router with Module Federation',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

---

## 4. 수동 Module Federation 로더

### 4.1 고급 로더 (에러 처리 + 재시도)

```typescript
// lib/advancedFederationLoader.ts
export interface LoaderConfig {
  url: string;
  scope: string;
  module: string;
  retries?: number;
  timeout?: number;
  onProgress?: (stage: string) => void;
}

export class FederationLoader {
  private static containers = new Map<string, any>();
  private static shareScope: any = null;

  /**
   * Share Scope 초기화 (한 번만 실행)
   */
  private static initShareScope() {
    if (this.shareScope) return this.shareScope;

    this.shareScope = {
      default: {
        react: {
          '18.2.0': {
            get: () => Promise.resolve(() => require('react')),
            loaded: true,
          }
        },
        'react-dom': {
          '18.2.0': {
            get: () => Promise.resolve(() => require('react-dom')),
            loaded: true,
          }
        }
      }
    };

    return this.shareScope;
  }

  /**
   * 타임아웃 지원 스크립트 로드
   */
  private static loadScriptWithTimeout(
    url: string,
    timeout: number = 10000
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const existingScript = document.querySelector(`script[src="${url}"]`);
      if (existingScript) {
        resolve();
        return;
      }

      const script = document.createElement('script');
      script.src = url;
      script.type = 'text/javascript';
      script.async = true;

      const timer = setTimeout(() => {
        script.remove();
        reject(new Error(`Script loading timeout: ${url}`));
      }, timeout);

      script.onload = () => {
        clearTimeout(timer);
        resolve();
      };

      script.onerror = () => {
        clearTimeout(timer);
        script.remove();
        reject(new Error(`Script loading failed: ${url}`));
      };

      document.head.appendChild(script);
    });
  }

  /**
   * 재시도 로직
   */
  private static async retry<T>(
    fn: () => Promise<T>,
    retries: number,
    delay: number = 1000
  ): Promise<T> {
    try {
      return await fn();
    } catch (error) {
      if (retries === 0) throw error;

      console.warn(`Retrying... (${retries} attempts left)`);
      await new Promise((resolve) => setTimeout(resolve, delay));

      return this.retry(fn, retries - 1, delay);
    }
  }

  /**
   * 메인 로딩 함수
   */
  static async load<T = any>(config: LoaderConfig): Promise<T> {
    const {
      url,
      scope,
      module,
      retries = 3,
      timeout = 10000,
      onProgress
    } = config;

    return this.retry(
      async () => {
        // 1. 스크립트 로드
        onProgress?.('loading-script');
        await this.loadScriptWithTimeout(url, timeout);

        // 2. 컨테이너 가져오기
        onProgress?.('getting-container');
        let container = this.containers.get(scope);

        if (!container) {
          // @ts-ignore
          container = window[scope];

          if (!container) {
            throw new Error(`Container "${scope}" not found`);
          }

          this.containers.set(scope, container);
        }

        // 3. Share Scope 초기화
        onProgress?.('initializing-share-scope');
        const shareScope = this.initShareScope();
        await container.init(shareScope);

        // 4. 모듈 가져오기
        onProgress?.('loading-module');
        const factory = await container.get(module);
        const Module = factory();

        onProgress?.('completed');
        return Module;
      },
      retries
    );
  }

  /**
   * 프리로드 (사전에 스크립트만 로드)
   */
  static async preload(url: string, scope: string): Promise<void> {
    await this.loadScriptWithTimeout(url);

    // @ts-ignore
    const container = window[scope];
    if (container) {
      this.containers.set(scope, container);
      const shareScope = this.initShareScope();
      await container.init(shareScope);
    }
  }

  /**
   * 캐시 클리어
   */
  static clearCache() {
    this.containers.clear();
  }
}
```

### 4.2 사용 예시

```typescript
'use client';

import React from 'react';
import { FederationLoader } from '@/lib/advancedFederationLoader';

export default function AdvancedExample() {
  const [Component, setComponent] = React.useState<React.ComponentType | null>(null);
  const [progress, setProgress] = React.useState<string>('');
  const [error, setError] = React.useState<Error | null>(null);

  React.useEffect(() => {
    FederationLoader.load({
      url: 'http://localhost:3001/remoteEntry.js',
      scope: 'remoteApp',
      module: './Button',
      retries: 3,
      timeout: 10000,
      onProgress: (stage) => {
        setProgress(stage);
        console.log('📦 Progress:', stage);
      }
    })
      .then((mod) => {
        setComponent(() => mod.default || mod);
      })
      .catch((err) => {
        console.error('❌ Error:', err);
        setError(err);
      });
  }, []);

  if (error) return <div>Error: {error.message}</div>;
  if (!Component) return <div>Loading... ({progress})</div>;

  return <Component label="Advanced Button" />;
}
```

---

## 5. 전체 통합 예제

### 5.1 실행 순서

```bash
# 1. Remote 앱 실행
cd remote-app
npm install
npm start
# → http://localhost:3001

# 2. Host 앱 실행
cd host-app
npm install
npm run dev
# → http://localhost:3000

# 3. 브라우저에서 확인
# http://localhost:3000/remote-components
```

### 5.2 실행 흐름

```
1. 사용자가 /remote-components 페이지 접근
   ↓
2. <Script> 태그로 http://localhost:3001/remoteEntry.js 로드
   ↓
3. window.remoteApp 전역 변수 생성됨
   ↓
4. RemoteWrapper 컴포넌트 마운트
   ↓
5. loadRemoteModule() 실행
   ├─ getContainer('remoteApp') → window.remoteApp
   ├─ createShareScope() → Host의 react/react-dom 등록
   ├─ container.init(shareScope) → Remote의 shared 모듈 초기화
   └─ container.get('./Button') → 모듈 팩토리 반환
   ↓
6. factory() 실행 → React 컴포넌트 반환
   ↓
7. 컴포넌트 렌더링
```

---

## 6. 트러블슈팅

### 6.1 CORS 에러

**문제**:
```
Access to script at 'http://localhost:3001/remoteEntry.js' from origin
'http://localhost:3000' has been blocked by CORS policy
```

**해결**:
```javascript
// remote-app/webpack.config.js
devServer: {
  headers: {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, PATCH, OPTIONS',
    'Access-Control-Allow-Headers': 'X-Requested-With, content-type, Authorization'
  }
}
```

### 6.2 React 버전 불일치

**문제**:
```
Warning: Invalid hook call. Hooks can only be called inside of the body
of a function component.
```

**원인**: Host와 Remote가 다른 React 인스턴스 사용

**해결**:
```javascript
// Remote webpack.config.js
shared: {
  react: {
    singleton: true,  // ✅ 반드시 true
    requiredVersion: '^18.0.0',
  },
  'react-dom': {
    singleton: true,  // ✅ 반드시 true
    requiredVersion: '^18.0.0',
  }
}
```

### 6.3 "Container not found" 에러

**문제**:
```typescript
Error: Container "remoteApp" not found
```

**원인**: 스크립트 로드 전에 컨테이너 접근

**해결**:
```typescript
// ✅ Script onLoad 콜백 사용
<Script
  src={REMOTE_URL}
  onLoad={() => setScriptLoaded(true)}
/>

{scriptLoaded && <RemoteComponent />}
```

### 6.4 SSR 에러

**문제**:
```
ReferenceError: window is not defined
```

**원인**: Server Component에서 window 접근

**해결**:
```typescript
// ✅ 반드시 'use client' 추가
'use client';

import React from 'react';
// ...
```

### 6.5 청크 로딩 실패

**문제**:
```
ChunkLoadError: Loading chunk src_components_Button_tsx failed
```

**원인**: publicPath 미설정

**해결**:
```javascript
// remote-app/webpack.config.js
output: {
  publicPath: 'http://localhost:3001/',  // ✅ 절대 경로
}
```

---

## 7. 프로덕션 배포 고려사항

### 7.1 환경 변수 활용

```typescript
// lib/config.ts
export const REMOTES = {
  remoteApp: {
    url: process.env.NEXT_PUBLIC_REMOTE_APP_URL || 'http://localhost:3001/remoteEntry.js',
    scope: 'remoteApp',
  }
};
```

```bash
# .env.production
NEXT_PUBLIC_REMOTE_APP_URL=https://remote.example.com/remoteEntry.js
```

### 7.2 버전 관리

```javascript
// remote-app/webpack.config.js
output: {
  filename: '[name].[contenthash:8].js',
  publicPath: 'https://cdn.example.com/remote-app/v1.2.3/',
}

plugins: [
  new ModuleFederationPlugin({
    name: 'remoteApp',
    filename: 'remoteEntry.[contenthash:8].js',  // 캐싱 대응
  })
]
```

### 7.3 에러 바운더리

```typescript
'use client';

import React from 'react';

interface Props {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export class RemoteErrorBoundary extends React.Component<
  Props,
  { hasError: boolean; error: Error | null }
> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Remote component error:', error, errorInfo);
    // 에러 로깅 서비스로 전송
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div>
          <h2>Something went wrong loading remote component</h2>
          <details>
            <summary>Error details</summary>
            <pre>{this.state.error?.message}</pre>
          </details>
        </div>
      );
    }

    return this.props.children;
  }
}
```

---

## 마무리

이제 Next.js App Router 환경에서 Module Federation을 수동으로 구현하는 방법을 완벽하게 이해했습니다!

**핵심 요약**:
1. **Remote**: 표준 Webpack + ModuleFederationPlugin
2. **Host**: 수동 스크립트 로딩 + `get`/`init` 호출
3. **Share Scope**: 수동 관리 필요
4. **Client Component**: 필수

**다음 단계**:
- 실제 프로젝트에 적용
- 성능 최적화 (preloading, caching)
- 모니터링 및 에러 추적 설정
