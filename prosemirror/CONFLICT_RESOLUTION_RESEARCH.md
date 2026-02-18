# 충돌 방지 시스템 설계 근거 및 조사 자료

## 1. ProseMirror 공식 문서 기반

### 1.1 Mark Excludes 시스템
**출처**: ProseMirror Guide - Schema 섹션

```typescript
// ProseMirror Guide에서 명시된 excludes 속성
export const code: MarkSpec = {
  excludes: '_', // 모든 다른 마크 제외
  code: true,
};
```

**ProseMirror Guide 원문 (PROSE_MIRROR_GUIDE.md, line 348-349)**:
> "마크는 인라인 콘텐츠에 추가 스타일링이나 기타 정보를 추가하는 데 사용됩니다."
> "기본적으로 인라인 콘텐츠가 있는 노드는 스키마에 정의된 모든 마크가 자식에 적용되도록 허용합니다."

**근거**:
- ProseMirror는 기본적으로 모든 마크가 공존할 수 있도록 설계됨
- `excludes` 속성을 통해 특정 마크 간 상호 배타성 정의 가능
- `"_"` 값은 모든 마크를 제외한다는 와일드카드

### 1.2 Canonicalization (정규화)
**출처**: ProseMirror Guide - 문서 구조 섹션

**ProseMirror Guide 원문 (line 159)**:
> "이것은 또한 각 문서가 하나의 유효한 표현을 갖는다는 것을 의미합니다. 동일한 마크 세트를 가진 인접한 텍스트 노드는 항상 함께 결합되며, 빈 텍스트 노드는 허용되지 않습니다. 마크가 나타나는 순서는 스키마에 의해 지정됩니다."

**근거**:
- 단일 정규 표현(canonical representation) 보장
- 동일 마크 세트의 인접 노드 자동 병합
- 스키마에서 정의한 순서로 마크 정렬

### 1.3 Mark Ordering
**출처**: ProseMirror Discussion Forum

**검색 결과에서 발견된 내용**:
> "The order in which marks appear is specified by the schema."
> "Each document has one valid representation, and adjacent text nodes with the same set of marks are always combined together"

**근거**:
- 스키마 정의 순서가 마크 정렬 순서를 결정
- 일관된 문서 표현 유지

## 2. TipTap (ProseMirror 기반) 설계

### 2.1 Priority 시스템
**출처**: TipTap 공식 문서 - Mark API

**웹 검색 결과**:
> "Priority affects the order in which extensions are processed, with higher priority extensions running first. For example, the Link mark has a higher priority, which means it will be rendered as `<a href="…"><strong>Example</strong></a>` instead of `<strong><a href="…">Example</a></strong>`."

**구현 반영**:
```typescript
const markPriority = {
  link: 100,          // 최외곽 레이어
  fontSize: 90,
  fontFamily: 85,
  color: 80,
  backgroundColor: 75,
  bold: 50,
  italic: 45,
  underline: 40,
  strikethrough: 35,
  code: 10,           // 최내곽 레이어
};
```

**근거**:
- 높은 priority = 외부 레이어 렌더링
- 링크가 포맷팅을 감싸는 구조 (UX 일관성)
- DOM 구조의 예측 가능성 향상

### 2.2 Excludes 예제
**출처**: TipTap 공식 문서

**웹 검색 결과**:
> "With the excludes attribute you can define which marks must not coexist with the mark. For example, the inline code mark excludes any other mark (bold, italic, and all others)."

```typescript
Mark.create({
  // 다른 모든 마크 제외
  excludes: '_',
  // 특정 마크만 제외
  excludes: ['bold', 'italic'],
})
```

**근거**:
- 코드 마크는 일반적으로 다른 포맷팅 제외
- 엔터프라이즈 에디터들의 일반적인 패턴

## 3. Quill Editor의 Delta Format

### 3.1 Canonicalization 요구사항
**출처**: Quill 공식 문서 - Delta Format

**웹 검색 결과**:
> "Quill adds an additional constraint that Deltas must be canonical - if two Deltas are equal, the content they represent must be equal, and there cannot be two unequal Deltas that represent the same content."

