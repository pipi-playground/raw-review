# 매칭 엔진 설계 문서 (하위프로젝트)

- 작성일: 2026-10-11
- 상태: 리뷰 대기
- 상위 spec: `2026-10-11-raw-review-design.md` (로드맵 1단계 1a)
- 개발 방식: TDD + DDD

## 1. 목적

임의의 상품 검색어에 대해, 수집된 글에서 **요청한 상품과 정확히 같은 상품 언급만** 골라내고, 그 언급에 붙은 의견만 채택한다. 휴대폰케이스에 묶이지 않고 양말·부츠·장갑·청소기 등 모든 카테고리를 같은 메커니즘으로 처리한다.

이 엔진은 두 입력을 받아 채택된 `Opinion[]`을 낸다.
- 사용자 질의 → 구조화 `SearchSpec`
- 수집된 글(`RawPost`) → 그 글에서 대상 상품에 해당하는 의견

### 1.1 해결하는 문제

1. **상품 일반화**: enum 소재, 숫자/범위 사이즈, 모델 경계 일치, 용도 동의어를 하나의 속성 모델로 표현한다.
2. **언급 단위 제외**: "면양말은 보풀, 울양말은 따뜻", "17 Pro는 안 맞고 17 투명케이스는 변색" 같은 비교·반론 글에서 대상 외 언급만 버리고 대상 의견은 살린다.
3. **속성별 매칭 규칙**: 모델=경계 일치, 소재=enum, 사이즈=범위 비교처럼 속성마다 다른 규칙을 적용한다.
4. **카테고리 연결**: "메리노삭스"→양말+소재:울, "라이딩 글러브"→자전거용 장갑.

### 1.2 성공 기준

- 비교 글에서 대상 외 상품의 의견이 대상 결과에 섞이지 않는다.
- 제외 변형(17 Pro 등)이 같은 글의 대상 상품 의견을 통째로 버리게 하지 않는다.
- 카테고리군(휴대폰/액세서리, 의류/신발, 스포츠용품, 생활가전, 기타) × 상황(정상·유사·제외·비교·애매·광고)을 포함한 골든셋에서 언급 매칭 precision ≥ 0.95.
- 룰 매처(EXACT/ENUM/NUMERIC/RANGE)는 LLM 없이 결정적이며 단위테스트로 전부 덮인다.

### 1.3 범위 밖

- 수집(크롤링) — 2단계. 이 엔진은 `RawPost`를 입력으로만 받는다.
- Laya 등 결정 모델 — 향후. 지금은 생성형 LLM + 룰.
- 실제 라벨 100건 LLM precision/recall 게이트 — 수집 후 2단계. 여기서는 룰 매처 + 손 라벨 케이스로 검증.

## 2. 전역 제약

- Python 3.12, Pydantic v2.
- 상위 spec의 공유 커널 규칙을 따른다. 경계를 넘는 계약은 `shared/`에 둔다.
- 룰 매처(`MentionMatcher`, match_mode 구현)는 순수 함수, 외부 의존 없음, 결정적.
- 생성형 LLM은 2-tier 포트 뒤에 둔다. 질의 이해 = 고tier, 글 추출 = 저tier. 모델명은 설정값.
- 매칭 **최종 판정은 룰**이 한다. LLM은 추출(SearchSpec 파싱, ProductMention 추출)만.
- 테스트는 특정 카테고리에 치우치지 않는다. 카테고리군 분포를 지킨다.

## 3. 도메인 모델

### 3.1 속성 정의 (카테고리 독립)

```python
class MatchMode(str, Enum):
    EXACT     # 정확 경계 일치 (모델명). 토큰 경계 고려, 부분문자열 금지
    ENUM      # 정규화된 허용값 집합 일치 (소재, 형태)
    SYNONYM   # 동의어·복합어 일치 (용도)
    NUMERIC   # 단일 숫자 비교 (+단위 변환)
    RANGE     # 범위 포함 비교 (240~260mm)
    SUBSTRING # 느슨한 부분 포함 (fallback)
    FREETEXT  # 룰로 판정 불가, LLM 보조

class AttributeDef(BaseModel):
    name: str                         # "대상기기" "소재" "사이즈"
    hard: bool
    match_mode: MatchMode
    canonical_values: list[str] = []  # ENUM/EXACT 허용값
    aliases: dict[str, list[str]] = {}# canonical -> 별칭
    exclusions: list[str] = []        # 이 속성 한정 제외값
    unit: str | None = None           # NUMERIC/RANGE: "mm" "g"
    normalization: str | None = None  # 정규화 규칙 식별자

class AttributeSchema(BaseModel):
    category: str
    category_aliases: list[str] = []  # 양말: [삭스, socks]
    attributes: list[AttributeDef]
```

