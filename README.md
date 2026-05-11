# bunjang-search-tool

## 상태

현재 레포지토리는 **개념 설계 문서만** 포함한다.  
초기 목표는 LangChain 에이전트가 호출할 수 있는 **번개장터 검색 네이티브 도구**의 구조를 먼저 고정하는 것이다.

---

## 1. 목표

이 프로젝트는 상위 LangChain 에이전트가 사용하는 단일 도구로 동작한다.

도구의 역할은 다음과 같다.

1. 번개장터에서 검색 결과를 대량 수집한다.
2. 각 매물의 상세 본문과 메타데이터를 확보한다.
3. 평가 가능한 크기의 청크로 데이터를 분할한다.
4. 각 청크를 LLM에 병렬 평가시켜 의미 있는 매물만 추린다.
5. 청크별 후보를 병합해 최종 상위 후보 ID를 반환한다.

즉, 상위 에이전트는 사용자와 대화하고 이 도구를 호출하기만 하며, 실제 대량 검색·정제·평가는 이 도구 내부에서 수행한다.

---

## 2. 비목표

초기 버전에서 하지 않는 것:

- 찜 추가
- 채팅 시작/전송
- 구매 흐름 자동화
- 내부 LLM의 에이전트 루프
- 내부 LLM의 툴 사용
- 자유 탐색형 장문 응답

초기 버전의 내부 LLM은 **에이전트가 아니라 구조화된 평가기**로만 사용한다.

---

## 3. 전체 구조

```text
LangChain Agent
  -> searchBunjangListingsForAgent(toolRequest)
      -> collectBunjangListingsForEvaluation()
      -> filterOutDeterministicNoiseListings()
      -> buildListingEvaluationChunks()
      -> scoreListingEvaluationChunk()
      -> validateChunkSelectionResult()
      -> reduceSelectedListingsAcrossChunks()
      -> 최종 후보 반환
```

핵심 원칙:

- 상위에는 에이전트 1개만 둔다.
- 내부 평가는 다회 LLM 호출이지만 에이전트화하지 않는다.
- 청크별 평가는 독립적이고 재시도 가능해야 한다.
- 최종 결과는 가능한 한 작고 검증 가능해야 한다.

---

## 4. 외부 도구 계약

상위 LangChain 에이전트는 아래와 같은 단일 도구 인터페이스를 사용한다.

이제 이 프로젝트는 CLI 플래그 인터페이스가 아니라 **LangChain 네이티브 구조화 도구**를 목표로 하므로, 필드 이름도 축약형보다 의도가 드러나는 이름을 우선 사용한다.

### 입력 초안

```ts
type SearchBunjangListingsForAgentToolInput = {
  searchQueryText: string;
  minimumPriceKrw?: number;
  maximumPriceKrw?: number;
  searchSortOrder?: "score" | "date" | "price_asc" | "price_desc";
  searchStartPageNumber?: number;
  searchPageCount?: number;
  maximumListingsToCollect?: number;
  finalCandidateCount?: number;
  rankingIntentDescription?: string;
  excludedKeywordPhrases?: string[];
  enableDebugArtifacts?: boolean;
};
```

필드 의미:

- `searchQueryText`: 사용자가 찾고 싶은 실제 검색어
- `minimumPriceKrw` / `maximumPriceKrw`: 가격 범위 제한
- `searchSortOrder`: 번개장터 검색 정렬 기준
- `searchStartPageNumber`: 검색 시작 페이지
- `searchPageCount`: 몇 페이지를 수집할지
- `maximumListingsToCollect`: 전체 수집 상한
- `finalCandidateCount`: 최종적으로 상위 몇 개를 돌려줄지
- `rankingIntentDescription`: 예: `실매물 위주`, `가성비 우선`, `미개봉 선호`
- `excludedKeywordPhrases`: 예: `교환`, `삽니다`, `케이스`
- `enableDebugArtifacts`: 디버그용 청크/응답 저장 여부

### 출력 초안

```ts
type SearchBunjangListingsForAgentToolResult = {
  selectedListingIds: string[];
  selectedListings: Array<{
    listingId: string;
    listingTitle: string;
    listingPriceKrw: number | null;
    listingUrl: string;
    combinedRelevanceScore: number;
    selectionReasonSummary: string;
    riskFlagSummaries: string[];
  }>;
  pipelineExecutionStats: {
    totalListingsCollected: number;
    totalListingsAfterDeterministicFiltering: number;
    totalEvaluationChunks: number;
    totalLlmEvaluationCalls: number;
    totalChunkLevelSelections: number;
  };
  pipelineWarnings?: string[];
};
```

상위 에이전트는 보통 `selectedListingIds`와 `selectedListings`만 사용해 최종 사용자 응답을 만든다.

---

## 5. 내부 파이프라인

### 5.1 수집기

`bunjang-cli` 기반으로 검색과 상세 수집을 수행한다.

기본 방향:

- 검색 결과만 가져오지 않고 가능한 한 초기에 상세 본문까지 확보한다.
- `with-detail` 수준의 데이터를 내부 평가 입력으로 사용한다.
- 검색/상세 수집 실패는 가능한 한 개별 listing 단위로 격리한다.

### 5.2 사전 필터

LLM 호출 전, 규칙 기반으로 노이즈를 먼저 제거한다.

예시:

- `교환`
- `삽니다`
- `케이스`
- `필름`
- `부품`
- `액세서리`
- 잘못된 모델명 혼입
- 중복 ID
- 명백한 가격 이상치

원칙:

- 싸고 명확한 판단은 코드로 먼저 한다.
- LLM은 규칙으로 걸러지지 않는 애매한 후보 판단에 집중시킨다.

### 5.3 청크 생성기

수집된 listing들을 평가 가능한 크기의 청크로 분할한다.

원칙:

- 저장용 청크와 평가용 청크는 분리해서 생각한다.
- 평가용 청크는 너무 크게 잡지 않는다.
- 각 청크는 독립적으로 재평가 가능해야 한다.

초기 권장 방향:

- 평가용 청크 목표 크기: 대략 12k~20k 토큰 수준
- 청크당 여러 listing을 포함하되, 응답 파싱 안정성을 우선한다.

### 5.4 청크별 LLM 평가기

각 청크는 동일한 시스템 프롬프트와 동일한 출력 스키마로 평가한다.

평가기 역할:

- 의미 있는 매물 ID만 선택
- 각 후보에 `combinedRelevanceScore` 부여
- 짧은 `selectionReasonSummary` 제공
- `riskFlagSummaries` 기록

평가기 제약:

- 입력에 없는 `listingId`를 생성하면 안 됨
- JSON만 출력해야 함
- 추측 금지
- 입력 텍스트 기반 근거만 사용

### 5.5 결과 검증기

LLM 응답은 그대로 믿지 않고 검증한다.

검증 항목:

- JSON 파싱 성공 여부
- 필수 필드 존재 여부
- `combinedRelevanceScore` 범위 정상 여부
- 선택한 `listingId`가 실제 청크 내부에 존재하는지
- `selectionReasonSummary` / `riskFlagSummaries` 형식 이상 여부

실패 시 정책:

- 1회 재시도
- 재시도 실패 시 해당 청크 경고 처리
- 전체 후보 수가 충분하면 계속 진행
- 전체 후보 수가 부족하면 전체 실패로 올림

### 5.6 병합기

청크별 후보를 합쳐 최종 결과를 만든다.

처리:

- ID 기준 dedupe
- 점수 기준 정렬
- `finalCandidateCount`만큼 추출
- 필요 시 최소 점수 threshold 적용

병합기의 책임:

- LLM이 고른 후보를 다시 한 번 기계적으로 정리
- 상위 에이전트가 바로 사용할 수 있는 작은 응답으로 축약

---

## 6. 내부 LLM 출력 형식 초안

청크별 평가 출력 예시:

```json
{
  "selectedListings": [
    {
      "listingId": "396049093",
      "combinedRelevanceScore": 0.91,
      "selectionReasonSummary": "본체 매물이며 설명이 구체적이고 액세서리/교환글이 아님",
      "riskFlagSummaries": ["배터리 상태 미기재"]
    }
  ]
}
```

이 형식의 목적:

- 상위 reducer가 쉽게 병합할 수 있게 함
- 자유서술형 결과를 줄임
- 추후 스키마 검증과 재시도를 단순화함

---

## 7. 실행 모드

### 기본 모드

- 메모리 내 처리
- 중간 파일 저장 없이 빠르게 평가

### 디버그 모드

- 청크 파일 저장
- 청크별 LLM 응답 저장
- 최종 병합 결과 저장

디버그 모드는 재현성과 튜닝을 위해 사용한다.

---

## 8. 제안 모듈 구조

```text
src/
  tools/
    search-bunjang-listings-for-agent-tool.ts

  bunjang/
    collect-bunjang-listings-for-evaluation.ts
    filter-out-deterministic-noise-listings.ts
    build-listing-evaluation-chunks.ts
    score-listing-evaluation-chunk.ts
    validate-chunk-selection-result.ts
    reduce-selected-listings-across-chunks.ts
    bunjang-tool-prompts.ts
    bunjang-tool-schemas.ts
    bunjang-tool-types.ts

  llm/
    llm-client.ts
    parse-structured-llm-output.ts
```

설계 원칙:

- 도구 외부 계약과 내부 파이프라인을 분리한다.
- 수집/평가/병합 로직을 분리해서 테스트 가능하게 만든다.
- LLM 호출부는 별도 모듈로 감싸 교체 가능하게 만든다.

---

## 9. 초기 기본값 제안

- `maximumListingsToCollect`: 100
- `searchPageCount`: 10
- `finalCandidateCount`: 5
- `listingDetailFetchConcurrency`: 5
- `listingEvaluationChunkTokenLimit`: 15000
- `chunkLevelSelectionCount`: 8
- `llmEvaluationConcurrency`: 4

초기에는 100개 규모에서 안정성을 먼저 확인하고, 이후 200~300개로 확장한다.

---

## 10. 향후 단계

### 1단계

- read-only 검색 도구 완성
- 수집 + 사전 필터 + 청크 평가 + 병합

### 2단계

- 디버그 산출물 정리
- 캐시 및 재시도 정책 강화
- 청크 점수 기준 튜닝

### 3단계

- top 후보에 대한 2차 rerank
- query 유형별 평가 rubric 분화
- 이후 필요 시 찜/채팅 같은 side-effect 도구를 별도 추가

---

## 11. 현재 결론

이 프로젝트는 **LangChain 에이전트가 호출하는 단일 번개장터 검색 네이티브 도구**를 만드는 것을 목표로 하며, 내부적으로는 `bunjang-cli` 기반 수집과 구조화된 다회 LLM 평가 파이프라인을 가진다.

핵심은 다음 세 가지다.

1. 상위에는 에이전트 1개만 둔다.
2. 내부 LLM은 에이전트가 아니라 평가기로 사용한다.
3. 최종 결과는 작은 후보 목록으로 환원해 상위 에이전트에 돌려준다.
