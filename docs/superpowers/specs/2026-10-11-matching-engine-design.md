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
    category_aliases: list[str] = []   # 캐시 보존·분해용. 파서가 채운다
    constraints: list[AttributeConstraint]

class AmbiguousQuery(BaseModel):
    query: str
    options: list[str]
```

전역 제외어 없음. 제외는 항상 속성 단위 canonical.

**모델 단계 검증(validator)**:
- `AttributeConstraint`: `match_mode == NUMERIC`이면 `wanted`의 모든 값이 float로 변환 가능해야 한다. `match_mode == RANGE`이면 `range`가 있어야 하고 `range[0] <= range[1]`.
- `AttributeDef`/`AttributeConstraint`: hard면 mode ∈ {EXACT,ENUM,NUMERIC,RANGE,SYNONYM}.

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

공식 반환은 **삼상태** `AttrMatch ∈ {MATCH, NOMATCH, UNCERTAIN}`. 디스패치 함수가 계약이다.

```python
class AttrMatch(str, Enum):
    MATCH; NOMATCH; UNCERTAIN

def match_attribute(e: ExtractedAttribute | None, c: AttributeConstraint) -> AttrMatch
```

- 추출값 `e is None` → `UNCERTAIN`(하드 속성 누락).
- 모든 문자열 비교(ENUM/SYNONYM/EXACT 토큰화/제외)는 **공통 `shared.text.normalize`** 를 거친다. "메리 노"=="메리노", 공백·대소문자 무시.
- **EXACT**: 토큰 시퀀스 **완전 일치**(`token_seq_equal`). `아이폰17`이 `아이폰17 Pro`와 일치 아님(부분문자열 금지). 경계는 한글/영문/숫자 전이·구분자로 끊는다.
- **ENUM**: `normalized_value`(canonical)를 `wanted ∪ aliases`의 canonical 집합과 비교. 전부 normalize 적용.
- **SYNONYM**: canonical 동등. 추출이 `normalized_value`에 canonical을 넣음. `normalized_value is None`(LLM이 변환 못 함) → **`UNCERTAIN`**(실제 불일치와 구분). "오토바이 라이딩 장갑"→용도:오토바이 → wanted 자전거와 NOMATCH, "라이딩" substring 오인 없음.
- **NUMERIC**: `c.unit`과 `e.unit`이 둘 다 있어야 변환 후 비교. **`e.unit is None`이면 `UNCERTAIN`**(단위 없는 숫자를 target unit으로 가정하지 않음). 변환 후 `numeric_value`를 `wanted`(float)와 허용 오차로 비교.
- **RANGE**: `c.range=(lo,hi)`. 제시가 단일(`numeric_value`)이면 `lo≤v≤hi`. 제시가 범위(`lower`/`upper`)면 **교집합** `not (upper<lo or lower>hi)`. 단위 없으면 `UNCERTAIN`.
- **SUBSTRING**: 느슨한 포함. **soft 전용**. precision 게이트에서 사용분 별도 측정.
- **FREETEXT**: `normalized_value` 있으면 ENUM처럼, 없으면 `UNCERTAIN`. **soft 전용.**

제외 판정 `exclusion_hit(e, c)`도 **canonical+normalize**를 적용한다. `c.exclusions`의 각 값을 alias로 canonical화하고 normalize한 뒤 토큰 subsequence로 비교한다. 예: 제외 "합성섬유"의 alias "polyester" → "polyester 양말" 제외 성립.

### 4.3 MentionExtractor (글 → ProductMention[])

- 저tier 생성형 LLM 포트. 본문·댓글 + SearchSpec(어떤 속성·canonical을 뽑을지 힌트)을 넣는다.
- JSON schema 구조화 출력으로 `ProductMention[]`. FREETEXT/SYNONYM 속성은 canonical로 정규화해서 반환하도록 프롬프트.
- 비교 글은 언급을 쪼개고 각 언급에 `role`을 부여하도록 프롬프트("실제 쓴 것"=USED, "비교"=COMPARED, "고려했다 뺌"=REJECTED_ALT).
- **recall 우선**: 후보 언급은 일단 뽑는다. 역할이 애매하면 `MENTIONED`(→채택 안 됨), 날조 금지.
- 각 언급의 `raw`(근거 원문 조각)는 **필수**(role·속성의 근거). 비면 모델 검증 실패로 본다.
- 앞단 **CoarseGate**(순수 룰): 글에 카테고리/속성 키워드가 하나도 없으면 추출 호출 건너뛰고 `EMPTY`(비용 1차 방어).
- **EMPTY vs UNANALYZED 구분**(둘을 합치지 않는다):
  - 정상 응답인데 언급이 없음 → `EMPTY`("관련 상품 언급 없음"). 재시도하지 않는다.
  - 예외(timeout·rate limit·일시적 5xx·빈 content·schema 검증 실패) → 백오프로 재시도(횟수·간격 설정값), 끝내 실패하면 `UNANALYZED`("분석 못 함").
  - 두 상태는 통계에서 따로 집계한다("후기 없음"과 "분석 실패"는 다른 의미).

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

`to_opinions(matches: list[MentionMatch], post: RawPost, normalizer) -> list[Opinion]`:
- `MATCHED`인 언급의 `opinions`만 변환.
- `Opinion.brand = mention.brand`(None이면 리포트에서 "브랜드 미상" 버킷).
- `aspect = AspectNormalizer.canonical(...)`, `quote` 120자 제한.
- **인용 원문 검증**: LLM이 돌려준 `quote`가 실제 원문에 있는지 확인한다. `source_part=post`면 `normalize(제목+본문)`에 `normalize(quote)`가 포함돼야 하고, `source_part=comment`면 어느 댓글에 포함돼야 한다. 없으면(날조·변형) 그 의견을 버린다(통계에 dropped_quote로 집계).
- `source_part`를 보존하고, 댓글 의견도 글 `url`을 출처로 유지.
- 한 언급은 브랜드 1개이므로 다중 브랜드 혼입 없음(추출기가 보장).

### 4.7 분석 오케스트레이션 (글 1건 파이프라인)

매칭 엔진은 **글 1건을 받아 Opinion[]까지 조립하는 application service**를 제공한다. 전체 흐름(여러 글·리포트)은 1b가 조립한다.

```python
class PostAnalysis(BaseModel):
    url: str
    source: str
    posted_at: datetime | None
    likes: int
    status: ExtractStatus      # OK | EMPTY | UNANALYZED
    opinions: list[Opinion]
    ad_rejected: bool = False  # (광고 판별은 1b/상위에서. 1a는 매칭까지)