### 3.2 검색 요구 (SearchSpec)

```python
class AttributeConstraint(BaseModel):
    name: str
    hard: bool
    match_mode: MatchMode
    wanted: list[str] = []            # 원하는 값(canonical)
    range: tuple[float, float] | None = None
    unit: str | None = None
    aliases: dict[str, list[str]] = {}
    exclusions: list[str] = []        # 속성 단위 제외

class SearchSpec(BaseModel):
    category: str
    constraints: list[AttributeConstraint]

class AmbiguousQuery(BaseModel):
    query: str
    options: list[str]
```

전역 `exclusion_variants`는 없다. 제외는 항상 속성 단위다.

### 3.3 상품 언급 (ProductMention)

LLM이 글에서 추출하는 단위. 하나의 글에 여러 언급이 있을 수 있다.

```python
class MentionOpinion(BaseModel):
    polarity: Polarity                # PRO | CON
    aspect: str
    quote: str                        # 원문 조각(120자 제한)
    source_part: str                  # post | comment

class ProductMention(BaseModel):
    raw: str                          # 원문 조각(이 언급의 근거)
    brand: str | None = None
    attributes: dict[str, str] = {}   # name -> 추출된 값
    opinions: list[MentionOpinion] = []
```

### 3.4 매칭 결과

```python
class MentionMatch(BaseModel):
    mention: ProductMention
    matched: bool
    matched_attributes: list[str]
    reject_reason: str | None         # 제외 걸림 / 하드 불일치 / 애매
```

## 4. 컴포넌트

### 4.1 CategoryResolver (질의 → SearchSpec)

- 고tier 생성형 LLM 포트(`SpecParser`)를 쓴다.
- `category_aliases`와 상품명 분해 사전으로 질의를 카테고리에 연결한다.
- `SchemaCache`에 카테고리 스키마가 있으면 파서에 힌트로 넘겨 일관된 SearchSpec을 만든다. 없으면 LLM이 스키마 생성 후 캐시.
- 신뢰도 < 임계값이면 `AmbiguousQuery`로 되묻는다.

### 4.2 MatchMode 매처 (결정적 룰)

mode별 순수 함수. 각각 `(제시값, Constraint) -> bool`.

- **EXACT**: 정규화 후 토큰 경계 일치. `아이폰17` 이 `아이폰17pro`에 부분 포함되는 것을 매칭으로 인정하지 않는다. 경계 판정은 숫자/영문/한글 전이와 공백·구분자로 토큰을 끊어 비교한다.
- **ENUM**: 값과 별칭을 canonical로 정규화한 뒤 집합 일치.
- **SYNONYM**: 동의어·복합어 사전으로 정규화 후 일치. "라이딩" ⊂ "자전거용".
- **NUMERIC**: 단위 변환 후 숫자 일치(허용 오차 설정값).
- **RANGE**: 값이 `range` 안에 드는지. `wanted`가 범위고 제시값이 단일이면 포함 검사.
- **SUBSTRING**: 느슨한 포함. fallback.
- **FREETEXT**: 룰 판정 불가 표시 → LLM 보조 결과를 사용.

각 Constraint의 `exclusions`는 해당 속성 매칭 전에 먼저 본다. 제시값이 제외값에 걸리면 그 언급은 이 속성에서 불일치가 아니라 **제외**로 표시한다.

### 4.3 MentionExtractor (글 → ProductMention[])