**근거**:
- 동일한 콘텐츠는 동일한 표현
- Deep comparison 가능
- 버전 관리 및 협업 편집 용이

### 3.2 Line Format 충돌
**출처**: Quill Delta Format 문서

**웹 검색 결과**:
> "For line-level formats specifically, many line formats are exclusive - for example, Quill does not allow a line to simultaneously be both a header and a list, despite being possible to represent in the Delta format."

**근거**:
- 블록 레벨 포맷은 상호 배타적
- 애플리케이션 레벨에서 제약 강제
- Delta 포맷 자체보다 상위 레벨 규칙

## 4. Google Docs의 접근 방식

### 4.1 변경 로그 기반
**출처**: Google Drive Blog

**웹 검색 결과**:
> "Google documents represent the history of a document as a series of changes, where all edits boil down to three basic types: inserting text, deleting text, and applying styles to a range of text"

**근거**:
- 모든 편집을 변경 로그로 관리
- 스타일 적용도 하나의 operation
- 타임스탬프 기반 충돌 해결

### 4.2 커스텀 렌더링 엔진
**웹 검색 결과**:
> "Google Docs doesn't use standard designMode or contentEditable features; instead, it has its own rendering engine written in JavaScript."

**근거**:
- 완전한 제어를 위한 커스텀 엔진
- 브라우저 기본 동작 의존하지 않음

## 5. CRDT 기반 충돌 해결

### 5.1 Yjs의 접근
**출처**: 웹 검색 - Real-time Collaborative Editing

**웹 검색 결과**:
> "Yjs emerged as a leading solution due to its CRDT implementation, scalability, and emphasis on network agnosticism and conflict resolution. ProseMirror's Collab module does not guarantee conflict resolution, which can lead to merge conflicts in documents."

**근거**:
- CRDT(Conflict-free Replicated Data Type)로 자동 병합
- 네트워크 독립적
- ProseMirror 기본 collab보다 강력한 충돌 해결

### 5.2 Automerge & Peritext
**출처**: 웹 검색 결과

**웹 검색 결과**:
> "Automerge 2.2 released rich text support in 2024, including a ProseMirror binding for collaborative applications with realtime and asynchronous editing."

**근거**:
- Rich text에 특화된 CRDT
- 실시간 및 비동기 편집 지원
- ProseMirror와의 통합

## 6. 실제 구현에 적용한 전략

### 6.1 toggleMarkSafe 명령
```typescript
export function toggleMarkSafe(markType: MarkType, attrs?: any): Command {
  return (state, dispatch) => {
    const { from, to } = state.selection;
    const { doc, tr } = state;

    if (dispatch) {
      const marks = doc.rangeHasMark(from, to, markType);

      if (marks) {
        // 이미 마크가 있으면 제거
        tr.removeMark(from, to, markType);
      } else {
        const newMark = markType.create(attrs);
        const excludes = markType.spec.excludes;

        if (excludes) {
          // excludes: "_" 인 경우 모든 마크 제거
          if (excludes === '_') {
            Object.keys(state.schema.marks).forEach((markName) => {
              const otherMarkType = state.schema.marks[markName];
              if (otherMarkType && otherMarkType !== markType) {
                tr.removeMark(from, to, otherMarkType);
              }
            });
          } else {
            // 특정 마크들 제거
            const excludesList = excludes.split(' ');
            excludesList.forEach((markName) => {
              const otherMarkType = state.schema.marks[markName];
              if (otherMarkType) {
                tr.removeMark(from, to, otherMarkType);
              }
            });
          }
        }

        // 같은 타입의 다른 속성 마크 제거 (fontSize, color 등)
        if (attrs) {
          tr.removeMark(from, to, markType);
        }

        tr.addMark(from, to, newMark);
      }

      dispatch(tr.scrollIntoView());
    }

    return true;
  };
}
```

**적용 근거**:
1. **ProseMirror Guide**: excludes 속성 사용법
2. **TipTap**: 자동 충돌 해결 패턴
3. **Quill**: 정규화된 표현 유지