async def analyze_post(post, spec, extractor, normalizer) -> PostAnalysis:
    #  CoarseGate → (통과시) extractor.extract → MentionMatcher.match_post
    #  → to_opinions. status는 extract 결과를 따른다.
```

- CoarseGate 탈락 또는 extract `EMPTY` → `opinions=[]`, status 그대로.
- `UNANALYZED` → `opinions=[]`, status=UNANALYZED(통계 노출).
- 이 서비스가 "CategoryResolver 다음, ReportBuilder 앞"의 글 단위 조립 지점이다. 검색 전체(질의 해석 → 여러 글 → 리포트)는 1b의 `run_search`가 조립하고, 워커(3단계)가 이를 호출한다.

## 5. 데이터 흐름

```
질의 ─(고tier LLM)→ SearchSpec(구조화)
RawPost ─(CoarseGate 룰)─ 키워드 전무 → 버림
         └ 통과 ─(저tier LLM)→ ProductMention[](role 포함)
                 (룰 MentionMatcher: role 필터 → 속성단위 제외 → 하드 매칭)
                 └ MATCHED 의 Opinion[] ─→ (1b 리포트)
         빈 추출/재시도 실패 → UNANALYZED(통계 노출)
```

## 6. 포트와 어댑터 (공식 계약)

- `SpecParser`(고tier): `async parse(query, known_schema?) -> ParseResult`
- `MentionExtractor`(저tier): `async extract(post, spec) -> ExtractResult{mentions, status}`. status enum `ExtractStatus{OK, EMPTY, UNANALYZED}`. **반환 타입은 `ExtractResult`**(list 아님 — 상태를 실어야 EMPTY/UNANALYZED 구분).
- `SchemaCache`: `get(category)/put(schema)/classify_category(query)`
- 매처 계약은 삼상태 `match_attribute(e, c) -> AttrMatch`(§4.2). spec과 plan 모두 이 계약을 쓴다.
- 어댑터: `OpenAiSpecParser`, `OpenAiMentionExtractor`(모델명 설정값), 테스트용 `FakeSpecParser`/`FakeMentionExtractor`.
- 매처·MentionMatcher·CoarseGate·to_opinions·analyze_post는 포트가 아니라 순수 도메인/application.
- **재시도 어댑터 책임**: OpenAI 어댑터는 timeout·rate limit·일시적 5xx·빈 content·schema 검증 실패를 설정값(횟수·백오프)으로 재시도한다. spec parser도 동일. 정상 빈 응답은 재시도 대상 아님(EMPTY).

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
- **인용 검증 테스트**: 원문에 없는 quote는 버려지는지.
- **평가 지표(함수로 명시)**: eval 모듈에 다음을 구현한다.
  - `mention_precision`, `mention_recall`
  - `excluded_mention_false_positive`(제외/비대상 언급을 채택한 수)
  - `target_mention_false_negative`(대상 USED 언급을 놓친 수)
  - `role_accuracy`(라벨 role 대비 예측 role 일치율 — 추출 eval, 2단계 LLM 포함 시 본격화. 1단계는 골든셋에 `expected_role`을 두고 계산 함수만 검증)
  - `by_category(...)`, `by_scenario(...)` 집계, `comparison_only(...)`
- **골든셋 분포 자동 검증**: 각 케이스에 `category_group`과 `scenario`(정상·유사·제외·비교·애매·광고) 필드를 둔다. 테스트가 **scenario별 최소 개수**(각 ≥ 2)와 category_group ≥ 4를 검증한다. 룰 매처를 라벨에 돌려 precision 측정. 100건+ 실제 라벨과 LLM precision/recall·role 게이트는 2단계.

## 8. 확정된 결정 (구현 전 결정 완료)

- **하드 속성 추출 누락** → `UNCERTAIN`, 채택 안 함(precision 우선, recall 손실은 지표로 측정).
- **FREETEXT 최종 판정** → LLM이 canonical로 변환, 룰이 판정. 변환 불가면 UNCERTAIN. FREETEXT는 soft 전용.
- **부정·비교 표현** → `ProductMention.role`로 처리. USED만 채택, 제외는 USED subject에만 적용.
- **NUMERIC/RANGE 단위** → 길이(mm/cm/inch), 무게(g/kg)부터. 확장 가능.
- **SUBSTRING** → soft 전용, hard 금지(validator).
- **SYNONYM** → 방향 있는 alias→canonical, 추출 단계에서 canonical 산출. canonical 없으면 UNCERTAIN.
- **NUMERIC/RANGE 단위 없음** → UNCERTAIN(단위 없는 숫자를 target unit으로 가정 안 함).
- **인용 원문 없음** → 그 의견 버림(dropped_quote 집계).
- **role 애매** → 추출기가 MENTIONED, 채택 안 함. role 근거는 `mention.raw`(필수).
- **반환 계약** → extractor는 `ExtractResult`, 매처는 `AttrMatch` 삼상태. EMPTY≠UNANALYZED.
- **오케스트레이션** → 1a는 `analyze_post`(글 1건), 1b `run_search`(검색 전체), 워커(3단계)가 run_search 호출.

## 9. 남은 열린 질문 (구현 중 조정 가능, 계약 불변)

- EXACT 토큰 경계를 전이 기반으로 충분한지, 형태소 분석기가 필요한지 (초기 전이 기반, 부족하면 보강).
- role 분류 정확도가 낮을 때 보완책(초기엔 추출 프롬프트, 지표로 관찰).
- category_aliases·상품명 분해 사전의 초기 적재 범위.