- 저tier 생성형 LLM 포트. 본문·댓글 + SearchSpec(어떤 속성을 뽑아야 하는지 힌트)을 넣는다.
- JSON schema 구조화 출력으로 `ProductMention[]`을 받는다.
- 비교 글이면 복수 언급을 분리해 추출하도록 프롬프트한다.
- 앞단에 **CoarseGate**(순수 룰): 글에 카테고리/속성 키워드가 하나도 없으면 추출 호출을 건너뛰고 빈 결과.

### 4.4 MentionMatcher (언급 → 채택 여부)

- 각 `ProductMention`을 `SearchSpec.constraints`와 비교한다.
- 하드 제약을 전부 만족(각 속성의 match_mode 매처 통과)하고, 어떤 속성의 exclusions에도 걸리지 않으면 `matched=True`.
- 하나라도 제외에 걸리면 `matched=False, reject_reason="제외: <속성>=<값>"`.
- 하드 속성 중 추출값이 없는 것이 있으면 애매로 두고, FREETEXT/소프트 신호와 LLM 보조로 보완(설정에 따라 기본은 불일치 처리).
- 채택된 언급의 `opinions`만 하류로 넘긴다.

### 4.5 AspectNormalizer

- aspect 동의어를 카테고리 단위 canonical로 정규화("황변"→"변색").

## 5. 데이터 흐름

```
질의 ─(고tier LLM)→ SearchSpec(구조화)
                        │
RawPost ─(CoarseGate 룰)─ 키워드 전무 → 버림
         └─ 통과 ─(저tier LLM)→ ProductMention[]
                        │
         (룰 MentionMatcher: match_mode + 속성단위 제외)
                        │
            채택 언급의 Opinion[]  ─→ (1b 리포트로)
```

## 6. 포트와 어댑터

- `SpecParser`(고tier): `parse(query, known_schema?) -> ParseResult`
- `MentionExtractor`(저tier): `extract(post, spec) -> list[ProductMention]`
- `SchemaCache`: `get/put/classify_category`
- 어댑터: `OpenAiSpecParser`, `OpenAiMentionExtractor`(모델명 설정값), 테스트용 `FakeSpecParser`/`FakeMentionExtractor`.
- 매처(`MentionMatcher`, MatchMode 구현)는 포트가 아니라 순수 도메인. LLM 없음.

## 7. 테스트 전략

- **MatchMode 매처 단위테스트**: mode별로 일치/불일치/경계/제외 케이스. EXACT의 부분문자열 비매칭, RANGE 경계, NUMERIC 단위 변환 포함.
- **MentionMatcher 테스트**: 비교 글(복수 언급)에서 대상만 채택, 제외 언급 배제, 하드 누락 처리.
- **CategoryResolver 테스트**: 캐시 히트/미스, 상품명 분해, 별칭 연결, 애매 되묻기.
- **카테고리 분산 테스트군**(휴대폰 치우침 제거): 최소 아래를 포함한다.
  - 울양말 / 면양말 / 메리노울 (소재 enum, 언급 단위 제외)
  - 워커부츠 / 첼시부츠 / 등산화 (종류 enum)
  - 자전거용 장갑 / 오토바이 장갑 / 작업용 장갑 (용도 synonym)
  - 로봇청소기 / 무선청소기 (종류 enum)
  - 숫자·범위 상품 (사이즈 240~260mm 등)
  - 한 글에서 여러 상품 비교
  - 동의어·별칭만으로 상품 지칭 (라이딩 글러브, 메리노삭스)
- **LLM 포트**: 추출 단계는 fake로 단위테스트. 추출 프롬프트/파싱은 순수 함수로 분리해 계약 테스트.
- **골든셋(룰 계층)**: 카테고리군 × 상황(정상·유사·제외·비교·애매·광고) 분포로 손 라벨 케이스. 룰 매처를 라벨에 대고 돌려 precision 측정. 100건+ 실제 라벨과 LLM precision/recall 게이트는 2단계.

## 8. 열린 질문

- EXACT 토큰 경계 판정을 한글/영문/숫자 전이 기반으로 할지, 형태소 분석기까지 쓸지 (초기엔 전이 기반, 부족하면 보강)
- NUMERIC/RANGE 단위 사전의 초기 범위 (mm/cm, g/kg부터)
- 하드 속성 추출값 누락 시 기본 정책(불일치 vs 애매→LLM 보조)의 카테고리별 조정 여부
