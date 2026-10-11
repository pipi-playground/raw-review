# 매칭 엔진 설계 문서 (하위프로젝트)

- 작성일: 2026-10-11
- 상태: 리뷰 대기 (2차 개정 — 리뷰 10개 지적 반영)
- 상위 spec: `2026-10-11-raw-review-design.md` (로드맵 1단계 1a)
- 개발 방식: TDD + DDD

## 1. 목적

임의의 상품 검색어에 대해, 수집된 글에서 **요청한 상품과 정확히 같은 상품 언급만** 골라내고, 그 언급에 붙은 의견만 채택한다. 휴대폰케이스에 묶이지 않고 양말·부츠·장갑·청소기 등 모든 카테고리를 같은 메커니즘으로 처리한다.

입력 둘, 출력 하나.
- 사용자 질의 → 구조화 `SearchSpec`
- 수집된 글(`RawPost`) → 대상 상품에 해당하는 `Opinion[]`

### 1.1 해결하는 문제

1. **상품 일반화**: enum 소재, 숫자/범위 사이즈, 모델 경계 일치, 용도 동의어를 하나의 속성 모델로 표현.
2. **언급 단위 제외·비교**: "면양말은 보풀, 울양말은 따뜻", "17 Pro는 안 맞고 17 투명케이스는 변색"에서 대상 외 언급은 버리고 대상 의견은 살린다.
3. **속성별 매칭 규칙**: 모델=경계 일치, 소재=enum, 사이즈=범위 비교.
4. **카테고리 연결**: "메리노삭스"→양말+소재:울, "라이딩 글러브"→장갑+용도:자전거.

### 1.2 성공 기준

- 비교 글에서 대상 외 상품 의견이 대상 결과에 섞이지 않는다 (제외 언급 false positive = 0 목표).
- 제외 변형이 같은 글의 대상 의견을 통째로 버리지 않는다.
- 카테고리군(휴대폰/액세서리, 의류/신발, 스포츠용품, 생활가전, 기타) × 상황(정상·유사·제외·비교·애매·광고) 골든셋에서 **언급 매칭 precision ≥ 0.95**.
- 룰 매처(EXACT/ENUM/NUMERIC/RANGE/SYNONYM)는 LLM 없이 결정적이며 단위테스트로 전부 덮인다.

### 1.3 범위 밖

- 수집(크롤링) — 2단계. 이 엔진은 `RawPost`를 입력으로만 받는다.
- Laya 등 결정 모델 — 향후. 지금은 생성형 LLM + 룰.
- 실제 라벨 100건 LLM precision/recall 게이트 — 수집 후 2단계.

## 2. 전역 제약

- Python 3.12, Pydantic v2.
- 상위 spec의 공유 커널 규칙을 따른다. 경계를 넘는 계약은 `shared/`.
- 룰 매처는 순수 함수, 외부 의존 없음, 결정적.
- 생성형 LLM은 2-tier 포트 뒤에. 질의 이해 = 고tier, 글 추출 = 저tier. 모델명은 설정값.
- **역할 분담 원칙**: LLM은 **추출과 정규화**만 한다 — 질의→`SearchSpec` 파싱, 글→`ProductMention[]` 추출, FREETEXT 값의 canonical 정규화. **매칭 최종 판정(채택/불일치/제외/애매)은 전부 룰**이 한다.
- **match_mode 유효성 규칙**:
  - hard 속성은 `EXACT`/`ENUM`/`NUMERIC`/`RANGE`/`SYNONYM`만 쓸 수 있다.
  - `SUBSTRING`은 **soft 속성에서만** 허용(fallback). hard+SUBSTRING은 금지(아이폰17⊂17Pro, 울⊂울혼방 재발 방지).
  - `FREETEXT`는 **soft 속성 전용**. LLM이 canonical 후보로 변환한 뒤 룰이 판정하며, 변환 불가면 그 속성은 UNCERTAIN.
  - `AttributeDef`/`AttributeConstraint` 생성 시 이 규칙을 Pydantic validator로 강제한다.
- 테스트는 특정 카테고리에 치우치지 않는다. 카테고리군 분포를 지킨다.

## 3. 도메인 모델

### 3.1 속성 정의 (카테고리 독립)

