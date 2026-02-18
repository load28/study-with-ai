# 웹팩 핵심 아키텍처 분석 (코드 기반)

## 목차
1. [웹팩 빌드 플로우 개요](#1-웹팩-빌드-플로우-개요)
2. [핵심 클래스 상세 분석](#2-핵심-클래스-상세-분석)
3. [플러그인 시스템 (Tapable)](#3-플러그인-시스템-tapable)
4. [모듈 시스템 아키텍처](#4-모듈-시스템-아키텍처)

---

## 1. 웹팩 빌드 플로우 개요

### 1.1 엔트리 포인트: webpack() 함수

**파일**: [lib/webpack.js](lib/webpack.js)

```javascript
// lib/webpack.js:137-210
const webpack = (options, callback) => {
    const create = () => {
        // 1. 옵션 검증
        validateSchema(webpackOptionsSchema, options);

        // 2. Compiler 또는 MultiCompiler 생성
        let compiler;
        if (Array.isArray(options)) {
            compiler = createMultiCompiler(options);
        } else {
            compiler = createCompiler(options);
        }

        return { compiler, watch, watchOptions };
    };

    // 3. 실행 또는 watch 모드
    if (callback) {
        const { compiler, watch } = create();
        if (watch) {
            compiler.watch(watchOptions, callback);
        } else {
            compiler.run((err, stats) => {
                compiler.close((err2) => callback(err || err2, stats));
            });
        }
    }
};
```

**핵심 흐름**:
1. **옵션 정규화**: `getNormalizedWebpackOptions()` - 사용자 설정을 표준 형식으로 변환
2. **Compiler 생성**: `createCompiler()` - 빌드 프로세스의 중앙 관리자 생성
3. **플러그인 적용**: 모든 플러그인의 `apply()` 메서드 호출
4. **실행**: `compiler.run()` 또는 `compiler.watch()` 호출

---

## 2. 핵심 클래스 상세 분석

### 2.1 Compiler - 빌드 생명주기 관리자

**파일**: [lib/Compiler.js](lib/Compiler.js)

**역할**:
- 웹팩의 전체 빌드 프로세스를 관리하는 싱글톤 클래스
- 모든 빌드 생명주기 훅(Hook)을 제공
- 파일 시스템 접근 관리

**주요 속성**:
```javascript
class Compiler {
    constructor(context, options) {
        // 웹팩 설정
        this.options = options;
        this.context = context;

        // 파일 시스템
        this.inputFileSystem = null;
        this.outputFileSystem = null;

        // 훅 (Tapable)
        this.hooks = {
            initialize: new SyncHook([]),
            beforeRun: new AsyncSeriesHook(["compiler"]),
            run: new AsyncSeriesHook(["compiler"]),
            emit: new AsyncSeriesHook(["compilation"]),
            afterEmit: new AsyncSeriesHook(["compilation"]),
            done: new AsyncSeriesHook(["stats"]),
            // ... 더 많은 훅들
        };

        // 모듈 팩토리
        this.normalModuleFactory = null;
        this.contextModuleFactory = null;
    }
}
```

**핵심 메서드**:

1. **createCompilation()** - 새로운 Compilation 인스턴스 생성
   ```javascript
   // lib/Compiler.js (대략적인 구조)
   createCompilation() {
       const compilation = new Compilation(this);
       // NormalModuleFactory와 ContextModuleFactory 생성
       const params = {
           normalModuleFactory: this.createNormalModuleFactory(),
           contextModuleFactory: this.createContextModuleFactory()
       };
       this.hooks.compilation.call(compilation, params);
       return compilation;
   }
   ```

2. **compile()** - 실제 컴파일 프로세스 시작
   ```javascript
   compile(callback) {
       const params = this.newCompilationParams();
       this.hooks.beforeCompile.callAsync(params, err => {
           this.hooks.compile.call(params);
           const compilation = this.newCompilation(params);
           this.hooks.make.callAsync(compilation, err => {
               // 모듈 빌드 및 의존성 처리
               compilation.finish(err => {
                   compilation.seal(err => {
                       // 청크 생성 및 최적화
                       this.hooks.afterCompile.callAsync(compilation, callback);
                   });
               });
           });
       });
   }
   ```

---

### 2.2 Compilation - 실제 빌드 실행

**파일**: [lib/Compilation.js](lib/Compilation.js)

**역할**:
- 개별 빌드 프로세스 실행 (watch 모드에서는 여러 번 생성됨)
- 모듈 그래프 구성 및 관리
- 청크 생성 및 코드 생성

**주요 속성**:
```javascript
class Compilation {
    constructor(compiler, params) {
        this.compiler = compiler;
        this.hooks = {
            buildModule: new SyncHook(["module"]),
            succeedModule: new SyncHook(["module"]),
            finishModules: new AsyncSeriesHook(["modules"]),
            seal: new SyncHook([]),
            optimizeChunks: new SyncBailHook(["chunks"]),
            // ... 100+ 개의 훅
        };

        // 모듈 및 청크 관리
        this.modules = new Set();
        this.chunks = new Set();
        this.entries = new Map();

        // 그래프 구조
        this.moduleGraph = new ModuleGraph();
        this.chunkGraph = new ChunkGraph(this.moduleGraph);

        // 의존성 관리
        this.dependencyFactories = new Map();
        this.dependencyTemplates = new DependencyTemplates();
    }
}
```

**핵심 프로세스**:

1. **addEntry()** - 엔트리 포인트 추가
   ```javascript
   // lib/Compilation.js
   addEntry(context, entry, optionsOrName, callback) {
       // 1. 엔트리 의존성 생성
       // 2. ModuleFactory를 통해 모듈 생성
       // 3. 모듈을 모듈 그래프에 추가
       // 4. 의존성 재귀 처리
   }
   ```

2. **seal()** - 모듈 그래프 봉인 및 청크 생성
   ```javascript
   seal(callback) {
       this.hooks.seal.call();

       // 1. 청크 그래프 빌드
       buildChunkGraph(this, this.entries);

       // 2. 청크 최적화
       this.hooks.optimizeChunks.call(this.chunks);

       // 3. 코드 생성
       this.codeGeneration(err => {
           // 4. 런타임 요구사항 처리
           this.createChunkAssets();
           this.hooks.optimizeAssets.callAsync(this.assets, callback);
       });
   }
   ```

---

### 2.3 NormalModuleFactory - 모듈 생성 팩토리

**파일**: [lib/NormalModuleFactory.js](lib/NormalModuleFactory.js)

**역할**:
- 일반 모듈(JavaScript, CSS 등) 생성
- 로더 적용
- 모듈 리졸빙

**핵심 훅**:
```javascript
class NormalModuleFactory extends ModuleFactory {
    constructor() {
        this.hooks = {
            // 모듈 생성 전
            beforeResolve: new AsyncSeriesBailHook(["resolveData"]),
            // 리졸브 중
            factorize: new AsyncSeriesBailHook(["resolveData"]),
            resolve: new AsyncSeriesBailHook(["resolveData"]),
            // 모듈 생성 후
            createModule: new AsyncSeriesBailHook(["createData", "resolveData"]),
            module: new SyncWaterfallHook(["module", "createData", "resolveData"])
        };
    }

    create(data, callback) {
        // 1. beforeResolve 훅 실행
        // 2. factorize: 모듈 타입 결정 (플러그인이 개입 가능)
        // 3. resolve: 파일 경로 리졸빙
        // 4. 로더 적용
        // 5. NormalModule 인스턴스 생성
    }
}
```

**플러그인 개입 지점**: `factorize` 훅
- **Module Federation의 핵심**: RemoteModule을 반환하여 외부 모듈 로딩 가능

---

### 2.4 ModuleGraph - 의존성 그래프

**파일**: [lib/ModuleGraph.js](lib/ModuleGraph.js)

**역할**:
- 모듈 간 의존성 관계 추적
- 모듈 메타데이터 저장

**구조**:
```javascript
class ModuleGraph {
    constructor() {
        // 모듈 -> 의존성 연결 정보
        this._dependencyMap = new Map();
        // 모듈 메타데이터
        this._metaMap = new Map();
    }

    // 의존성 추가
    setResolvedModule(originModule, dependency, module) {
        const connection = new ModuleGraphConnection(
            originModule,
            dependency,
            module
        );
        this._dependencyMap.set(dependency, connection);
    }

    // 의존성 조회
    getModule(dependency) {
        const connection = this._dependencyMap.get(dependency);
        return connection ? connection.module : null;
    }
}
```

---

### 2.5 Dependency - 의존성 표현

**파일**: [lib/Dependency.js](lib/Dependency.js)

**역할**:
- 모듈 간 의존 관계를 나타내는 추상 클래스
- 각 의존성 타입별로 구체 클래스 구현

**주요 의존성 타입**:
```javascript
// ESM import
class HarmonyImportSideEffectDependency extends Dependency {}

// CommonJS require
class CommonJsRequireDependency extends Dependency {}

// Dynamic import
class ImportDependency extends Dependency {}

// Module Federation 관련
class ContainerEntryDependency extends Dependency {}
class RemoteToExternalDependency extends Dependency {}
```

---

## 3. 플러그인 시스템 (Tapable)

### 3.1 Tapable 이란?

**핵심 개념**: 웹팩의 모든 생명주기를 이벤트 기반으로 관리하는 라이브러리

**훅 타입**:
```javascript
const {
    SyncHook,           // 동기, 반환값 무시
    SyncBailHook,       // 동기, null 아닌 값 반환 시 중단
    SyncWaterfallHook,  // 동기, 이전 결과를 다음으로 전달
    AsyncSeriesHook,    // 비동기, 순차 실행
    AsyncParallelHook   // 비동기, 병렬 실행
} = require('tapable');
```

### 3.2 플러그인 작성 패턴

```javascript
class MyPlugin {
    apply(compiler) {
        // Compiler 레벨 훅
        compiler.hooks.compile.tap('MyPlugin', (params) => {
            console.log('컴파일 시작!');
        });

        // Compilation 레벨 훅
        compiler.hooks.compilation.tap('MyPlugin', (compilation) => {
            compilation.hooks.buildModule.tap('MyPlugin', (module) => {
                console.log('모듈 빌드:', module.identifier());
            });
        });
    }
}
```

### 3.3 주요 훅 실행 순서

```
compiler.hooks.initialize
    ↓
compiler.hooks.beforeRun
    ↓
compiler.hooks.run
    ↓
compiler.hooks.compile
    ↓
compiler.hooks.make → compilation.addEntry()
    ↓                      ↓
    |           normalModuleFactory.hooks.factorize
    |                      ↓
    |           normalModuleFactory.hooks.resolve
    |                      ↓
    |           compilation.hooks.buildModule
    ↓                      ↓
compilation.hooks.finishModules
    ↓
compilation.hooks.seal
    ↓
compilation.hooks.optimizeChunks
    ↓
compilation.hooks.optimizeAssets
    ↓
compiler.hooks.emit
    ↓
compiler.hooks.afterEmit
    ↓
compiler.hooks.done
```

---

## 4. 모듈 시스템 아키텍처

### 4.1 Module 추상 클래스

**파일**: [lib/Module.js](lib/Module.js)

```javascript
class Module {
    constructor(type, context) {
        this.type = type;  // 'javascript/auto', 'javascript/esm', etc.
        this.context = context;
        this.dependencies = [];
        this.blocks = [];  // AsyncDependenciesBlock for dynamic imports
    }

    // 모듈 식별자 (고유)
    identifier() {
        throw new Error("Module.identifier: Must be overridden");
    }

    // 빌드 필요 여부
    needBuild(context, callback) {}

    // 실제 빌드 수행
    build(options, compilation, resolver, fs, callback) {}

    // 코드 생성
    codeGeneration(context) {}
}
```

### 4.2 NormalModule - 일반 파일 모듈

```javascript
class NormalModule extends Module {
    constructor({
        type,
        request,      // 원본 요청 경로
        userRequest,  // 사용자가 작성한 경로
        rawRequest,   // 원본 문자열
        loaders,      // 적용할 로더 목록
        resource,     // 실제 파일 경로
        parser,       // JavaScript 파서
        generator     // 코드 생성기
    }) {
        super(type);
        // ...
    }

    build(options, compilation, resolver, fs, callback) {
        // 1. 로더 실행
        runLoaders({
            resource: this.resource,
            loaders: this.loaders,
            context: loaderContext
        }, (err, result) => {
            // 2. 파싱 (AST 생성)
            this.parser.parse(result.source, {
                current: this,
                module: this,
                compilation
            });
            // 3. 의존성 수집 완료
            callback();
        });
    }
}
```

### 4.3 청크(Chunk) 생성 프로세스

**파일**: [lib/buildChunkGraph.js](lib/buildChunkGraph.js)

**핵심 알고리즘**:
```javascript
function buildChunkGraph(compilation, inputEntrypoints) {
    // 1. 엔트리 포인트별로 청크 생성
    for (const [name, entrypoint] of inputEntrypoints) {
        const chunk = new Chunk(name);
        entrypoint.setRuntimeChunk(chunk);
        entrypoint.pushChunk(chunk);

        // 2. 모듈을 청크에 할당
        connectChunkGroupAndChunk(entrypoint, chunk);
    }

    // 3. 모듈 그래프 순회하며 청크 연결
    for (const module of compilation.modules) {
        // 동적 import는 새로운 청크 생성
        for (const block of module.blocks) {
            const chunkGroup = new ChunkGroup(block.chunkName);
            // 비동기 청크 연결
        }
    }

    // 4. 청크 그래프에 모듈 매핑
    for (const chunk of compilation.chunks) {
        for (const module of chunkGraph.getChunkModulesIterable(chunk)) {
            chunkGraph.connectChunkAndModule(chunk, module);
        }
    }
}
```

### 4.4 코드 생성 단계

```javascript
class Compilation {
    codeGeneration(callback) {
        // 모든 모듈에 대해 코드 생성
        const jobs = [];
        for (const module of this.modules) {
            for (const runtime of chunkGraph.getModuleRuntimes(module)) {
                jobs.push({
                    module,
                    runtime,
                    hash: this._createModuleHash(module, runtime)
                });
            }
        }

        // 병렬 실행
        asyncLib.each(jobs, ({ module, runtime }, callback) => {
            const result = module.codeGeneration({
                moduleGraph: this.moduleGraph,
                chunkGraph: this.chunkGraph,
                runtimeTemplate: this.runtimeTemplate,
                runtime
            });

            this.codeGenerationResults.add(module, runtime, result);
            callback();
        }, callback);
    }
}
```

---

## 5. 핵심 데이터 흐름 정리

```
사용자 설정 (webpack.config.js)
    ↓
webpack() 함수
    ↓
Compiler 생성
    ↓
플러그인 적용 (plugin.apply(compiler))
    ↓
compiler.run() / compiler.watch()
    ↓
Compilation 생성
    ↓
엔트리 포인트 처리 (compilation.addEntry)
    ↓
NormalModuleFactory.create()
    ↓
Module 인스턴스 생성
    ↓
Module.build() - 로더 실행 + 파싱
    ↓
의존성 수집 → 재귀적으로 Module 생성
    ↓
ModuleGraph 구성 완료
    ↓
compilation.seal() - 청크 생성
    ↓
buildChunkGraph() - 모듈을 청크에 할당
    ↓
codeGeneration() - 각 모듈의 코드 생성
    ↓
createChunkAssets() - 청크별 파일 생성
    ↓
emit() - 파일 시스템에 쓰기
    ↓
done() - 빌드 완료
```

---

## 6. 주요 개념 요약

| 개념 | 역할 | 파일 위치 |
|------|------|-----------|
| **Compiler** | 빌드 생명주기 관리, 싱글톤 | [lib/Compiler.js](lib/Compiler.js) |
| **Compilation** | 개별 빌드 실행, 모듈 그래프 관리 | [lib/Compilation.js](lib/Compilation.js) |
| **Module** | 소스 파일 추상화 | [lib/Module.js](lib/Module.js) |
| **Dependency** | 모듈 간 의존 관계 표현 | [lib/Dependency.js](lib/Dependency.js) |
| **ModuleGraph** | 의존성 그래프 | [lib/ModuleGraph.js](lib/ModuleGraph.js) |
| **ChunkGraph** | 청크 그래프 (모듈 → 청크 매핑) | [lib/ChunkGraph.js](lib/ChunkGraph.js) |
| **NormalModuleFactory** | 일반 모듈 생성 팩토리 | [lib/NormalModuleFactory.js](lib/NormalModuleFactory.js) |
| **Tapable Hooks** | 플러그인 시스템의 핵심 | 모든 클래스에 존재 |

---

## 다음 문서

다음은 **Module Federation의 작동 원리**를 심층 분석하여, 위 아키텍처가 어떻게 활용되는지 살펴보겠습니다.

→ [MODULE_FEDERATION_DEEP_DIVE.md](MODULE_FEDERATION_DEEP_DIVE.md)