### 6.2 Code Mark 특별 처리
```typescript
export const code: MarkSpec = {
  excludes: '_', // 모든 다른 마크 제외
  code: true,
  parseDOM: [{ tag: 'code' }],
  toDOM() {
    return ['code', { class: 'inline-code' }, 0];
  },
};
```

**적용 근거**:
1. **TipTap 문서**: "inline code mark excludes any other mark"
2. **일반적인 UX 패턴**: 코드는 일반 텍스트와 구분
3. **ProseMirror Guide**: excludes="_" 와일드카드 사용

### 6.3 Mark Priority 순서
```typescript
const DEFAULT_MARK_PRIORITY: Record<string, number> = {
  link: 100,          // 외부 레이어
  fontSize: 90,
  fontFamily: 85,
  color: 80,
  backgroundColor: 75,
  bold: 50,
  italic: 45,
  underline: 40,
  strikethrough: 35,
  code: 10,           // 내부 레이어
};
```

**적용 근거**:
1. **TipTap**: Priority 시스템 사용
2. **DOM 구조**: 링크가 포맷팅을 감싸는 구조
3. **UX 일관성**: 예측 가능한 렌더링 순서

## 7. 충돌 해결 패턴 요약

### 7.1 발견된 주요 패턴

| 에디터 | 충돌 해결 전략 | 적용 여부 |
|--------|---------------|-----------|
| ProseMirror | excludes, canonicalization | ✅ 적용 |
| TipTap | priority, excludes | ✅ 적용 |
| Quill | canonical delta, exclusive formats | ✅ 개념 적용 |
| Google Docs | change log, custom rendering | ❌ 미적용 (복잡도) |
| Yjs/CRDT | automatic merging | ❌ 미적용 (범위 외) |

### 7.2 구현된 충돌 방지 메커니즘

1. **Schema Level**
   - Mark excludes 정의
   - Mark ordering (스키마 순서)

2. **Command Level**
   - toggleMarkSafe: 자동 충돌 해결
   - excludes 체크 및 자동 제거
   - 같은 타입 마크 교체

3. **Document Level**
   - Canonicalization (ProseMirror 기본)
   - 인접 노드 자동 병합

## 8. 참고 문헌

### 공식 문서
1. [ProseMirror Guide](https://prosemirror.net/docs/guide/) - Schema, Marks, Document Structure
2. [ProseMirror Reference](https://prosemirror.net/docs/ref/) - MarkSpec API
3. [TipTap Documentation](https://tiptap.dev/docs/editor/core-concepts/schema) - Priority & Excludes
4. [Quill Delta Format](https://quilljs.com/docs/delta) - Canonicalization

### 연구 자료
5. [Peritext: A CRDT for Rich-Text Collaboration](https://www.inkandswitch.com/peritext/)
6. [Automerge 2.2: Rich Text](https://automerge.org/blog/2024/04/06/richtext/)
7. [Yjs CRDT](https://discuss.yjs.dev/) - Handling unknown nodes/marks

### 커뮤니티
8. [ProseMirror Discuss Forum](https://discuss.prosemirror.net/)
   - Mark ranking discussion
   - Overlapping marks issues
   - Copy-pasting marks

### 블로그 & 아티클
9. [Google Drive Blog: Conflict Resolution](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs_22.html)
10. [Building Rich-text Experiences with ProseMirror](https://brunoscheufler.com/blog/2024-02-11-building-outstanding-rich-text-experiences-with-prosemirror-and-react)

## 9. 결론

이 에디터의 충돌 방지 시스템은:

1. **ProseMirror 공식 문서**의 excludes, canonicalization 개념을 직접 구현
2. **TipTap**의 priority 시스템을 참고하여 마크 순서 결정
3. **Quill**의 정규화 원칙을 개념적으로 적용
4. **엔터프라이즈 에디터**들의 일반적인 패턴 (코드 마크 특별 처리 등) 반영
5. **ProseMirror Discuss Forum**의 실제 사용 사례 및 이슈 참고

모든 설계 결정은 공식 문서와 실제 프로덕션 에디터들의 검증된 패턴에 근거하여 이루어졌습니다.