```python
class MatchMode(str, Enum):
    EXACT     # 정확 경계 일치(모델명). 토큰 경계 고려, 부분문자열 금지
    ENUM      # 정규화된 허용값 집합 일치(소재, 형태)
    SYNONYM   # 방향 있는 alias→canonical 일치(용도)
    NUMERIC   # 단일 숫자 비교(+단위 변환)
    RANGE     # 범위 포함 비교(240~260mm)
    SUBSTRING # 느슨한 부분 포함(soft 전용 fallback)
    FREETEXT  # 룰 판정 불가. LLM이 canonical 변환 후 룰 판정(soft 전용)

class AttributeDef(BaseModel):
    name: str                          # "대상기기" "소재" "사이즈" "용도"
    hard: bool
    match_mode: MatchMode
    canonical_values: list[str] = []   # ENUM/EXACT/SYNONYM 허용 canonical
    aliases: dict[str, list[str]] = {} # canonical -> 별칭(방향: 별칭이 canonical로)
    exclusions: list[str] = []         # 이 속성 한정 제외 canonical
    unit: str | None = None            # NUMERIC/RANGE 기준 단위 "mm" "g"
    normalization: str | None = None   # 정규화 규칙 식별자
    # validator: hard면 match_mode ∈ {EXACT,ENUM,NUMERIC,RANGE,SYNONYM}

class AttributeSchema(BaseModel):
    category: str
    category_aliases: list[str] = []   # 양말: [삭스, socks]
    attributes: list[AttributeDef]
```

### 3.2 검색 요구 (SearchSpec)

```python
class AttributeConstraint(BaseModel):
    name: str
    hard: bool
    match_mode: MatchMode
    wanted: list[str] = []             # 원하는 canonical
    range: tuple[float, float] | None = None
    unit: str | None = None
    aliases: dict[str, list[str]] = {}
    exclusions: list[str] = []         # 속성 단위 제외 canonical
    # validator: AttributeDef와 동일한 hard/mode 규칙

class SearchSpec(BaseModel):
    category: str
    constraints: list[AttributeConstraint]

class AmbiguousQuery(BaseModel):
    query: str
    options: list[str]
```

전역 제외어 없음. 제외는 항상 속성 단위 canonical.

### 3.3 상품 언급 (ProductMention)

LLM이 글에서 추출. 한 글에 여러 언급 가능. **브랜드 1개당 언급 1개**(추출기가 브랜드별로 쪼갠다).

```python
class MentionRole(str, Enum):
    USED               # 글쓴이가 실제 사용/구매한 제품
    COMPARED           # 비교 대상으로 언급
    REJECTED_ALT       # 고려했다 뺀 대안("17 Pro는 안 맞았다")
    MENTIONED          # 단순 언급

class ExtractedAttribute(BaseModel):
    name: str
    raw_value: str                     # 원문 표현 "240", "240~260mm", "1.5 kg"
    normalized_value: str | None = None# canonical(ENUM/SYNONYM/FREETEXT 변환 결과)
    numeric_value: float | None = None # NUMERIC
    lower: float | None = None         # RANGE 하한
    upper: float | None = None         # RANGE 상한
    unit: str | None = None

class MentionOpinion(BaseModel):
    polarity: Polarity                 # PRO | CON
    aspect: str
    quote: str                         # 원문 조각(120자 제한)
    source_part: str                   # post | comment

class ProductMention(BaseModel):
    raw: str                           # 이 언급의 근거 원문 조각
    role: MentionRole
    brand: str | None = None
    attributes: list[ExtractedAttribute] = []
    opinions: list[MentionOpinion] = []
```

### 3.4 매칭 결과

```python
class MatchOutcome(str, Enum):
    MATCHED            # 대상과 일치 → 채택
    REJECTED           # 속성 단위 제외에 걸림
    MISMATCH           # 하드 불일치
    UNCERTAIN          # 하드 속성 추출 누락 등 → 채택 안 함

class MentionMatch(BaseModel):
    mention: ProductMention
    outcome: MatchOutcome
    matched_attributes: list[str] = []
    reason: str | None = None
```

## 4. 컴포넌트

### 4.1 CategoryResolver (질의 → SearchSpec)

- 고tier 생성형 LLM 포트(`SpecParser`).
- `category_aliases`·상품명 분해 사전으로 질의를 카테고리에 연결("메리노삭스"→양말+소재:울).
- `SchemaCache`에 카테고리 스키마가 있으면 파서에 힌트로 넘겨 일관된 SearchSpec 생성. 없으면 LLM이 스키마 생성 후 캐시.
- 신뢰도 < 임계값이면 `AmbiguousQuery`로 되묻는다.

### 4.2 MatchMode 매처 (결정적 룰)

mode별 순수 함수 `match(extracted: ExtractedAttribute, c: AttributeConstraint) -> bool`. 호출 전에 `c.exclusions` 검사는 상위(MentionMatcher)가 한다.

