# 데이터 설계서 (DDD)

> **산출물 MD05** | RETS-MODULE-04 도메인 모델링 및 용어 사전 구축 도구  
> 작성일: 2026-05-07 | 버전: 1.0 | 작성자: MODULE-04 개발팀  
> 참조: `rets_02c_req_schema.json`, `rets_02c_req_schema_explanation.md`  
> 연관 산출물: MD03 SRS (`rets_module_04_03_srs.xlsx`)

---

## 1. 개요

### 1.1 목적

본 문서는 RETS-MODULE-04 모듈 내에서 처리되는 주요 데이터의 구조, 속성, 관계, 제약조건, 입출력 관점의 데이터 항목을 정의하고, 각 데이터 객체와 요구사항 간의 추적 관계를 정리한다.

### 1.2 데이터 객체 구성

본 모듈은 백엔드 없이 브라우저 메모리와 localStorage에서 동작하므로, 모든 데이터는 JavaScript 객체로 관리된다. 데이터 객체는 다음 4개 그룹으로 구분된다.

| 그룹 | 설명 | 연관 요구사항 |
|------|------|--------------|
| DDD-01 | 입력 데이터 (DEV REQ 요구사항) | DEV_FR_001 |
| DDD-02 | 도메인 모델 데이터 (엔터티·관계) | DEV_FR_002, DEV_FR_003 |
| DDD-03 | 용어 사전 데이터 | DEV_FR_004 |
| DDD-04 | 출력 데이터 (내보내기) | DEV_FR_007 |

---

## 2. 데이터 객체 상세 정의

---

### DDD-01. 입력 데이터 — DevRequirement (DEV REQ)

**설명**: RETS 공통 스키마(`rets_02c_req_schema.json`)를 준수하는 개발 요구사항 데이터. JSON 파일로 로드되어 모듈 메모리에 유지된다.

**연관 요구사항**: `DEV_FR_001_01`, `DEV_FR_001_02`, `DEV_FR_005_01`, `DEV_FR_005_02`

#### 2.1.1 DevRequirement 객체 구조

```typescript
interface DevRequirement {
  // ── 공통 필드 (필수) ───────────────────────────────────────────
  req_type: "DEV";                          // 고정값
  req_id: string;                           // 패턴: ^DEV_(FR|NFR)_\d{3}(_\d{2}){0,4}$
  req_version: string;                      // 예: "1.0", "1.2"
  req_level: 1 | 2 | 3 | 4 | 5;           // req_id 계층 깊이와 일치 필수
  req_name: string;                         // 최대 200자
  functional_class: "기능" | "비기능";
  req_category: ReqCategory;
  description: string;
  req_status: ReqStatus;
  created_by: string;
  last_modified_date: string;               // ISO 8601 date (YYYY-MM-DD)

  // ── 공통 필드 (선택) ───────────────────────────────────────────
  domain_area?: DomainArea;
  acceptance_criteria?: string[];
  verification_method?: VerificationMethod;
  metrics?: string[];
  priority?: "MUST" | "SHOULD" | "COULD" | "WONT";
  importance?: "Critical" | "High" | "Medium" | "Low";
  stakeholders?: string[];
  manager?: string;
  reviewer?: string;
  change_history?: ChangeHistoryEntry[];

  // ── DEV 전용 필드 (선택) ────────────────────────────────────────
  io_spec?: IOSpec;
  story_points?: number;                    // 0~100, 피보나치 권장
  estimated_effort?: number;               // Man-Day 단위
  source_rfp_ids?: string[];               // 패턴: ^RFP_\d{3}(_\d{2}){0,4}$
  predecessor_dev_ids?: string[];          // 패턴: ^DEV_(FR|NFR)_\d{3}(_\d{2}){0,4}$
  dev_status?: DevStatus;
  developer?: string;

  // ── 모듈 내부 확장 필드 (DDD-01 전용) ────────────────────────────
  _nonStandardTerms?: NonStandardTermMatch[];  // 비표준 용어 탐지 결과 (런타임)
}
```

#### 2.1.2 Enum 타입 정의

```typescript
type ReqCategory = "기능" | "인터페이스" | "데이터" | "배치" | "성능"
                 | "보안" | "가용성" | "유지보수성" | "운영" | "제약사항" | "기타";

type DomainArea = "UI/UX" | "인증/인가" | "데이터관리" | "통합/인터페이스"
                | "AI/LLM" | "성능" | "보안" | "운영" | "인프라" | "공통" | "기타";

type VerificationMethod = "테스트" | "검사" | "분석" | "시연" | "검토";

type ReqStatus = "Draft" | "InReview" | "Approved" | "Rejected"
               | "Deferred" | "Deprecated";

type DevStatus = "Todo" | "InProgress" | "Done" | "Blocked" | "Skipped";
```

