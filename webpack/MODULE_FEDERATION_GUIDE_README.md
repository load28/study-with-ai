# 웹팩 & Module Federation 완벽 가이드

> **코드 기반 웹팩 아키텍처 분석 + Next.js App Router Module Federation 실전 구현**

## 📖 개요

이 가이드는 Webpack 5의 내부 아키텍처를 **실제 소스 코드**를 기반으로 분석하고, Module Federation을 활용하여 Next.js App Router 환경에서 Micro Frontend Architecture(MFA)를 구현하는 방법을 다룹니다.

## 🎯 이 가이드가 필요한 사람

- ✅ 웹팩의 내부 작동 원리를 깊이 이해하고 싶은 개발자
- ✅ Module Federation을 실전에 적용하려는 개발자
- ✅ Next.js App Router에서 MFA를 구현하려는 개발자
- ✅ 플러그인 작성 등 웹팩을 확장하고 싶은 개발자
- ✅ 코드 레벨에서 번들러의 작동 방식을 이해하고 싶은 개발자

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
- [lib/webpack.js](lib/webpack.js) - 엔트리 포인트
- [lib/Compiler.js](lib/Compiler.js) - 빌드 생명주기 관리
- [lib/Compilation.js](lib/Compilation.js) - 실제 빌드 실행
- [lib/NormalModuleFactory.js](lib/NormalModuleFactory.js) - 모듈 생성

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
- [lib/container/ModuleFederationPlugin.js](lib/container/ModuleFederationPlugin.js) - 메인 플러그인
- [lib/container/ContainerPlugin.js](lib/container/ContainerPlugin.js) - Expose 기능
- [lib/container/ContainerReferencePlugin.js](lib/container/ContainerReferencePlugin.js) - Remote 기능
- [lib/container/ContainerEntryModule.js](lib/container/ContainerEntryModule.js) - Container Entry 코드 생성
- [lib/container/RemoteRuntimeModule.js](lib/container/RemoteRuntimeModule.js) - Remote 로딩 런타임

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

### 4. [통합 가이드 (한글)](COMPLETE_GUIDE_KR.md)
**읽는 데 걸리는 시간**: 약 20분

전체 문서의 요약본으로, 핵심 개념과 실전 적용 방법을 빠르게 파악할 수 있습니다.

**다루는 내용**:
- 핵심 개념 요약
- 아키텍처 다이어그램
- 실행 가이드
- 심화 학습 주제
- 성능 최적화
- 일반적인 문제 해결

---

## 🚀 빠른 시작

### 문서 읽기 순서

#### 초보자 (웹팩 경험 부족)
```
1. COMPLETE_GUIDE_KR.md (개요 파악)
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

## 💡 핵심 개념 미리보기

### 웹팩 아키텍처

```
webpack(config)
    ↓
Compiler (싱글톤, 빌드 생명주기 관리)
    ↓
Compilation (개별 빌드 실행)
    ├─ NormalModuleFactory (모듈 생성)
    ├─ ModuleGraph (의존성 그래프)
    ├─ ChunkGraph (청크 그래프)
    └─ CodeGeneration (코드 생성)
    ↓
Assets (번들 파일)
```

### Module Federation 구조

```javascript
new ModuleFederationPlugin({
    name: 'app',
    exposes: { './Button': './src/Button' },  // → ContainerPlugin
    remotes: { remote: 'remote@url' },        // → ContainerReferencePlugin
    shared: { react: {...} }                  // → SharePlugin
})
```

### Next.js 수동 구현

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

## 📁 프로젝트 구조

이 가이드의 문서들은 다음과 같이 구성되어 있습니다:

```
webpack/
├── MODULE_FEDERATION_GUIDE_README.md     # 이 파일
├── WEBPACK_ARCHITECTURE_ANALYSIS.md      # 웹팩 아키텍처 분석
├── MODULE_FEDERATION_DEEP_DIVE.md        # Module Federation 심층 분석
├── NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md  # Next.js 구현 가이드
├── COMPLETE_GUIDE_KR.md                  # 통합 가이드 (한글)
│
├── lib/                                  # 웹팩 소스 코드
│   ├── webpack.js
│   ├── Compiler.js
│   ├── Compilation.js
│   ├── NormalModuleFactory.js
│   └── container/
│       ├── ModuleFederationPlugin.js
│       ├── ContainerPlugin.js
│       ├── ContainerReferencePlugin.js
│       ├── ContainerEntryModule.js
│       └── RemoteRuntimeModule.js
│
└── examples/
    └── module-federation/                # 공식 예제
        └── webpack.config.js
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
| COMPLETE_GUIDE_KR.md | ⭐ 쉬움 | 기본 JavaScript | 20분 |
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

## 🐛 트러블슈팅

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

## 🎯 최종 목표

이 가이드를 완료하면:

✅ 웹팩의 내부 작동 원리를 코드 레벨에서 이해
✅ Module Federation의 3가지 핵심 플러그인 작동 방식 파악
✅ Next.js App Router에서 MFA 구현 가능
✅ 커스텀 플러그인 작성 능력 습득
✅ 프로덕션 환경에서의 최적화 및 문제 해결 능력 향상

---

## 🚀 시작하기

지금 바로 첫 번째 문서를 읽어보세요!

👉 [웹팩 핵심 아키텍처 분석 시작하기](WEBPACK_ARCHITECTURE_ANALYSIS.md)

또는

👉 [통합 가이드로 빠르게 시작하기](COMPLETE_GUIDE_KR.md)

Happy Learning! 📖✨
