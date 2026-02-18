# Module Federation 작동 원리 심층 분석

## 목차
1. [Module Federation 개요](#1-module-federation-개요)
2. [플러그인 구조 분석](#2-플러그인-구조-분석)
3. [Container (Expose) 원리](#3-container-expose-원리)
4. [Remote (Consume) 원리](#4-remote-consume-원리)
5. [Share Scope 메커니즘](#5-share-scope-메커니즘)
6. [런타임 코드 생성](#6-런타임-코드-생성)

---

## 1. Module Federation 개요

### 1.1 핵심 개념

**Module Federation**은 여러 독립적인 빌드를 런타임에 결합하는 기술입니다.

```
┌─────────────────┐         ┌─────────────────┐
│   Host App      │         │   Remote App    │
│                 │         │                 │
│  ┌───────────┐  │         │  ┌───────────┐  │
│  │ Container │  │◀────────│  │ Container │  │
│  │  Entry    │  │  HTTP   │  │  Entry    │  │
│  └───────────┘  │         │  └───────────┘  │
│                 │         │                 │
│  ┌───────────┐  │         │  ┌───────────┐  │
│  │  Shared   │◀─┼─────────┼─▶│  Shared   │  │
│  │   Scope   │  │         │  │   Scope   │  │
│  └───────────┘  │         │  └───────────┘  │
└─────────────────┘         └─────────────────┘
```

### 1.2 3가지 핵심 컴포넌트

| 컴포넌트 | 역할 | 플러그인 |
|---------|------|---------|
| **Container** | 모듈 노출 (expose) | ContainerPlugin |
| **Remote** | 외부 모듈 소비 (consume) | ContainerReferencePlugin |
| **Shared** | 의존성 공유 | SharePlugin |

---

## 2. 플러그인 구조 분석

### 2.1 ModuleFederationPlugin - 통합 인터페이스

**파일**: [lib/container/ModuleFederationPlugin.js](lib/container/ModuleFederationPlugin.js:78-128)

```javascript
class ModuleFederationPlugin {
    constructor(options) {
        this._options = options;
    }

    apply(compiler) {
        const { _options: options } = this;

        compiler.hooks.afterPlugins.tap('ModuleFederationPlugin', () => {
            // 1. Container 설정이 있으면 ContainerPlugin 적용
            if (options.exposes) {
                new ContainerPlugin({
                    name: options.name,
                    library: options.library,
                    filename: options.filename,
                    runtime: options.runtime,
                    shareScope: options.shareScope,
                    exposes: options.exposes
                }).apply(compiler);
            }

            // 2. Remote 설정이 있으면 ContainerReferencePlugin 적용
            if (options.remotes) {
                new ContainerReferencePlugin({
                    remoteType: options.remoteType,
                    shareScope: options.shareScope,
                    remotes: options.remotes
                }).apply(compiler);
            }

            // 3. Shared 설정이 있으면 SharePlugin 적용
            if (options.shared) {
                new SharePlugin({
                    shared: options.shared,
                    shareScope: options.shareScope
                }).apply(compiler);
            }

            // 4. 최적화 플러그인
            new HoistContainerReferences().apply(compiler);
        });
    }
}
```

**핵심**: ModuleFederationPlugin은 단순히 3개의 플러그인을 조합하는 래퍼입니다!

---

## 3. Container (Expose) 원리

### 3.1 ContainerPlugin - 모듈 노출

**파일**: [lib/container/ContainerPlugin.js](lib/container/ContainerPlugin.js:71-115)

**작동 방식**:
```javascript
class ContainerPlugin {
    apply(compiler) {
        const { name, exposes, shareScope, filename, library } = this._options;

        // make 훅: 컴파일 시작 시 실행
        compiler.hooks.make.tapAsync('ContainerPlugin', (compilation, callback) => {
            // ContainerEntryDependency 생성
            const dep = new ContainerEntryDependency(name, exposes, shareScope);
            dep.loc = { name };

            // 엔트리로 추가 (별도의 번들 생성)
            compilation.addEntry(
                compilation.options.context,
                dep,
                {
                    name,
                    filename,  // 예: 'remoteEntry.js'
                    library    // 전역 변수로 노출
                },
                callback
            );
        });

        // Dependency Factory 등록
        compiler.hooks.thisCompilation.tap('ContainerPlugin', (compilation) => {
            compilation.dependencyFactories.set(
                ContainerEntryDependency,
                new ContainerEntryModuleFactory()
            );
        });
    }
}
```

### 3.2 ContainerEntryModule - 핵심 모듈

**파일**: [lib/container/ContainerEntryModule.js](lib/container/ContainerEntryModule.js)

**build 단계**: 노출할 모듈을 AsyncDependenciesBlock으로 구성
```javascript
class ContainerEntryModule extends Module {
    build(options, compilation, resolver, fs, callback) {
        this.clearDependenciesAndBlocks();

        // 각 expose 항목을 비동기 블록으로 변환
        for (const [name, options] of this._exposes) {
            const block = new AsyncDependenciesBlock(
                { name: options.name },
                { name },
                options.import[options.import.length - 1]
            );

            // 각 import를 의존성으로 추가
            for (const request of options.import) {
                const dep = new ContainerExposedDependency(name, request);
                block.addDependency(dep);
            }

            this.addBlock(block);
        }

        // get, init 함수를 export
        this.addDependency(new StaticExportsDependency(['get', 'init'], false));
        callback();
    }
}
```

### 3.3 생성되는 런타임 코드

**파일**: [lib/container/ContainerEntryModule.js](lib/container/ContainerEntryModule.js:146-240)

```javascript
codeGeneration({ moduleGraph, chunkGraph, runtimeTemplate }) {
    const source = Template.asString([
        // 1. moduleMap: 노출된 모듈 매핑
        "var moduleMap = {",
        Template.indent([
            `"./Component": function() {`,
            Template.indent([
                `return __webpack_require__.e("src_Component_js").then(() => {`,
                Template.indent([
                    `return () => __webpack_require__("./src/Component.js");`
                ]),
                `});`
            ]),
            `}`
        ]),
        "};",

        // 2. get 함수: 모듈을 가져오는 함수
        `var get = function(module, getScope) {`,
        Template.indent([
            `__webpack_require__.currentRemoteGetScope = getScope;`,
            `getScope = (`,
            Template.indent([
                `__webpack_require__.o(moduleMap, module)`,
                Template.indent([
                    `? moduleMap[module]()`,
                    `: Promise.reject(new Error('Module "' + module + '" does not exist'))`
                ])
            ]),
            `);`,
            `__webpack_require__.currentRemoteGetScope = undefined;`,
            `return getScope;`
        ]),
        `};`,

        // 3. init 함수: share scope 초기화
        `var init = function(shareScope, initScope) {`,
        Template.indent([
            `if (!__webpack_require__.S) return;`,
            `var name = "default"`,
            `var oldScope = __webpack_require__.S[name];`,
            `if(oldScope && oldScope !== shareScope) {`,
            Template.indent([
                `throw new Error("Container initialization failed");`
            ]),
            `}`,
            `__webpack_require__.S[name] = shareScope;`,
            `return __webpack_require__.I(name, initScope);`
        ]),
        `};`,

        // 4. exports 설정
        `__webpack_require__.d(exports, {`,
        Template.indent([
            `get: function() { return get; },`,
            `init: function() { return init; }`
        ]),
        `});`
    ]);

    return { sources: new Map([["javascript", new RawSource(source)]]) };
}
```

**생성된 코드 예시** (간소화):
```javascript
// remoteEntry.js
var moduleMap = {
    "./Component": function() {
        return __webpack_require__.e("Component_chunk").then(() => {
            return () => __webpack_require__("./src/Component.js");
        });
    }
};

var get = function(module, getScope) {
    __webpack_require__.currentRemoteGetScope = getScope;
    var result = moduleMap[module]
        ? moduleMap[module]()
        : Promise.reject(new Error('Module not found'));
    __webpack_require__.currentRemoteGetScope = undefined;
    return result;
};

var init = function(shareScope, initScope) {
    __webpack_require__.S["default"] = shareScope;
    return __webpack_require__.I("default", initScope);
};

export { get, init };
```

---

## 4. Remote (Consume) 원리

### 4.1 ContainerReferencePlugin - 외부 모듈 참조

**파일**: [lib/container/ContainerReferencePlugin.js](lib/container/ContainerReferencePlugin.js:63-136)

```javascript
class ContainerReferencePlugin {
    apply(compiler) {
        const { _remotes: remotes, _remoteType: remoteType } = this;

        // 1. External 설정
        const remoteExternals = {};
        for (const [key, config] of remotes) {
            for (let i = 0; i < config.external.length; i++) {
                remoteExternals[
                    `webpack/container/reference/${key}${i ? `/fallback-${i}` : ""}`
                ] = config.external[i];
            }
        }
        new ExternalsPlugin(remoteType, remoteExternals).apply(compiler);

        // 2. NormalModuleFactory 훅 - 모듈 생성 가로채기
        compiler.hooks.compilation.tap(
            'ContainerReferencePlugin',
            (compilation, { normalModuleFactory }) => {
                normalModuleFactory.hooks.factorize.tap(
                    'ContainerReferencePlugin',
                    (data) => {
                        // "remoteApp/Component" 같은 요청 감지
                        for (const [key, config] of remotes) {
                            if (data.request.startsWith(`${key}/`)) {
                                // RemoteModule 반환 (일반 모듈 대신)
                                return new RemoteModule(
                                    data.request,
                                    config.external,
                                    `.${data.request.slice(key.length)}`,
                                    config.shareScope
                                );
                            }
                        }
                    }
                );
            }
        );
    }
}
```

**핵심 메커니즘**:
1. `import Component from 'remoteApp/Component'` 요청 발생
2. `factorize` 훅에서 요청 가로채기
3. `RemoteModule` 인스턴스 생성 (일반 파일 모듈 대신)
4. RemoteModule은 런타임에 외부 컨테이너에서 모듈 로드

### 4.2 RemoteModule - 동적 모듈 로더

**파일**: [lib/container/RemoteModule.js](lib/container/RemoteModule.js:40-158)

```javascript
class RemoteModule extends Module {
    constructor(request, externalRequests, internalRequest, shareScope) {
        super(WEBPACK_MODULE_TYPE_REMOTE);
        this.request = request;              // "remoteApp/Component"
        this.externalRequests = externalRequests;  // ["remoteApp@http://..."]
        this.internalRequest = internalRequest;    // "./Component"
        this.shareScope = shareScope;        // "default"
    }

    build(options, compilation, resolver, fs, callback) {
        // External 의존성 추가
        if (this.externalRequests.length === 1) {
            this.addDependency(
                new RemoteToExternalDependency(this.externalRequests[0])
            );
        } else {
            // Fallback 지원
            this.addDependency(
                new FallbackDependency(this.externalRequests)
            );
        }
        callback();
    }

    codeGeneration({ moduleGraph, chunkGraph }) {
        const module = moduleGraph.getModule(this.dependencies[0]);
        const id = chunkGraph.getModuleId(module);

        // 런타임 코드: 외부 컨테이너 초기화
        const sources = new Map();
        sources.set('remote', new RawSource(''));

        const data = new Map();
        data.set('share-init', [{
            shareScope: this.shareScope,
            initStage: 20,
            init: `initExternal(${JSON.stringify(id)});`
        }]);

        return { sources, data, runtimeRequirements: RUNTIME_REQUIREMENTS };
    }
}
```

### 4.3 RemoteRuntimeModule - 런타임 로딩 로직

**파일**: [lib/container/RemoteRuntimeModule.js](lib/container/RemoteRuntimeModule.js:27-141)

**생성되는 런타임 코드**:
```javascript
class RemoteRuntimeModule extends RuntimeModule {
    generate() {
        return Template.asString([
            // 청크 ID → Remote Module ID 매핑
            `var chunkMapping = {`,
            `  "main": [123, 456]`,
            `};`,

            // Module ID → [shareScope, moduleName, externalModuleId]
            `var idToExternalAndNameMapping = {`,
            `  123: ["default", "./Component", 789]`,
            `};`,

            // ensureChunkHandlers에 remotes 핸들러 등록
            `__webpack_require__.f.remotes = function(chunkId, promises) {`,
            Template.indent([
                `if (chunkMapping[chunkId]) {`,
                Template.indent([
                    `chunkMapping[chunkId].forEach(function(id) {`,
                    Template.indent([
                        `var data = idToExternalAndNameMapping[id];`,
                        `var promise = __webpack_require__(data[2])  // external 로드`,
                        Template.indent([
                            `.then(function(external) {`,
                            Template.indent([
                                `return __webpack_require__.I(data[0])  // share scope 초기화`,
                                Template.indent([
                                    `.then(function() {`,
                                    Template.indent([
                                        `return external.get(data[1]);  // 모듈 가져오기`
                                    ]),
                                    `});`
                                ]),
                            ]),
                            `})`,
                            `.then(function(factory) {`,
                            Template.indent([
                                `__webpack_require__.m[id] = function(module) {`,
                                Template.indent([
                                    `module.exports = factory();`
                                ]),
                                `};`
                            ]),
                            `});`
                        ]),
                        `promises.push(promise);`
                    ]),
                    `});`
                ]),
                `}`
            ]),
            `};`
        ]);
    }
}
```

**실제 생성 코드 예시**:
```javascript
__webpack_require__.f.remotes = function(chunkId, promises) {
    if (chunkMapping[chunkId]) {
        chunkMapping[chunkId].forEach(function(id) {
            var data = idToExternalAndNameMapping[id];

            // 1. External 컨테이너 로드
            var promise = __webpack_require__(data[2])
                .then(function(external) {
                    // 2. Share Scope 초기화
                    return __webpack_require__.I(data[0])
                        .then(function() {
                            // 3. 모듈 가져오기
                            return external.get(data[1]);
                        });
                })
                .then(function(factory) {
                    // 4. 모듈 팩토리 등록
                    __webpack_require__.m[id] = function(module) {
                        module.exports = factory();
                    };
                });

            promises.push(promise);
        });
    }
};
```

---

## 5. Share Scope 메커니즘

### 5.1 SharePlugin - 의존성 공유

**파일**: [lib/sharing/SharePlugin.js](lib/sharing/SharePlugin.js:19-91)

```javascript
class SharePlugin {
    constructor(options) {
        const sharedOptions = parseOptions(options.shared);

        // Consume용 설정
        this._consumes = sharedOptions.map(([key, options]) => ({
            [key]: {
                import: options.import,
                shareKey: options.shareKey || key,
                requiredVersion: options.requiredVersion,
                singleton: options.singleton,
                eager: options.eager
            }
        }));

        // Provide용 설정
        this._provides = sharedOptions.map(([key, options]) => ({
            [key]: {
                shareKey: options.shareKey || key,
                version: options.version,
                eager: options.eager
            }
        }));
    }