#### 2.1.3 중첩 타입 정의

```typescript
interface ChangeHistoryEntry {
  version: string;           // 패턴: ^\d+\.\d+(\.\d+)?$
  date: string;              // ISO 8601 date
  author: string;
  description: string;
  changed_fields?: string[];
}

interface IOSpec {
  inputs?: string[];
  outputs?: string[];
}

interface NonStandardTermMatch {
  term: string;              // 탐지된 비표준 표현
  standardTerm: string;      // 용어 사전 표준 용어
  position: string;          // 발견 위치 (예: "description")
}
```

#### 2.1.4 계층 규칙 (RETS 공통 스키마 준수)

```
req_level = (req_id 내 SEQ 이후 _XX 구성요소 수) + 1

예시:
  DEV_FR_001           → req_level = 1 (Epic)
  DEV_FR_001_01        → req_level = 2 (Feature)
  DEV_FR_001_01_03     → req_level = 3 (User Story)
  DEV_FR_001_01_03_11  → req_level = 4 (Task)
```

---

### DDD-02. 도메인 모델 데이터 — DomainModel

**설명**: LLM 추출 및 사용자 직접 편집으로 구성되는 도메인 엔터티와 관계 데이터. Mermaid ER 다이어그램 렌더링의 소스가 된다.

**연관 요구사항**: `DEV_FR_002_01~03`, `DEV_FR_003_01~03`, `DEV_FR_006_01~02`

#### 2.2.1 DomainModel 최상위 구조

```typescript
interface DomainModel {
  entities: Entity[];
  relationships: Relationship[];
  metadata: DomainModelMetadata;
}

interface DomainModelMetadata {
  projectName: string;
  createdAt: string;            // ISO 8601 datetime
  lastModifiedAt: string;
  version: string;
  sourceReqIds: string[];       // 분석에 사용된 DEV REQ ID 목록
}
```

#### 2.2.2 Entity 객체

```typescript
interface Entity {
  id: string;                   // UUID 또는 auto-increment ID
  name: string;                 // 엔터티명 (고유, 최대 100자)
  description: string;          // 비즈니스 설명
  attributes: EntityAttribute[];
  sourceType: "llm" | "manual"; // 추출 출처
  status: "candidate" | "confirmed" | "rejected"; // 확정 상태
  linkedGlossaryTermIds: string[];  // 연결된 용어 사전 ID
  createdAt: string;
  updatedAt: string;
}

interface EntityAttribute {
  name: string;                 // 속성명
  type: string;                 // 데이터 타입 (예: "String", "Integer")
  isPrimaryKey: boolean;
  isRequired: boolean;
  description?: string;
}
```

#### 2.2.3 Relationship 객체

```typescript
interface Relationship {
  id: string;                   // UUID
  sourceEntityId: string;       // 소스 엔터티 ID
  targetEntityId: string;       // 타겟 엔터티 ID
  relationType: RelationType;
  label: string;                // 관계 설명 레이블
  sourceCardinality: Cardinality;
  targetCardinality: Cardinality;
  sourceType: "llm" | "manual";
  status: "candidate" | "confirmed" | "rejected";
  createdAt: string;
  updatedAt: string;
}

type RelationType = "association" | "aggregation"
                  | "composition" | "inheritance" | "dependency";

type Cardinality = "one" | "many" | "zero-or-one" | "zero-or-many";
```

#### 2.2.4 Mermaid ER 다이어그램 생성 규칙

```typescript
// Entity → Mermaid erDiagram 변환
function toMermaidER(model: DomainModel): string {
  // 카디널리티 매핑
  const cardMap = {
    "one":          { left: "||", right: "||" },
    "many":         { left: "}|", right: "|{" },
    "zero-or-one":  { left: "|o", right: "o|" },
    "zero-or-many": { left: "}o", right: "o{" },
  };
  // 예시 출력:
  // erDiagram
  //   요구사항 {
  //     string req_id PK
  //     string req_name
  //   }
  //   이해관계자 {
  //     string id PK
  //     string name
  //   }
  //   요구사항 ||--o{ 이해관계자 : "has"
}
```

---

### DDD-03. 용어 사전 데이터 — GlossaryStore

**설명**: 도메인 엔터티의 공식 용어 정의, 동의어, 관련 엔터티를 관리하는 데이터 저장소.