- **EXACT**: 정규화 후 **토큰 경계 일치**. `아이폰17`이 `아이폰17pro`에 부분 포함돼도 매칭 아님. 경계는 한글/영문/숫자 전이와 공백·구분자로 토큰을 끊어 비교.
- **ENUM**: `normalized_value`가 `wanted`(canonical 집합)에 속하는지.
- **SYNONYM**: **방향 있는 alias→canonical**. 추출 단계에서 LLM이 이미 canonical(`normalized_value`)을 넣음. 룰은 canonical 동등 비교. "라이딩 글러브"→용도:자전거, "오토바이 라이딩 장갑"→용도:오토바이. "라이딩" substring으로 자전거 오인 안 함.
- **NUMERIC**: 단위 변환 후 `numeric_value`와 `wanted` 숫자 비교(허용 오차 설정값).
- **RANGE**: `wanted`가 범위고 제시가 단일이면 포함 검사. 제시가 범위(`lower`/`upper`)면 교집합 여부.
- **SUBSTRING**: 느슨한 포함. **soft 전용**. precision 게이트에서 사용분을 별도 측정.
- **FREETEXT**: 룰 직접 판정 불가. `normalized_value`가 있으면 ENUM처럼 비교, 없으면 UNCERTAIN. **soft 전용.**

### 4.3 MentionExtractor (글 → ProductMention[])

- 저tier 생성형 LLM 포트. 본문·댓글 + SearchSpec(어떤 속성·canonical을 뽑을지 힌트)을 넣는다.
- JSON schema 구조화 출력으로 `ProductMention[]`. FREETEXT/SYNONYM 속성은 canonical로 정규화해서 반환하도록 프롬프트.
- 비교 글은 언급을 쪼개고 각 언급에 `role`을 부여하도록 프롬프트("실제 쓴 것"=USED, "비교"=COMPARED, "고려했다 뺌"=REJECTED_ALT).
- **recall 우선**: 후보 언급은 일단 뽑는다.
- 앞단 **CoarseGate**(순수 룰): 글에 카테고리/속성 키워드가 하나도 없으면 추출 호출 건너뛰고 빈 결과(비용 1차 방어).
- **추출 누락 처리**: CoarseGate 통과했는데 빈 배열이면 1회 재시도. 그래도 비면 그 글을 `UNANALYZED`로 기록(silently drop 금지, 통계 노출). schema 검증 실패도 1회 재시도 후 `UNANALYZED`.

### 4.4 MentionMatcher (언급 → 채택 여부, 결정적 룰)

각 `ProductMention`에 대해:

1. **role 필터**: `role ∈ {USED}`만 채택 후보. `COMPARED`/`REJECTED_ALT`/`MENTIONED`은 채택하지 않는다(비교·반론 글의 대안 제품 의견이 섞이지 않음). → outcome은 MISMATCH가 아니라 역할상 비대상이므로 집계에서 분리.
2. **속성 단위 제외**: USED 언급의 어떤 속성 값이 그 속성의 `exclusions`에 걸리면 `REJECTED`. 제외는 **실제 사용 제품(USED)에만** 적용되므로, "17 Pro는 안 맞고(REJECTED_ALT) 17 썼다(USED)"에서 17 Pro는 제외 트리거가 아니다.
3. **하드 매칭**: 모든 하드 constraint에 대해 해당 속성의 추출값을 mode별 매처로 비교.
   - 추출값이 없는 하드 속성이 있으면 → `UNCERTAIN`(채택 안 함). precision 우선.
   - 하드 매처가 하나라도 불일치면 `MISMATCH`.
   - 전부 통과하고 제외 없음 → `MATCHED`.
4. 채택(`MATCHED`) 언급의 `opinions`만 하류로 넘긴다.

### 4.5 AspectNormalizer

- aspect 동의어를 카테고리 단위 canonical로 정규화("황변"→"변색").

### 4.6 Opinion 변환 계약

매칭 엔진 출력을 상위 분석의 `Opinion[]`으로 잇는다.

```
RawPost
  → MentionExtractor → ProductMention[]
  → MentionMatcher   → MentionMatch[] (outcome별)
  → MATCHED 의 MentionOpinion
  → to_opinions()    → Opinion[]
  → (1b ReportBuilder)
```

`to_opinions(match: MentionMatch, post: RawPost) -> list[Opinion]`:
- `MATCHED`인 언급의 `opinions`만 변환.
- `Opinion.brand = mention.brand`(None이면 리포트에서 "브랜드 미상" 버킷).
- `Opinion.quote = aspect_normalized MentionOpinion.quote`(120자 제한), `aspect = AspectNormalizer.canonical(...)`.
- `source_part`를 보존하고, 댓글 의견도 글 `url`을 출처로 유지.
- 한 언급은 브랜드 1개이므로 다중 브랜드 혼입 없음(추출기가 보장).