    apply(compiler) {
        // 소비자 플러그인
        new ConsumeSharedPlugin({
            shareScope: this._shareScope,
            consumes: this._consumes
        }).apply(compiler);

        // 제공자 플러그인
        new ProvideSharedPlugin({
            shareScope: this._shareScope,
            provides: this._provides
        }).apply(compiler);
    }
}
```

### 5.2 Share Scope 데이터 구조

```javascript
// 런타임에 생성되는 전역 Share Scope
__webpack_require__.S = {
    "default": {
        "react": {
            "18.2.0": {
                get: () => Promise.resolve(() => require("react")),
                loaded: 1,
                from: "host"
            },
            "18.1.0": {
                get: () => Promise.resolve(() => require("react")),
                loaded: 1,
                from: "remote1"
            }
        },
        "react-dom": {
            "18.2.0": {
                get: () => Promise.resolve(() => require("react-dom")),
                loaded: 1,
                from: "host"
            }
        }
    }
};
```

### 5.3 버전 해결 알고리즘

```javascript
// 생성되는 런타임 코드 (간소화)
function resolveSharedModule(shareScope, shareKey, requiredVersion) {
    var scope = __webpack_require__.S[shareScope];
    var versions = scope[shareKey];

    // 1. 모든 버전 수집
    var availableVersions = Object.keys(versions);

    // 2. 요구사항 충족하는 최신 버전 찾기
    var bestVersion = availableVersions
        .filter(v => satisfies(v, requiredVersion))
        .sort((a, b) => compareVersions(b, a))[0];

    // 3. singleton 옵션 확인
    if (config.singleton && availableVersions.length > 1) {
        console.warn('Singleton violation:', shareKey);
    }

    // 4. 해당 버전의 get 함수 반환
    return versions[bestVersion].get();
}
```

### 5.4 초기화 프로세스

```javascript
// __webpack_require__.I: Share Scope 초기화 함수
__webpack_require__.I = function(name, initScope) {
    if (!initScope) initScope = [];

    var scope = __webpack_require__.S[name];
    if (!scope) return Promise.resolve();

    var promises = [];

    // 각 shared 모듈 초기화
    Object.keys(scope).forEach(function(key) {
        var versions = scope[key];
        Object.keys(versions).forEach(function(version) {
            var entry = versions[version];

            // 이미 초기화된 경우 스킵
            if (entry.loaded) return;

            // 초기화 프로미스 생성
            var promise = entry.get().then(function(module) {
                entry.loaded = 1;
                return module;
            });

            promises.push(promise);
        });
    });

    return Promise.all(promises);
};
```

---

## 6. 런타임 코드 생성

### 6.1 전체 실행 흐름

```
┌──────────────────────────────────────────────────────────────┐
│ Host 앱 시작                                                  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 1. Share Scope 초기화                                         │
│    __webpack_require__.S = { "default": {} }                 │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. 자신의 shared 모듈 등록                                     │
│    __webpack_require__.S["default"]["react"] = {             │
│      "18.2.0": { get: () => ..., loaded: 1 }                │
│    }                                                         │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. Remote 모듈 import 발생                                    │
│    import('remoteApp/Component')                             │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. RemoteRuntimeModule.remotes 핸들러 실행                   │
│    - External 스크립트 로드 (remoteEntry.js)                  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 5. Remote 컨테이너 초기화                                      │
│    remoteApp.init(__webpack_require__.S["default"])          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 6. Remote의 shared 모듈도 Share Scope에 등록                  │
│    Share Scope 병합 및 버전 해결                              │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 7. Remote 모듈 가져오기                                        │
│    remoteApp.get('./Component')                              │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 8. 팩토리 함수 실행 → 모듈 반환                                │
│    factory() → React Component                               │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 실제 코드 예시