**연관 요구사항**: `DEV_FR_004_01~04`, `DEV_FR_005_01~02`

#### 2.3.1 GlossaryStore 구조

```typescript
interface GlossaryStore {
  terms: GlossaryTerm[];
  metadata: GlossaryMetadata;
}

interface GlossaryMetadata {
  totalTerms: number;
  lastUpdatedAt: string;
  projectName: string;
  version: string;
}
```

#### 2.3.2 GlossaryTerm 객체

```typescript
interface GlossaryTerm {
  id: string;                   // UUID
  term: string;                 // 공식 표준 용어명 (고유, 최대 100자)
  definition: string;           // 공식 정의 (비즈니스 맥락 반영)
  synonyms: string[];           // 동의어 목록
  relatedEntityIds: string[];   // 연결된 Entity ID 목록
  sourceType: "llm" | "manual"; // 정의 출처
  status: "candidate" | "active" | "deprecated";
  note?: string;                // 비고
  conflictsWith?: string[];     // 충돌 중인 term ID 목록
  mergedFrom?: string[];        // 통합된 구 term ID 목록
  createdAt: string;
  updatedAt: string;
}
```

#### 2.3.3 용어 충돌 탐지 로직

```typescript
// 중복·충돌 용어 탐지 알고리즘
function detectConflicts(terms: GlossaryTerm[]): ConflictPair[] {
  // 1. 완전 일치 탐지: term 이름 동일
  // 2. 동의어 중복 탐지: synonyms 배열 내 동일 단어
  // 3. 의미 유사도: LLM 기반 유사도 분석 (선택적)
  // 반환: 충돌 쌍 목록 [(termId_A, termId_B, conflictReason)]
}

interface ConflictPair {
  termIdA: string;
  termIdB: string;
  reason: "exact_match" | "synonym_overlap" | "semantic_similarity";
  similarity?: number;  // 0~1 (semantic_similarity 시)
}
```

---

### DDD-04. 출력 데이터 — ExportOutput

**설명**: 용어 사전 및 도메인 모델 다이어그램의 내보내기 출력 형식 정의.

**연관 요구사항**: `DEV_FR_007_01`, `DEV_FR_007_02`

#### 2.4.1 용어 사전 Markdown 출력 형식

```markdown
# 용어 사전 (Glossary)
> 프로젝트: {projectName}  
> 생성일: {date}  
> 총 용어 수: {count}

---

## {term}
**정의**: {definition}  
**동의어**: {synonyms.join(", ")}  
**관련 엔터티**: {relatedEntities.join(", ")}  
**비고**: {note}

---
```

#### 2.4.2 용어 사전 CSV 출력 형식

```
term,definition,synonyms,related_entities,note,status,created_at
요구사항,"시스템이 수행해야 하는 기능 또는 제약","소프트웨어 요구사항,기능 요구사항","요구사항 문서,기능 명세",,active,2026-05-07
```

#### 2.4.3 도메인 모델 다이어그램 HTML 출력 형식

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>도메인 모델 다이어그램 - {projectName}</title>
  <script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
</head>
<body>
  <h1>도메인 모델</h1>
  <p>생성일: {date} | 엔터티: {entityCount}개 | 관계: {relCount}개</p>
  <div class="mermaid">
    erDiagram
    {mermaidERCode}
  </div>
  <script>mermaid.initialize({ startOnLoad: true });</script>
</body>
</html>
```

#### 2.4.4 도메인 모델 다이어그램 Markdown 출력 형식

````markdown
# 도메인 모델 다이어그램
> 프로젝트: {projectName}  
> 생성일: {date}

```mermaid
erDiagram
  {mermaidERCode}
```
````

---

## 3. 모듈 내부 상태 관리 (AppState)

```typescript
interface AppState {
  // 로드된 요구사항 데이터
  requirements: DevRequirement[];
  requirementsLoaded: boolean;

  // 도메인 모델
  domainModel: DomainModel;

  // 용어 사전
  glossaryStore: GlossaryStore;

  // LLM 세션 관리
  llmSession: {
    sessionId: string | null;
    teamId: string;
    isLoading: boolean;
    lastError: string | null;
  };

  // UI 상태
  ui: {
    activeTab: "requirements" | "domain-model" | "glossary" | "review";
    theme: "light" | "dark";
    language: "ko" | "en";
    selectedEntityId: string | null;
    selectedTermId: string | null;
  };