## 5. 데이터 흐름

```
질의 ─(고tier LLM)→ SearchSpec(구조화)
RawPost ─(CoarseGate 룰)─ 키워드 전무 → 버림
         └ 통과 ─(저tier LLM)→ ProductMention[](role 포함)
                 (룰 MentionMatcher: role 필터 → 속성단위 제외 → 하드 매칭)
                 └ MATCHED 의 Opinion[] ─→ (1b 리포트)
         빈 추출/재시도 실패 → UNANALYZED(통계 노출)
```

## 6. 포트와 어댑터

- `SpecParser`(고tier): `parse(query, known_schema?) -> ParseResult`
- `MentionExtractor`(저tier): `extract(post, spec) -> ExtractResult{mentions, status}` (status: OK | EMPTY | UNANALYZED)
- `SchemaCache`: `get/put/classify_category`
- 어댑터: `OpenAiSpecParser`, `OpenAiMentionExtractor`(모델명 설정값), 테스트용 `FakeSpecParser`/`FakeMentionExtractor`.
- 매처(`MentionMatcher`, MatchMode 구현)는 포트가 아니라 순수 도메인.

## 7. 테스트 전략

- **MatchMode 매처 단위테스트**: mode별 일치/불일치/경계/제외. EXACT 부분문자열 비매칭, RANGE 경계·교집합, NUMERIC 단위 변환, SYNONYM 방향(오토바이 vs 자전거), SUBSTRING hard 금지 validator.
- **MentionMatcher 테스트**: role 필터(USED만 채택), 제외는 USED에만 적용, 하드 누락→UNCERTAIN, 비교 글 복수 언급 분리.
- **CategoryResolver 테스트**: 캐시 히트/미스, 상품명 분해, 별칭 연결, 애매 되묻기.
- **변환 계약 테스트**: MATCHED→Opinion, 브랜드 None 버킷, 댓글 출처 보존.
- **카테고리 분산 테스트군**(휴대폰 치우침 제거): 최소
  - 울양말/면양말/메리노울 (소재 enum, 언급 단위 제외·비교)
  - 워커부츠/첼시부츠/등산화 (종류 enum)
  - 자전거용 장갑/오토바이 장갑/작업용 장갑 (용도 synonym 방향)
  - 로봇청소기/무선청소기 (종류 enum)
  - 숫자·범위 상품 (사이즈 240~260mm 등)
  - 한 글에서 여러 상품 비교 (role)
  - 동의어·별칭만으로 지칭 (라이딩 글러브, 메리노삭스)
- **평가 지표(함수로 명시)**: eval 모듈에 다음을 구현한다.
  - `mention_precision`, `mention_recall`
  - `excluded_mention_false_positive`(제외/비대상 언급을 채택한 수)
  - `target_mention_false_negative`(대상 USED 언급을 놓친 수)
  - `by_category(...)` 카테고리별 집계
  - `comparison_only(...)` 비교 글 전용 집계
- **골든셋(룰 계층)**: 카테고리군 × 상황 분포 손 라벨 케이스. 룰 매처를 라벨에 대고 돌려 precision 측정. 100건+ 실제 라벨과 LLM precision/recall 게이트는 2단계.

## 8. 확정된 결정 (구현 전 결정 완료)

- **하드 속성 추출 누락** → `UNCERTAIN`, 채택 안 함(precision 우선, recall 손실은 지표로 측정).
- **FREETEXT 최종 판정** → LLM이 canonical로 변환, 룰이 판정. 변환 불가면 UNCERTAIN. FREETEXT는 soft 전용.
- **부정·비교 표현** → `ProductMention.role`로 처리. USED만 채택, 제외는 USED subject에만 적용.
- **NUMERIC/RANGE 단위** → 길이(mm/cm/inch), 무게(g/kg)부터. 확장 가능.
- **SUBSTRING** → soft 전용, hard 금지(validator).
- **SYNONYM** → 방향 있는 alias→canonical, 추출 단계에서 canonical 산출.

## 9. 남은 열린 질문 (구현 중 조정 가능, 계약 불변)

- EXACT 토큰 경계를 전이 기반으로 충분한지, 형태소 분석기가 필요한지 (초기 전이 기반, 부족하면 보강).
- role 분류 정확도가 낮을 때 보완책(초기엔 추출 프롬프트, 지표로 관찰).
- category_aliases·상품명 분해 사전의 초기 적재 범위.