**Host 앱**:
```javascript
// Host 번들에 생성되는 코드
import('remoteApp/Component').then(Component => {
    // RemoteRuntimeModule이 다음과 같이 변환:

    // 1. remoteEntry.js 로드
    __webpack_require__.e("remoteApp")
        .then(() => __webpack_require__("webpack/container/reference/remoteApp"))
        .then(container => {
            // 2. Share Scope 초기화
            return __webpack_require__.I("default")
                .then(() => container.init(__webpack_require__.S["default"]))
                .then(() => {
                    // 3. 모듈 가져오기
                    return container.get("./Component");
                });
        })
        .then(factory => {
            // 4. 컴포넌트 사용
            const Component = factory();
            // ...
        });
});
```

**Remote 앱 (remoteEntry.js)**:
```javascript
var moduleMap = {
    "./Component": function() {
        return __webpack_require__.e("Component_chunk")
            .then(() => () => __webpack_require__("./src/Component.js"));
    }
};

var get = function(module, getScope) {
    __webpack_require__.currentRemoteGetScope = getScope;
    var result = moduleMap[module]();
    __webpack_require__.currentRemoteGetScope = undefined;
    return result;
};

var init = function(shareScope, initScope) {
    // Host의 Share Scope 받기
    __webpack_require__.S["default"] = shareScope;

    // Remote의 shared 모듈 등록
    return __webpack_require__.I("default", initScope);
};

export { get, init };
```