  // LLM 제안 결과 (검토 대기)
  pendingCandidates: {
    entities: Entity[];           // 수락/거부 대기 엔터티
    relationships: Relationship[]; // 수락/거부 대기 관계
    glossaryTerms: GlossaryTerm[]; // 수락/거부 대기 용어
    improvements: ImprovementSuggestion[]; // 완성도 개선 제안
  };
}

interface ImprovementSuggestion {
  id: string;
  type: "missing_entity" | "missing_relationship" | "missing_term" | "incomplete_definition";
  description: string;
  suggestedData: Entity | Relationship | GlossaryTerm | null;
  status: "pending" | "accepted" | "rejected";
}
```

---

## 4. 데이터 흐름

```
[파일 입력]
    ↓
DEV REQ JSON (rets_02c_req_schema)
    ↓
requirements: DevRequirement[]
    ↓ LLM 분석
entities: Entity[] (candidate)
    ↓ 사용자 수락
entities: Entity[] (confirmed)
    ↓ LLM 분석
relationships: Relationship[] (candidate)
    ↓ 사용자 수락
relationships: Relationship[] (confirmed)
    ↓              ↓
DomainModel     glossaryTerms: GlossaryTerm[]
    ↓              ↓
[HTML/MD 출력]  [MD/CSV 출력]
```

---

## 5. 데이터 제약사항 및 유효성 규칙

| 데이터 필드 | 제약사항 | 오류 처리 |
|------------|---------|-----------|
| `req_id` | `^DEV_(FR\|NFR)_\d{3}(_\d{2}){0,4}$` 패턴 | Toast(warning) + 해당 항목 스킵 |
| `req_level` | req_id 계층 깊이와 일치 | Toast(error) 안내 |
| `entity.name` | 고유(unique), 최대 100자, 공백 금지 | 저장 전 유효성 검증, 오류 메시지 표시 |
| `glossaryTerm.term` | 고유(unique), 최대 100자 | 저장 전 중복 확인, 오류 메시지 표시 |
| `relationship` | sourceEntityId ≠ targetEntityId | 동일 엔터티 자기 참조 차단 |
| `LLM JSON 응답` | JSON.parse 후 스키마 검증 | 파싱 오류 시 Toast(error), 원본 텍스트 표시 |

---

## 6. localStorage 저장 구조

```javascript
// localStorage key 구조
const STORAGE_KEYS = {
  APP_STATE:    'rets_m04_state',     // AppState (직렬화)
  DOMAIN_MODEL: 'rets_m04_domain',    // DomainModel
  GLOSSARY:     'rets_m04_glossary',  // GlossaryStore
  SETTINGS:     'rets_m04_settings',  // 테마, 언어, team_id
};

// 설정 데이터 구조
interface UserSettings {
  theme: "light" | "dark";
  language: "ko" | "en";
  teamId: string;
}
```

---

## 7. 요구사항 추적 매핑

| 데이터 객체 | 연관 요구사항 | SRS 참조 |
|------------|--------------|---------|
| DevRequirement | DEV_FR_001_01, DEV_FR_001_02 | FR EPIC 1 |
| DevRequirement._nonStandardTerms | DEV_FR_005_01, DEV_FR_005_02 | FR EPIC 5 |
| Entity | DEV_FR_002_01, DEV_FR_002_02, DEV_FR_002_03 | FR EPIC 2 |
| Relationship | DEV_FR_003_01, DEV_FR_003_02 | FR EPIC 3 |
| DomainModel (Mermaid 출력) | DEV_FR_003_03 | FR EPIC 3 |
| GlossaryTerm | DEV_FR_004_01, DEV_FR_004_02 | FR EPIC 4 |
| ConflictPair | DEV_FR_004_03 | FR EPIC 4 |
| GlossaryStore (검색) | DEV_FR_004_04 | FR EPIC 4 |
| ImprovementSuggestion | DEV_FR_006_01, DEV_FR_006_02 | FR EPIC 6 |
| ExportOutput (용어 사전) | DEV_FR_007_01 | FR EPIC 7 |
| ExportOutput (다이어그램) | DEV_FR_007_02 | FR EPIC 7 |
| AppState.llmSession | DEV_NFR_001_03 | NFR EPIC 1 |
| AppState.ui.theme | DEV_NFR_003_02 | NFR EPIC 3 |
| AppState.ui.language | DEV_NFR_004_01 | NFR EPIC 4 |

---

## 8. 변경 이력

| 버전 | 일자 | 작성자 | 내용 |
|------|------|--------|------|
| 1.0 | 2026-05-07 | MODULE-04 팀 | 최초 작성 |