---

## 7. 핵심 인사이트

### 7.1 플러그인별 책임

| 플러그인 | 컴파일 타임 | 런타임 |
|---------|-------------|--------|
| **ContainerPlugin** | ContainerEntryModule 생성 | `get`/`init` 함수 export |
| **ContainerReferencePlugin** | RemoteModule 생성 | External 로딩 + 초기화 |
| **SharePlugin** | ConsumeShared/ProvideShared 모듈 | Share Scope 관리 |

### 7.2 Module Federation의 마법

1. **NormalModuleFactory.factorize 훅 활용**
   - 일반 파일 모듈 대신 RemoteModule 반환
   - 웹팩의 모듈 시스템을 그대로 활용

2. **AsyncDependenciesBlock 활용**
   - 코드 스플리팅 메커니즘 재사용
   - 동적 import와 동일한 방식으로 처리

3. **전역 Share Scope**
   - 런타임에 모든 앱이 공유하는 전역 객체
   - 버전 충돌 해결 및 싱글톤 보장

### 7.3 설정 예시와 생성 코드 매핑

**설정**:
```javascript
new ModuleFederationPlugin({
    name: "host",
    remotes: {
        remote1: "remote1@http://localhost:3001/remoteEntry.js"
    },
    shared: {
        react: { singleton: true, requiredVersion: "^18.0.0" }
    }
})
```

**생성되는 코드**:
```javascript
// 1. External 설정
externals: {
    "webpack/container/reference/remote1": "remote1@http://localhost:3001/remoteEntry.js"
}

// 2. RemoteModule 생성
import Component from 'remote1/Component'
→ new RemoteModule(
    "remote1/Component",
    ["webpack/container/reference/remote1"],
    "./Component",
    "default"
)

// 3. Share Scope 등록
__webpack_require__.S["default"]["react"] = {
    "18.2.0": {
        get: () => Promise.resolve(() => require("react")),
        loaded: 1
    }
}
```

---

## 8. 다음 단계

이제 Module Federation의 내부 작동 원리를 이해했습니다. 다음 문서에서는:

1. **Next.js App Router 환경에서의 제약사항**
2. **수동 스크립트 로딩 구현 방법**
3. **실전 예제 코드**

를 다루겠습니다.

→ [NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md](NEXTJS_MODULE_FEDERATION_IMPLEMENTATION.md)
