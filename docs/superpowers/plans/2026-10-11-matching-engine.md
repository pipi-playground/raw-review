# 매칭 엔진 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 임의의 상품 질의를 구조화 `SearchSpec`으로 해석하고, 글에서 추출한 `ProductMention[]`을 룰로 매칭해 대상 상품 의견만 `Opinion[]`으로 낸다. 수집·API 없이 고정 입력과 fake로 완결된다.

**Architecture:** 공유 커널(`shared`)에 경계를 넘는 계약을 둔다. `catalog`(질의→SearchSpec)와 `matching`(언급→채택) context는 서로 import하지 않고 `shared`만 공유한다. LLM은 2-tier 생성형 포트(고tier 질의이해 / 저tier 추출) 뒤에 두고 추출·정규화만 맡는다. **매칭 최종 판정은 전부 순수 룰.**

**Tech Stack:** Python 3.12, Pydantic v2, pytest, OpenAI SDK, import-linter.

**Spec:** `docs/superpowers/specs/2026-10-11-matching-engine-design.md` (상위: `2026-10-11-raw-review-design.md`)

## Global Constraints

- Python 3.12, Pydantic v2.
- 공유 커널 규칙: 경계 계약은 `shared/`. `catalog`·`matching`은 서로 import 금지, `shared`만 import. 각 `domain/`은 `adapters`·`openai` import 금지. import-linter로 강제.
- LLM은 추출·정규화만. 매칭 판정(MATCHED/REJECTED/MISMATCH/UNCERTAIN)은 순수 룰.
- match_mode 유효성: hard 속성은 EXACT/ENUM/NUMERIC/RANGE/SYNONYM만. SUBSTRING·FREETEXT는 soft 전용. Pydantic validator로 강제.
- 제외는 속성 단위 canonical. role=USED 언급의 subject 값에만 적용.
- 하드 속성 추출 누락 → UNCERTAIN(채택 안 함). precision 우선.
- `Opinion.quote` 120자 제한. 단점 인용 필수.
- 단위 변환: 길이 mm 기준(cm×10, inch×25.4), 무게 g 기준(kg×1000).
- 모델명·임계값은 `config.py` 설정값. 하드코딩 금지.

## Review Focus

- **EXACT 부분문자열**: `아이폰17`이 추출값 `아이폰17 Pro`와 일치로 처리되면 안 된다. 토큰 시퀀스 완전 일치여야 한다. (Task 7)
- **제외가 USED에만**: "17 Pro는 안 맞고(REJECTED_ALT) 17 썼다(USED)"에서 17 Pro가 제외 트리거가 되면 안 된다. (Task 12)
- **비교 글 대안 혼입**: "면양말 별로, 울양말 좋음"에서 면양말(COMPARED) 의견이 채택되면 안 된다. (Task 12)
- **하드 누락 오채택**: 형태 미추출 글이 MATCHED로 채택되면 precision이 깨진다. UNCERTAIN이어야 한다. (Task 12)
- **SYNONYM 방향 오인**: "오토바이 라이딩 장갑"이 용도=자전거로 매칭되면 안 된다. canonical 동등이어야 한다. (Task 8)
- **hard+SUBSTRING 설정**: 하드 속성에 SUBSTRING/FREETEXT를 주면 생성 시점에 막혀야 한다. (Task 4)

---

## File Structure

```
apps/api/
  pyproject.toml
  setup.cfg
  src/raw_review/
    __init__.py
    config.py
    shared/
      __init__.py
      text.py              # normalize, tokenize, 토큰 시퀀스 유틸
      models.py            # RawPost, RawComment
      search_spec.py       # MatchMode, AttributeDef, AttributeSchema,
                           #   AttributeConstraint, SearchSpec, AmbiguousQuery (+validator)
      mention.py           # MentionRole, ExtractedAttribute, MentionOpinion, ProductMention
      verdict.py           # Polarity, Opinion
      match_result.py      # MatchOutcome, MentionMatch
      ports.py             # ParseResult, SpecParser, ExtractResult, MentionExtractor, SchemaCache
    matching/
      __init__.py
      domain/
        __init__.py
        matchers.py        # mode별 매처 + match_attribute 디스패치
        coarse_gate.py     # CoarseGate
        mention_matcher.py # MentionMatcher
        aspect_normalizer.py
        to_opinions.py     # 변환 계약
    catalog/
      __init__.py
      application/
        __init__.py
        schema_cache.py    # InMemorySchemaCache
        resolve_category.py
    adapters/
      __init__.py
      fakes.py             # FakeSpecParser, FakeMentionExtractor
      openai_spec_parser.py
      openai_mention_extractor.py
    eval/
      __init__.py
      metrics.py
  tests/
    __init__.py
    test_boundaries.py
    test_config.py
    shared/__init__.py  test_text.py test_search_spec.py test_mention.py test_verdict.py
    matching/__init__.py  test_matchers.py test_coarse_gate.py test_mention_matcher.py
                          test_aspect_normalizer.py test_to_opinions.py
    catalog/__init__.py  test_resolve_category.py
    adapters/__init__.py  test_openai_spec_parser.py test_openai_mention_extractor.py
    eval/__init__.py  golden_cases.jsonl conftest.py test_matching_eval.py
```

---

### Task 1: 스캐폴드와 경계 계약

**Files:**
- Create: `apps/api/pyproject.toml`, `apps/api/setup.cfg`, 모든 패키지 `__init__.py`
- Test: `apps/api/tests/test_boundaries.py`

**Interfaces:**
- Consumes: 없음
- Produces: 설치 가능한 `raw_review`, import-linter 계약

- [ ] **Step 1: pyproject.toml**

```toml
[project]
name = "raw-review-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["pydantic>=2.6", "pydantic-settings>=2.2", "openai>=1.40"]

[project.optional-dependencies]
dev = ["pytest>=8", "pytest-asyncio>=0.23", "import-linter>=2.0"]

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
pythonpath = ["src"]
asyncio_mode = "auto"
```

- [ ] **Step 2: setup.cfg 계약**

```ini
[importlinter]
root_package = raw_review

[importlinter:contract:context-independence]
name = catalog/matching는 서로 import하지 않는다
type = independence
modules =
    raw_review.catalog
    raw_review.matching

[importlinter:contract:domain-pure]
name = matching.domain은 adapters/openai를 모른다
type = forbidden
source_modules =
    raw_review.matching.domain
forbidden_modules =
    raw_review.adapters
    openai

[importlinter:contract:shared-pure]
name = shared는 어떤 context도 모른다
type = forbidden
source_modules =
    raw_review.shared
forbidden_modules =
    raw_review.catalog
    raw_review.matching
    raw_review.adapters
    openai
```

- [ ] **Step 3: 빈 `__init__.py` 생성** (File Structure 전부)

- [ ] **Step 4: 경계 테스트**

`tests/test_boundaries.py`:

```python
import subprocess, sys

def test_import_contracts_hold():
    r = subprocess.run([sys.executable, "-m", "importlinter.cli", "lint"],
                       cwd="apps/api", capture_output=True, text=True)
    assert r.returncode == 0, r.stdout + r.stderr
```

- [ ] **Step 5: 설치 + 실행**

Run: `cd apps/api && pip install -e ".[dev]" && pytest tests/test_boundaries.py -v`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api
git commit -m "chore: scaffold matching-engine package with boundaries"
```

---

### Task 2: 텍스트 유틸 (normalize, tokenize)

EXACT 토큰 경계 판정의 기반. 한글/영문/숫자 전이와 구분자로 토큰을 끊는다.

**Files:**
- Create: `apps/api/src/raw_review/shared/text.py`
- Test: `apps/api/tests/shared/test_text.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `normalize(text: str) -> str` — 공백 제거 + 소문자
  - `tokenize(text: str) -> list[str]` — 한글/영문/숫자 묶음 단위 토큰
  - `token_seq_equal(a: str, b: str) -> bool` — 토큰 시퀀스 완전 일치
  - `token_subseq(needle: str, haystack: str) -> bool` — needle 토큰열이 haystack 토큰열의 연속 부분열

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_text.py`:

```python
from raw_review.shared.text import normalize, tokenize, token_seq_equal, token_subseq

def test_normalize():
    assert normalize("아이폰 17") == "아이폰17"
    assert normalize("  iPhone 17 ") == "iphone17"

def test_tokenize_splits_script_and_digit_transitions():
    assert tokenize("아이폰17 Pro") == ["아이폰", "17", "pro"]
    assert tokenize("240mm") == ["240", "mm"]

def test_token_seq_equal_exact_only():
    assert token_seq_equal("아이폰17", "아이폰 17") is True
    assert token_seq_equal("아이폰17", "아이폰17 Pro") is False  # 부분문자열 아님

def test_token_subseq_for_exclusions():
    assert token_subseq("17 Pro", "아이폰17 Pro") is True
    assert token_subseq("17 Pro", "아이폰17") is False
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_text.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/shared/text.py`:

```python
import re

_SPACE = re.compile(r"\s+")
# 한글 / 영문 / 숫자 덩어리
_TOKEN = re.compile(r"[가-힣]+|[a-zA-Z]+|[0-9]+")

def normalize(text: str) -> str:
    return _SPACE.sub("", text.strip()).lower()

def tokenize(text: str) -> list[str]:
    return [t.lower() for t in _TOKEN.findall(text)]

def token_seq_equal(a: str, b: str) -> bool:
    return tokenize(a) == tokenize(b)

def token_subseq(needle: str, haystack: str) -> bool:
    n, h = tokenize(needle), tokenize(haystack)
    if not n:
        return False
    return any(h[i:i+len(n)] == n for i in range(len(h) - len(n) + 1))
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/shared/test_text.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/shared/text.py apps/api/tests/shared/test_text.py
git commit -m "feat: add text tokenization with script-boundary tokens"
```

---

### Task 3: 입력 모델 (RawPost / RawComment)

**Files:**
- Create: `apps/api/src/raw_review/shared/models.py`
- Test: `apps/api/tests/shared/test_models.py` (아래 Task 5 테스트와 합쳐도 됨; 별도 유지)

**Interfaces:**
- Consumes: 없음
- Produces: `RawComment(text, likes=0)`, `RawPost(source, url, title, body, posted_at, rating, likes, comments)`

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_models.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost, RawComment

def test_rawpost():
    p = RawPost(source="dc", url="u", title="t", body="b",
                posted_at=datetime(2026,10,1), rating=None, likes=2,
                comments=[RawComment(text="c")])
    assert p.comments[0].likes == 0 and p.rating is None
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_models.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/shared/models.py`:

```python
from datetime import datetime
from pydantic import BaseModel

class RawComment(BaseModel):
    text: str
    likes: int = 0

class RawPost(BaseModel):
    source: str
    url: str
    title: str
    body: str
    posted_at: datetime | None = None
    rating: int | None = None
    likes: int = 0
    comments: list[RawComment] = []
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/shared/test_models.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/shared/models.py apps/api/tests/shared/test_models.py
git commit -m "feat: add RawPost/RawComment"
```

---

### Task 4: 속성·SearchSpec 모델 (+ match_mode validator)

**Files:**
- Create: `apps/api/src/raw_review/shared/search_spec.py`
- Test: `apps/api/tests/shared/test_search_spec.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `MatchMode` enum: `EXACT, ENUM, SYNONYM, NUMERIC, RANGE, SUBSTRING, FREETEXT`
  - `AttributeDef(name, hard, match_mode, canonical_values, aliases, exclusions, unit, normalization)` — hard면 mode ∈ {EXACT,ENUM,NUMERIC,RANGE,SYNONYM}, 아니면 ValueError
  - `AttributeSchema(category, category_aliases, attributes)`
  - `AttributeConstraint(name, hard, match_mode, wanted, range, unit, aliases, exclusions)` — 동일 validator
  - `SearchSpec(category, constraints)`
  - `AmbiguousQuery(query, options)`

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_search_spec.py`:

```python
import pytest
from raw_review.shared.search_spec import (
    MatchMode, AttributeDef, AttributeConstraint, SearchSpec)

def test_hard_substring_rejected():
    with pytest.raises(ValueError):
        AttributeDef(name="소재", hard=True, match_mode=MatchMode.SUBSTRING)

def test_hard_freetext_rejected():
    with pytest.raises(ValueError):
        AttributeConstraint(name="용도", hard=True, match_mode=MatchMode.FREETEXT)

def test_soft_substring_ok():
    a = AttributeDef(name="느낌", hard=False, match_mode=MatchMode.SUBSTRING)
    assert a.match_mode == MatchMode.SUBSTRING

def test_hard_exact_ok():
    c = AttributeConstraint(name="대상기기", hard=True, match_mode=MatchMode.EXACT,
                            wanted=["아이폰17"], exclusions=["17 Pro"])
    assert c.wanted == ["아이폰17"]
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_search_spec.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/shared/search_spec.py`:

```python
from enum import Enum
from pydantic import BaseModel, model_validator

class MatchMode(str, Enum):
    EXACT = "EXACT"
    ENUM = "ENUM"
    SYNONYM = "SYNONYM"
    NUMERIC = "NUMERIC"
    RANGE = "RANGE"
    SUBSTRING = "SUBSTRING"
    FREETEXT = "FREETEXT"

_HARD_ALLOWED = {MatchMode.EXACT, MatchMode.ENUM, MatchMode.NUMERIC,
                 MatchMode.RANGE, MatchMode.SYNONYM}

def _check_hard_mode(hard: bool, mode: MatchMode) -> None:
    if hard and mode not in _HARD_ALLOWED:
        raise ValueError(f"hard 속성은 {mode} 를 쓸 수 없다 (EXACT/ENUM/NUMERIC/RANGE/SYNONYM만)")

class AttributeDef(BaseModel):
    name: str
    hard: bool
    match_mode: MatchMode
    canonical_values: list[str] = []
    aliases: dict[str, list[str]] = {}
    exclusions: list[str] = []
    unit: str | None = None
    normalization: str | None = None

    @model_validator(mode="after")
    def _v(self):
        _check_hard_mode(self.hard, self.match_mode)
        return self

class AttributeSchema(BaseModel):
    category: str
    category_aliases: list[str] = []
    attributes: list[AttributeDef]

class AttributeConstraint(BaseModel):
    name: str
    hard: bool
    match_mode: MatchMode
    wanted: list[str] = []
    range: tuple[float, float] | None = None
    unit: str | None = None
    aliases: dict[str, list[str]] = {}
    exclusions: list[str] = []

    @model_validator(mode="after")
    def _v(self):
        _check_hard_mode(self.hard, self.match_mode)
        return self

class SearchSpec(BaseModel):
    category: str
    constraints: list[AttributeConstraint]

class AmbiguousQuery(BaseModel):
    query: str
    options: list[str]
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/shared/test_search_spec.py -v` → PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/shared/search_spec.py apps/api/tests/shared/test_search_spec.py
git commit -m "feat: add structured SearchSpec with match_mode validator"
```

---

### Task 5: 언급 모델 (ProductMention)

**Files:**
- Create: `apps/api/src/raw_review/shared/mention.py`
- Create: `apps/api/src/raw_review/shared/verdict.py`
- Test: `apps/api/tests/shared/test_mention.py`, `apps/api/tests/shared/test_verdict.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `Polarity` enum: `PRO, CON`
  - `Opinion(brand: str | None, polarity, aspect, quote, source_part)` — quote 120자 제한
  - `MentionRole` enum: `USED, COMPARED, REJECTED_ALT, MENTIONED`
  - `ExtractedAttribute(name, raw_value, normalized_value, numeric_value, lower, upper, unit)`
  - `MentionOpinion(polarity, aspect, quote, source_part)` — quote 120자 제한
  - `ProductMention(raw, role, brand, attributes: list[ExtractedAttribute], opinions: list[MentionOpinion])`

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_verdict.py`:

```python
from raw_review.shared.verdict import Opinion, Polarity

def test_opinion_quote_truncated():
    o = Opinion(brand="A", polarity=Polarity.CON, aspect="변색",
                quote="변"*200, source_part="post")
    assert len(o.quote) == 120
```

`tests/shared/test_mention.py`:

```python
from raw_review.shared.mention import (
    ProductMention, MentionRole, ExtractedAttribute, MentionOpinion)
from raw_review.shared.verdict import Polarity

def test_mention_with_numeric_attribute():
    m = ProductMention(
        raw="데카트론 울양말 240 따뜻", role=MentionRole.USED, brand="데카트론",
        attributes=[ExtractedAttribute(name="사이즈", raw_value="240",
                                       numeric_value=240.0, unit="mm")],
        opinions=[MentionOpinion(polarity=Polarity.PRO, aspect="보온",
                                 quote="따뜻", source_part="post")])
    assert m.role == MentionRole.USED
    assert m.attributes[0].numeric_value == 240.0

def test_mention_opinion_quote_truncated():
    mo = MentionOpinion(polarity=Polarity.CON, aspect="보풀",
                        quote="보"*200, source_part="post")
    assert len(mo.quote) == 120
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_mention.py tests/shared/test_verdict.py -v` → FAIL

- [ ] **Step 3: verdict.py 구현**

`src/raw_review/shared/verdict.py`:

```python
from enum import Enum
from pydantic import BaseModel, field_validator

QUOTE_MAX = 120

class Polarity(str, Enum):
    PRO = "pro"
    CON = "con"

class Opinion(BaseModel):
    brand: str | None = None
    polarity: Polarity
    aspect: str
    quote: str
    source_part: str

    @field_validator("quote")
    @classmethod
    def _t(cls, v: str) -> str:
        return v[:QUOTE_MAX]
```

- [ ] **Step 4: mention.py 구현**

`src/raw_review/shared/mention.py`:

```python
from enum import Enum
from pydantic import BaseModel, field_validator
from raw_review.shared.verdict import Polarity, QUOTE_MAX

class MentionRole(str, Enum):
    USED = "USED"
    COMPARED = "COMPARED"
    REJECTED_ALT = "REJECTED_ALT"
    MENTIONED = "MENTIONED"

class ExtractedAttribute(BaseModel):
    name: str
    raw_value: str
    normalized_value: str | None = None
    numeric_value: float | None = None
    lower: float | None = None
    upper: float | None = None
    unit: str | None = None

class MentionOpinion(BaseModel):
    polarity: Polarity
    aspect: str
    quote: str
    source_part: str

    @field_validator("quote")
    @classmethod
    def _t(cls, v: str) -> str:
        return v[:QUOTE_MAX]

class ProductMention(BaseModel):
    raw: str
    role: MentionRole
    brand: str | None = None
    attributes: list[ExtractedAttribute] = []
    opinions: list[MentionOpinion] = []
```

- [ ] **Step 5: 통과** — Run: `cd apps/api && pytest tests/shared/test_mention.py tests/shared/test_verdict.py -v` → PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/raw_review/shared/mention.py apps/api/src/raw_review/shared/verdict.py apps/api/tests/shared/test_mention.py apps/api/tests/shared/test_verdict.py
git commit -m "feat: add ProductMention and Opinion models"
```

---

### Task 6: 매칭 결과 모델

**Files:**
- Create: `apps/api/src/raw_review/shared/match_result.py`
- Test: `apps/api/tests/shared/test_match_result.py`

**Interfaces:**
- Consumes: `ProductMention` (Task 5)
- Produces:
  - `MatchOutcome` enum: `MATCHED, REJECTED, MISMATCH, UNCERTAIN`
  - `MentionMatch(mention, outcome, matched_attributes, reason)`

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_match_result.py`:

```python
from raw_review.shared.match_result import MatchOutcome, MentionMatch
from raw_review.shared.mention import ProductMention, MentionRole

def test_match_result():
    m = ProductMention(raw="r", role=MentionRole.USED)
    mm = MentionMatch(mention=m, outcome=MatchOutcome.MATCHED,
                      matched_attributes=["대상기기"])
    assert mm.outcome == MatchOutcome.MATCHED
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_match_result.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/shared/match_result.py`:

```python
from enum import Enum
from pydantic import BaseModel
from raw_review.shared.mention import ProductMention

class MatchOutcome(str, Enum):
    MATCHED = "MATCHED"
    REJECTED = "REJECTED"
    MISMATCH = "MISMATCH"
    UNCERTAIN = "UNCERTAIN"

class MentionMatch(BaseModel):
    mention: ProductMention
    outcome: MatchOutcome
    matched_attributes: list[str] = []
    reason: str | None = None
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/shared/test_match_result.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/shared/match_result.py apps/api/tests/shared/test_match_result.py
git commit -m "feat: add MentionMatch result model"
```

---

### Task 7: EXACT 매처 (토큰 경계)

**Files:**
- Create: `apps/api/src/raw_review/matching/domain/matchers.py`
- Test: `apps/api/tests/matching/test_matchers.py` (+ `tests/matching/__init__.py`)

**Interfaces:**
- Consumes: `ExtractedAttribute` (Task 5), `AttributeConstraint` (Task 4), text 유틸 (Task 2)
- Produces:
  - `exclusion_hit(extracted: ExtractedAttribute, c: AttributeConstraint) -> bool`
  - `match_exact(extracted: ExtractedAttribute, c: AttributeConstraint) -> bool`

- [ ] **Step 1: 실패 테스트**

`tests/matching/test_matchers.py`:

```python
from raw_review.shared.search_spec import AttributeConstraint, MatchMode
from raw_review.shared.mention import ExtractedAttribute
from raw_review.matching.domain.matchers import match_exact, exclusion_hit

def _c(**kw):
    return AttributeConstraint(name="대상기기", hard=True,
                               match_mode=MatchMode.EXACT, **kw)

def _e(v):
    return ExtractedAttribute(name="대상기기", raw_value=v, normalized_value=v)

def test_exact_matches_token_sequence():
    assert match_exact(_e("아이폰17"), _c(wanted=["아이폰17"])) is True
    assert match_exact(_e("아이폰 17"), _c(wanted=["아이폰17"])) is True

def test_exact_rejects_superstring():
    assert match_exact(_e("아이폰17 Pro"), _c(wanted=["아이폰17"])) is False

def test_exclusion_hit_on_subsequence():
    c = _c(wanted=["아이폰17"], exclusions=["17 Pro"])
    assert exclusion_hit(_e("아이폰17 Pro"), c) is True
    assert exclusion_hit(_e("아이폰17"), c) is False
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/matching/domain/matchers.py`:

```python
from raw_review.shared.mention import ExtractedAttribute
from raw_review.shared.search_spec import AttributeConstraint
from raw_review.shared.text import token_seq_equal, token_subseq

def _value(e: ExtractedAttribute) -> str:
    return e.normalized_value or e.raw_value

def exclusion_hit(e: ExtractedAttribute, c: AttributeConstraint) -> bool:
    v = _value(e)
    return any(token_subseq(x, v) for x in c.exclusions)

def match_exact(e: ExtractedAttribute, c: AttributeConstraint) -> bool:
    v = _value(e)
    return any(token_seq_equal(v, w) for w in c.wanted)
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/matchers.py apps/api/tests/matching/test_matchers.py
git commit -m "feat: add EXACT matcher with token-boundary equality"
```

---

### Task 8: ENUM / SYNONYM 매처

**Files:**
- Modify: `apps/api/src/raw_review/matching/domain/matchers.py`
- Test: `apps/api/tests/matching/test_matchers.py` (추가)

**Interfaces:**
- Consumes: Task 7 모듈
- Produces:
  - `match_enum(e, c) -> bool` — `normalized_value`(canonical)가 `wanted`에 속하는지. alias도 접어 비교.
  - `match_synonym(e, c) -> bool` — canonical 동등(추출 단계에서 방향 처리됨). 구현은 ENUM과 같은 canonical 비교지만 의미 분리.

- [ ] **Step 1: 실패 테스트 추가**

`tests/matching/test_matchers.py`에 추가:

```python
from raw_review.matching.domain.matchers import match_enum, match_synonym

def _ce(**kw):
    return AttributeConstraint(name="소재", hard=True, match_mode=MatchMode.ENUM, **kw)

def test_enum_canonical_membership():
    e = ExtractedAttribute(name="소재", raw_value="메리노", normalized_value="울")
    assert match_enum(e, _ce(wanted=["울"])) is True
    assert match_enum(e, _ce(wanted=["면"])) is False

def test_enum_folds_alias():
    e = ExtractedAttribute(name="소재", raw_value="wool", normalized_value="wool")
    c = _ce(wanted=["울"], aliases={"울": ["wool", "모"]})
    assert match_enum(e, c) is True

def test_synonym_direction_not_substring():
    # 용도: 추출이 canonical "오토바이"를 넣음 → 자전거와 불일치
    moto = ExtractedAttribute(name="용도", raw_value="오토바이 라이딩", normalized_value="오토바이")
    c = AttributeConstraint(name="용도", hard=True, match_mode=MatchMode.SYNONYM,
                            wanted=["자전거"])
    assert match_synonym(moto, c) is False
    bike = ExtractedAttribute(name="용도", raw_value="라이딩 글러브", normalized_value="자전거")
    assert match_synonym(bike, c) is True
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → FAIL

- [ ] **Step 3: 구현 추가**

`matchers.py`에 추가:

```python
def _canonical_set(c: AttributeConstraint) -> set[str]:
    out: set[str] = set()
    for w in c.wanted:
        out.add(w)
        for alias in c.aliases.get(w, []):
            out.add(alias)
    return {s.lower() for s in out}

def match_enum(e: ExtractedAttribute, c: AttributeConstraint) -> bool:
    v = (e.normalized_value or e.raw_value).lower()
    return v in _canonical_set(c)

def match_synonym(e: ExtractedAttribute, c: AttributeConstraint) -> bool:
    # 방향 정규화는 추출 단계가 normalized_value에 canonical을 넣어 끝냄.
    # 룰은 canonical 동등만 본다.
    v = (e.normalized_value or e.raw_value).lower()
    return v in {w.lower() for w in c.wanted}
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/matchers.py apps/api/tests/matching/test_matchers.py
git commit -m "feat: add ENUM/SYNONYM matchers"
```

---

### Task 9: NUMERIC / RANGE 매처 + 단위 변환

**Files:**
- Modify: `apps/api/src/raw_review/matching/domain/matchers.py`
- Test: `apps/api/tests/matching/test_matchers.py` (추가)

**Interfaces:**
- Consumes: Task 7 모듈
- Produces:
  - `convert(value: float, from_unit: str, to_unit: str) -> float` — 길이(mm/cm/inch), 무게(g/kg). 미지원 단위 조합이면 `ValueError`.
  - `match_numeric(e, c, *, tolerance=0.0) -> bool`
  - `match_range(e, c) -> bool`

- [ ] **Step 1: 실패 테스트 추가**

```python
from raw_review.matching.domain.matchers import convert, match_numeric, match_range

def test_convert_length_and_weight():
    assert convert(24.0, "cm", "mm") == 240.0
    assert convert(1.0, "inch", "mm") == 254.0
    assert convert(1.5, "kg", "g") == 1500.0

def test_numeric_with_unit_conversion():
    e = ExtractedAttribute(name="사이즈", raw_value="24cm", numeric_value=24.0, unit="cm")
    c = AttributeConstraint(name="사이즈", hard=True, match_mode=MatchMode.NUMERIC,
                            wanted=["240"], unit="mm")
    assert match_numeric(e, c) is True

def test_range_contains_value():
    e = ExtractedAttribute(name="사이즈", raw_value="250", numeric_value=250.0, unit="mm")
    c = AttributeConstraint(name="사이즈", hard=True, match_mode=MatchMode.RANGE,
                            range=(240.0, 260.0), unit="mm")
    assert match_range(e, c) is True
    e2 = ExtractedAttribute(name="사이즈", raw_value="270", numeric_value=270.0, unit="mm")
    assert match_range(e2, c) is False
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → FAIL

- [ ] **Step 3: 구현 추가**

```python
# 기준 단위로의 배율
_FACTORS = {
    "mm": 1.0, "cm": 10.0, "inch": 25.4,   # 길이 기준 mm
    "g": 1.0, "kg": 1000.0,                 # 무게 기준 g
}
_DIMENSION = {"mm": "len", "cm": "len", "inch": "len", "g": "wt", "kg": "wt"}

def convert(value: float, from_unit: str, to_unit: str) -> float:
    if from_unit not in _FACTORS or to_unit not in _FACTORS:
        raise ValueError(f"미지원 단위: {from_unit}->{to_unit}")
    if _DIMENSION[from_unit] != _DIMENSION[to_unit]:
        raise ValueError(f"차원 불일치: {from_unit}->{to_unit}")
    return value * _FACTORS[from_unit] / _FACTORS[to_unit]

def _to_unit(e: ExtractedAttribute, unit: str | None) -> float | None:
    if e.numeric_value is None:
        return None
    if unit and e.unit and unit != e.unit:
        return convert(e.numeric_value, e.unit, unit)
    return e.numeric_value

def match_numeric(e: ExtractedAttribute, c: AttributeConstraint, *, tolerance: float = 0.0) -> bool:
    v = _to_unit(e, c.unit)
    if v is None:
        return False
    return any(abs(v - float(w)) <= tolerance for w in c.wanted)

def match_range(e: ExtractedAttribute, c: AttributeConstraint) -> bool:
    if c.range is None:
        return False
    v = _to_unit(e, c.unit)
    if v is None:
        return False
    lo, hi = c.range
    return lo <= v <= hi
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/matchers.py apps/api/tests/matching/test_matchers.py
git commit -m "feat: add NUMERIC/RANGE matchers with unit conversion"
```

---

### Task 10: SUBSTRING / FREETEXT + 디스패치

**Files:**
- Modify: `apps/api/src/raw_review/matching/domain/matchers.py`
- Test: `apps/api/tests/matching/test_matchers.py` (추가)

**Interfaces:**
- Consumes: Task 7-9 모듈
- Produces:
  - `match_substring(e, c) -> bool` — 느슨한 포함(soft 전용)
  - `AttrMatch` enum: `MATCH, NOMATCH, UNCERTAIN`
  - `match_attribute(e: ExtractedAttribute | None, c: AttributeConstraint) -> AttrMatch` — mode별 디스패치. 추출값 None이면 UNCERTAIN. FREETEXT는 normalized_value 있으면 ENUM처럼, 없으면 UNCERTAIN.

- [ ] **Step 1: 실패 테스트 추가**

```python
from raw_review.matching.domain.matchers import match_attribute, AttrMatch

def test_freetext_uncertain_without_canonical():
    c = AttributeConstraint(name="느낌", hard=False, match_mode=MatchMode.FREETEXT,
                            wanted=["부드러움"])
    e = ExtractedAttribute(name="느낌", raw_value="말랑하다", normalized_value=None)
    assert match_attribute(e, c) == AttrMatch.UNCERTAIN

def test_freetext_match_with_canonical():
    c = AttributeConstraint(name="느낌", hard=False, match_mode=MatchMode.FREETEXT,
                            wanted=["부드러움"])
    e = ExtractedAttribute(name="느낌", raw_value="말랑", normalized_value="부드러움")
    assert match_attribute(e, c) == AttrMatch.MATCH

def test_missing_extraction_is_uncertain():
    c = AttributeConstraint(name="대상기기", hard=True, match_mode=MatchMode.EXACT,
                            wanted=["아이폰17"])
    assert match_attribute(None, c) == AttrMatch.UNCERTAIN

def test_dispatch_exact():
    c = AttributeConstraint(name="대상기기", hard=True, match_mode=MatchMode.EXACT,
                            wanted=["아이폰17"])
    e = ExtractedAttribute(name="대상기기", raw_value="아이폰17", normalized_value="아이폰17")
    assert match_attribute(e, c) == AttrMatch.MATCH
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → FAIL

- [ ] **Step 3: 구현 추가**

```python
from enum import Enum
from raw_review.shared.search_spec import MatchMode
from raw_review.shared.text import normalize

class AttrMatch(str, Enum):
    MATCH = "MATCH"
    NOMATCH = "NOMATCH"
    UNCERTAIN = "UNCERTAIN"

def match_substring(e: ExtractedAttribute, c: AttributeConstraint) -> bool:
    v = normalize(e.normalized_value or e.raw_value)
    return any(normalize(w) in v for w in c.wanted)

def _bool(b: bool) -> AttrMatch:
    return AttrMatch.MATCH if b else AttrMatch.NOMATCH

def match_attribute(e: ExtractedAttribute | None, c: AttributeConstraint) -> AttrMatch:
    if e is None:
        return AttrMatch.UNCERTAIN
    mode = c.match_mode
    if mode == MatchMode.EXACT:
        return _bool(match_exact(e, c))
    if mode == MatchMode.ENUM:
        return _bool(match_enum(e, c))
    if mode == MatchMode.SYNONYM:
        return _bool(match_synonym(e, c))
    if mode == MatchMode.NUMERIC:
        return _bool(match_numeric(e, c))
    if mode == MatchMode.RANGE:
        return _bool(match_range(e, c))
    if mode == MatchMode.SUBSTRING:
        return _bool(match_substring(e, c))
    # FREETEXT: canonical 있으면 ENUM처럼, 없으면 UNCERTAIN
    if e.normalized_value is None:
        return AttrMatch.UNCERTAIN
    return _bool(match_enum(e, c))
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_matchers.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/matchers.py apps/api/tests/matching/test_matchers.py
git commit -m "feat: add SUBSTRING/FREETEXT and match_attribute dispatch"
```

---

### Task 11: CoarseGate

**Files:**
- Create: `apps/api/src/raw_review/matching/domain/coarse_gate.py`
- Test: `apps/api/tests/matching/test_coarse_gate.py`

**Interfaces:**
- Consumes: `RawPost` (Task 3), `SearchSpec` (Task 4), text 유틸 (Task 2)
- Produces: `has_any_signal(post: RawPost, spec: SearchSpec) -> bool` — 글에 카테고리/속성 키워드(값·별칭·canonical)가 하나라도 있으면 True

- [ ] **Step 1: 실패 테스트**

`tests/matching/test_coarse_gate.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec, AttributeConstraint, MatchMode
from raw_review.matching.domain.coarse_gate import has_any_signal

def _spec():
    return SearchSpec(category="양말", constraints=[
        AttributeConstraint(name="소재", hard=True, match_mode=MatchMode.ENUM,
                            wanted=["울"], aliases={"울": ["wool", "메리노"]})])

def _post(body):
    return RawPost(source="dc", url="u", title="t", body=body,
                   posted_at=datetime(2026,10,1), likes=0, comments=[])

def test_signal_present():
    assert has_any_signal(_post("메리노 양말 따뜻함"), _spec()) is True

def test_no_signal():
    assert has_any_signal(_post("오늘 날씨 좋다"), _spec()) is False
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_coarse_gate.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/matching/domain/coarse_gate.py`:

```python
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.text import normalize

def _keywords(spec: SearchSpec) -> set[str]:
    kw = {normalize(spec.category)}
    for c in spec.constraints:
        for w in c.wanted:
            kw.add(normalize(w))
        for variants in c.aliases.values():
            for v in variants:
                kw.add(normalize(v))
    return {k for k in kw if k}

def has_any_signal(post: RawPost, spec: SearchSpec) -> bool:
    text = normalize(post.title + " " + post.body
                     + " ".join(c.text for c in post.comments))
    return any(kw in text for kw in _keywords(spec))
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_coarse_gate.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/coarse_gate.py apps/api/tests/matching/test_coarse_gate.py
git commit -m "feat: add coarse keyword gate"
```

---

### Task 12: MentionMatcher (핵심)

**Files:**
- Create: `apps/api/src/raw_review/matching/domain/mention_matcher.py`
- Test: `apps/api/tests/matching/test_mention_matcher.py`

**Interfaces:**
- Consumes: `ProductMention`/`MentionRole`/`ExtractedAttribute` (Task 5), `SearchSpec` (Task 4), `MentionMatch`/`MatchOutcome` (Task 6), `match_attribute`/`exclusion_hit`/`AttrMatch` (Task 7-10)
- Produces:
  - `match_mention(mention: ProductMention, spec: SearchSpec) -> MentionMatch`
  - 규칙: role≠USED → `MISMATCH`(reason="비대상 role"). USED면 각 하드 constraint의 추출값으로 `exclusion_hit` 먼저 검사(걸리면 `REJECTED`). 하드 속성 추출 없음 → `UNCERTAIN`. 하드 매처 NOMATCH → `MISMATCH`. 전부 MATCH → `MATCHED`.
  - `match_post(mentions, spec) -> list[MentionMatch]`

- [ ] **Step 1: 실패 테스트**

`tests/matching/test_mention_matcher.py`:

```python
from raw_review.shared.search_spec import SearchSpec, AttributeConstraint, MatchMode
from raw_review.shared.mention import ProductMention, MentionRole, ExtractedAttribute
from raw_review.shared.match_result import MatchOutcome
from raw_review.matching.domain.mention_matcher import match_mention

def _spec():
    return SearchSpec(category="휴대폰케이스", constraints=[
        AttributeConstraint(name="대상기기", hard=True, match_mode=MatchMode.EXACT,
                            wanted=["아이폰17"], exclusions=["17 Pro"]),
        AttributeConstraint(name="형태", hard=True, match_mode=MatchMode.ENUM,
                            wanted=["투명"], aliases={"투명": ["클리어"]})])

def _attr(name, val, norm=None):
    return ExtractedAttribute(name=name, raw_value=val, normalized_value=norm or val)

def _m(role, attrs):
    return ProductMention(raw="r", role=role, brand="다이소", attributes=attrs)

def test_used_full_match():
    m = _m(MentionRole.USED, [_attr("대상기기","아이폰17"), _attr("형태","투명")])
    assert match_mention(m, _spec()).outcome == MatchOutcome.MATCHED

def test_compared_mention_not_adopted():
    m = _m(MentionRole.COMPARED, [_attr("대상기기","아이폰17"), _attr("형태","투명")])
    assert match_mention(m, _spec()).outcome == MatchOutcome.MISMATCH

def test_rejected_alt_not_trigger_exclusion():
    # 17 Pro는 REJECTED_ALT → 제외 트리거 아님. (USED 아님이라 어차피 비채택)
    m = _m(MentionRole.REJECTED_ALT, [_attr("대상기기","아이폰17 Pro"), _attr("형태","투명")])
    assert match_mention(m, _spec()).outcome == MatchOutcome.MISMATCH

def test_used_with_excluded_value_is_rejected():
    m = _m(MentionRole.USED, [_attr("대상기기","아이폰17 Pro"), _attr("형태","투명")])
    assert match_mention(m, _spec()).outcome == MatchOutcome.REJECTED

def test_missing_hard_attr_is_uncertain():
    m = _m(MentionRole.USED, [_attr("대상기기","아이폰17")])  # 형태 없음
    assert match_mention(m, _spec()).outcome == MatchOutcome.UNCERTAIN

def test_hard_mismatch():
    m = _m(MentionRole.USED, [_attr("대상기기","갤럭시S25"), _attr("형태","투명")])
    assert match_mention(m, _spec()).outcome == MatchOutcome.MISMATCH
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_mention_matcher.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/matching/domain/mention_matcher.py`:

```python
from raw_review.shared.mention import ProductMention, MentionRole, ExtractedAttribute
from raw_review.shared.search_spec import SearchSpec, AttributeConstraint
from raw_review.shared.match_result import MentionMatch, MatchOutcome
from raw_review.matching.domain.matchers import match_attribute, exclusion_hit, AttrMatch

def _find(mention: ProductMention, name: str) -> ExtractedAttribute | None:
    for a in mention.attributes:
        if a.name == name:
            return a
    return None

def match_mention(mention: ProductMention, spec: SearchSpec) -> MentionMatch:
    if mention.role != MentionRole.USED:
        return MentionMatch(mention=mention, outcome=MatchOutcome.MISMATCH,
                            reason="비대상 role")

    hard = [c for c in spec.constraints if c.hard]

    # 1) 제외: USED subject 값이 제외에 걸리면 REJECTED
    for c in hard:
        e = _find(mention, c.name)
        if e is not None and exclusion_hit(e, c):
            return MentionMatch(mention=mention, outcome=MatchOutcome.REJECTED,
                                reason=f"제외: {c.name}")

    # 2) 하드 매칭
    matched: list[str] = []
    for c in hard:
        e = _find(mention, c.name)
        res = match_attribute(e, c)
        if res == AttrMatch.UNCERTAIN:
            return MentionMatch(mention=mention, outcome=MatchOutcome.UNCERTAIN,
                                reason=f"하드 속성 누락/모호: {c.name}")
        if res == AttrMatch.NOMATCH:
            return MentionMatch(mention=mention, outcome=MatchOutcome.MISMATCH,
                                reason=f"하드 불일치: {c.name}")
        matched.append(c.name)

    return MentionMatch(mention=mention, outcome=MatchOutcome.MATCHED,
                        matched_attributes=matched)

def match_post(mentions: list[ProductMention], spec: SearchSpec) -> list[MentionMatch]:
    return [match_mention(m, spec) for m in mentions]
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_mention_matcher.py -v` → PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/mention_matcher.py apps/api/tests/matching/test_mention_matcher.py
git commit -m "feat: add MentionMatcher with role filter and per-attribute exclusion"
```

---

### Task 13: AspectNormalizer

**Files:**
- Create: `apps/api/src/raw_review/matching/domain/aspect_normalizer.py`
- Test: `apps/api/tests/matching/test_aspect_normalizer.py`

**Interfaces:**
- Consumes: 없음
- Produces: `AspectNormalizer(synonyms)`, `.canonical(aspect) -> str`, `.default()`

- [ ] **Step 1: 실패 테스트**

`tests/matching/test_aspect_normalizer.py`:

```python
from raw_review.matching.domain.aspect_normalizer import AspectNormalizer

def test_canonical_and_passthrough():
    n = AspectNormalizer(synonyms={"변색": ["황변", "누래짐"]})
    assert n.canonical("황변") == "변색"
    assert n.canonical("무게") == "무게"

def test_default():
    assert AspectNormalizer.default().canonical("황변") == "변색"
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_aspect_normalizer.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/matching/domain/aspect_normalizer.py`:

```python
from __future__ import annotations
from pydantic import BaseModel

_DEFAULT = {
    "변색": ["황변", "누래짐", "누렇게", "색변함"],
    "보풀": ["보풀", "필링", "먼지"],
    "내구성": ["깨짐", "파손", "찢어짐", "구멍"],
    "보온": ["따뜻", "보온성", "따듯"],
    "그립감": ["미끄러움", "미끌", "손에 쥐는 느낌"],
    "가격": ["가성비", "비쌈", "저렴"],
}

class AspectNormalizer(BaseModel):
    synonyms: dict[str, list[str]]

    def canonical(self, aspect: str) -> str:
        for canon, syns in self.synonyms.items():
            if aspect == canon or aspect in syns:
                return canon
        return aspect

    @classmethod
    def default(cls) -> "AspectNormalizer":
        return cls(synonyms=_DEFAULT)
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_aspect_normalizer.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/aspect_normalizer.py apps/api/tests/matching/test_aspect_normalizer.py
git commit -m "feat: add aspect normalizer"
```

---

### Task 14: Opinion 변환 계약

**Files:**
- Create: `apps/api/src/raw_review/matching/domain/to_opinions.py`
- Test: `apps/api/tests/matching/test_to_opinions.py`

**Interfaces:**
- Consumes: `MentionMatch`/`MatchOutcome` (Task 6), `Opinion`/`Polarity` (Task 5), `AspectNormalizer` (Task 13), `RawPost` (Task 3)
- Produces: `to_opinions(matches: list[MentionMatch], post: RawPost, normalizer: AspectNormalizer) -> list[Opinion]` — MATCHED 언급의 의견만, aspect 정규화, brand 보존(None 허용), quote 120자

- [ ] **Step 1: 실패 테스트**

`tests/matching/test_to_opinions.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost
from raw_review.shared.mention import ProductMention, MentionRole, MentionOpinion
from raw_review.shared.verdict import Polarity
from raw_review.shared.match_result import MentionMatch, MatchOutcome
from raw_review.matching.domain.aspect_normalizer import AspectNormalizer
from raw_review.matching.domain.to_opinions import to_opinions

def _post():
    return RawPost(source="dc", url="http://x/1", title="t", body="b",
                   posted_at=datetime(2026,10,1), likes=0, comments=[])

def _match(outcome, brand, aspect):
    m = ProductMention(raw="r", role=MentionRole.USED, brand=brand,
        opinions=[MentionOpinion(polarity=Polarity.CON, aspect=aspect,
                                 quote="누렇게 뜸", source_part="post")])
    return MentionMatch(mention=m, outcome=outcome)

def test_only_matched_opinions_converted():
    ms = [_match(MatchOutcome.MATCHED, "다이소", "황변"),
          _match(MatchOutcome.MISMATCH, "신지모루", "변색"),
          _match(MatchOutcome.REJECTED, "슈피겐", "변색")]
    ops = to_opinions(ms, _post(), AspectNormalizer.default())
    assert len(ops) == 1
    assert ops[0].brand == "다이소"
    assert ops[0].aspect == "변색"  # 황변 정규화

def test_brand_none_preserved():
    ms = [_match(MatchOutcome.MATCHED, None, "변색")]
    ops = to_opinions(ms, _post(), AspectNormalizer.default())
    assert ops[0].brand is None
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/matching/test_to_opinions.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/matching/domain/to_opinions.py`:

```python
from raw_review.shared.models import RawPost
from raw_review.shared.match_result import MentionMatch, MatchOutcome
from raw_review.shared.verdict import Opinion
from raw_review.matching.domain.aspect_normalizer import AspectNormalizer

def to_opinions(matches: list[MentionMatch], post: RawPost,
                normalizer: AspectNormalizer) -> list[Opinion]:
    out: list[Opinion] = []
    for match in matches:
        if match.outcome != MatchOutcome.MATCHED:
            continue
        for op in match.mention.opinions:
            out.append(Opinion(
                brand=match.mention.brand,
                polarity=op.polarity,
                aspect=normalizer.canonical(op.aspect),
                quote=op.quote,
                source_part=op.source_part,
            ))
    return out
```

> 주: 출처 url은 호출측(워커/리포트)에서 `post.url`로 연결한다. `Opinion`에 url을 넣을지는 1b 리포트 플랜에서 결정한다(현재 `source_part`만 보존). 1b에서 url이 필요하면 `Opinion`에 `url: str` 필드를 추가하고 여기서 `post.url`을 채운다.

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/matching/test_to_opinions.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/matching/domain/to_opinions.py apps/api/tests/matching/test_to_opinions.py
git commit -m "feat: add MentionMatch->Opinion transform"
```

---

### Task 15: 포트와 fake

**Files:**
- Create: `apps/api/src/raw_review/shared/ports.py`
- Create: `apps/api/src/raw_review/adapters/fakes.py`
- Test: `apps/api/tests/adapters/test_fakes.py`

**Interfaces:**
- Consumes: shared 모델들
- Produces:
  - `ParseResult(spec: SearchSpec | None, ambiguous: AmbiguousQuery | None, confidence)`
  - `SpecParser` Protocol: `async parse(query, known_schema=None) -> ParseResult`
  - `ExtractStatus` enum: `OK, EMPTY, UNANALYZED`
  - `ExtractResult(mentions: list[ProductMention], status: ExtractStatus)`
  - `MentionExtractor` Protocol: `async extract(post, spec) -> ExtractResult`
  - `SchemaCache` Protocol: `get/put/classify_category`
  - `FakeSpecParser(result)`, `FakeMentionExtractor(by_url: dict[str, ExtractResult])`

- [ ] **Step 1: 실패 테스트**

`tests/adapters/test_fakes.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.ports import ExtractResult, ExtractStatus
from raw_review.adapters.fakes import FakeMentionExtractor

async def test_fake_extractor_returns_by_url():
    post = RawPost(source="dc", url="u1", title="t", body="b",
                   posted_at=datetime(2026,10,1), likes=0, comments=[])
    spec = SearchSpec(category="양말", constraints=[])
    fake = FakeMentionExtractor({"u1": ExtractResult(mentions=[], status=ExtractStatus.EMPTY)})
    r = await fake.extract(post, spec)
    assert r.status == ExtractStatus.EMPTY
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/adapters/test_fakes.py -v` → FAIL

- [ ] **Step 3: ports.py 구현**

`src/raw_review/shared/ports.py`:

```python
from enum import Enum
from typing import Protocol
from pydantic import BaseModel
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec, AttributeSchema, AmbiguousQuery
from raw_review.shared.mention import ProductMention

class ParseResult(BaseModel):
    spec: SearchSpec | None = None
    ambiguous: AmbiguousQuery | None = None
    confidence: float = 0.0

class SpecParser(Protocol):
    async def parse(self, query: str,
                    known_schema: AttributeSchema | None = None) -> ParseResult: ...

class ExtractStatus(str, Enum):
    OK = "OK"
    EMPTY = "EMPTY"
    UNANALYZED = "UNANALYZED"

class ExtractResult(BaseModel):
    mentions: list[ProductMention] = []
    status: ExtractStatus = ExtractStatus.OK

class MentionExtractor(Protocol):
    async def extract(self, post: RawPost, spec: SearchSpec) -> ExtractResult: ...

class SchemaCache(Protocol):
    def get(self, category: str) -> AttributeSchema | None: ...
    def put(self, schema: AttributeSchema) -> None: ...
    def classify_category(self, query: str) -> str | None: ...
```

- [ ] **Step 4: fakes.py 구현**

`src/raw_review/adapters/fakes.py`:

```python
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec, AttributeSchema
from raw_review.shared.ports import ParseResult, ExtractResult, ExtractStatus

class FakeSpecParser:
    def __init__(self, result: ParseResult):
        self._result = result
    async def parse(self, query: str, known_schema: AttributeSchema | None = None) -> ParseResult:
        return self._result

class FakeMentionExtractor:
    def __init__(self, by_url: dict[str, ExtractResult]):
        self._by_url = by_url
    async def extract(self, post: RawPost, spec: SearchSpec) -> ExtractResult:
        return self._by_url.get(post.url, ExtractResult(mentions=[], status=ExtractStatus.EMPTY))
```

- [ ] **Step 5: 통과 + 경계** — Run: `cd apps/api && pytest tests/adapters/test_fakes.py -v && python -m importlinter.cli lint` → PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/raw_review/shared/ports.py apps/api/src/raw_review/adapters/fakes.py apps/api/tests/adapters/test_fakes.py
git commit -m "feat: add ports and fakes"
```

---

### Task 16: 설정 모듈

**Files:**
- Create: `apps/api/src/raw_review/config.py`
- Test: `apps/api/tests/test_config.py`

**Interfaces:**
- Consumes: 없음
- Produces: `Settings`(env_prefix `RR_`): `openai_api_key`, `model_query="gpt-4o"`, `model_extract="gpt-4o-mini"`, `resolve_confidence_floor=0.6`, `numeric_tolerance=0.0`, `mention_precision_floor=0.95`; `get_settings()`

- [ ] **Step 1: 실패 테스트**

`tests/test_config.py`:

```python
from raw_review.config import Settings

def test_settings(monkeypatch):
    monkeypatch.setenv("RR_OPENAI_API_KEY", "sk")
    monkeypatch.setenv("RR_MODEL_QUERY", "gpt-4o")
    s = Settings()
    assert s.model_query == "gpt-4o"
    assert s.mention_precision_floor == 0.95
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/test_config.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/config.py`:

```python
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="RR_")
    openai_api_key: str = ""
    model_query: str = "gpt-4o"        # 고tier 질의 이해
    model_extract: str = "gpt-4o-mini" # 저tier 추출
    resolve_confidence_floor: float = 0.6
    numeric_tolerance: float = 0.0
    mention_precision_floor: float = 0.95

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/test_config.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/config.py apps/api/tests/test_config.py
git commit -m "feat: add settings with 2-tier model names"
```

---

### Task 17: CategoryResolver + SchemaCache

**Files:**
- Create: `apps/api/src/raw_review/catalog/application/schema_cache.py`
- Create: `apps/api/src/raw_review/catalog/application/resolve_category.py`
- Test: `apps/api/tests/catalog/test_resolve_category.py`

**Interfaces:**
- Consumes: `SearchSpec`/`AttributeSchema`/`AttributeDef`/`AmbiguousQuery` (Task 4), `ParseResult`/`SpecParser` (Task 15), text 유틸 (Task 2)
- Produces:
  - `InMemorySchemaCache` — `get/put/classify_category` (category_aliases 사전으로 추정)
  - `async resolve_category(query, parser, cache, *, confidence_floor=0.6) -> ParseResult` — classify→get→known_schema를 파서에 전달, 성공 시 스키마 put, 낮은 신뢰도면 ambiguous

- [ ] **Step 1: 실패 테스트**

`tests/catalog/test_resolve_category.py`:

```python
from raw_review.shared.search_spec import (
    SearchSpec, AttributeSchema, AttributeDef, AttributeConstraint, MatchMode, AmbiguousQuery)
from raw_review.shared.ports import ParseResult
from raw_review.catalog.application.resolve_category import resolve_category
from raw_review.catalog.application.schema_cache import InMemorySchemaCache

class RecordingParser:
    def __init__(self, result): self._result = result; self.seen = "unset"
    async def parse(self, query, known_schema=None):
        self.seen = known_schema
        return self._result

def _spec():
    return SearchSpec(category="양말", constraints=[
        AttributeConstraint(name="소재", hard=True, match_mode=MatchMode.ENUM, wanted=["울"])])

async def test_high_confidence_caches_schema():
    cache = InMemorySchemaCache()
    p = RecordingParser(ParseResult(spec=_spec(), confidence=0.9))
    r = await resolve_category("울양말", p, cache)
    assert r.spec.category == "양말"
    assert cache.get("양말") is not None

async def test_cache_hit_passes_known_schema(monkeypatch):
    cache = InMemorySchemaCache()
    cache.put(AttributeSchema(category="양말", category_aliases=["삭스"],
              attributes=[AttributeDef(name="소재", hard=True, match_mode=MatchMode.ENUM)]))
    p = RecordingParser(ParseResult(spec=_spec(), confidence=0.9))
    await resolve_category("삭스 추천", p, cache)
    assert p.seen is not None and p.seen.category == "양말"

async def test_low_confidence_ambiguous():
    amb = AmbiguousQuery(query="장갑", options=["자전거용","작업용","오토바이"])
    p = RecordingParser(ParseResult(spec=None, ambiguous=amb, confidence=0.2))
    r = await resolve_category("장갑", p, InMemorySchemaCache())
    assert r.ambiguous is not None
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/catalog/test_resolve_category.py -v` → FAIL

- [ ] **Step 3: schema_cache.py 구현**

`src/raw_review/catalog/application/schema_cache.py`:

```python
from raw_review.shared.search_spec import AttributeSchema
from raw_review.shared.text import normalize

class InMemorySchemaCache:
    def __init__(self):
        self._by_category: dict[str, AttributeSchema] = {}

    def get(self, category: str) -> AttributeSchema | None:
        return self._by_category.get(category)

    def put(self, schema: AttributeSchema) -> None:
        self._by_category[schema.category] = schema

    def classify_category(self, query: str) -> str | None:
        q = normalize(query)
        for category, schema in self._by_category.items():
            names = [category, *schema.category_aliases]
            if any(normalize(n) in q for n in names):
                return category
        return None
```

- [ ] **Step 4: resolve_category.py 구현**

`src/raw_review/catalog/application/resolve_category.py`:

```python
from raw_review.shared.search_spec import AttributeSchema, AttributeDef, AmbiguousQuery
from raw_review.shared.ports import ParseResult

async def resolve_category(query, parser, cache, *, confidence_floor: float = 0.6) -> ParseResult:
    known = None
    category = cache.classify_category(query)
    if category is not None:
        known = cache.get(category)

    result = await parser.parse(query, known_schema=known)

    if result.spec is None or result.confidence < confidence_floor:
        if result.ambiguous is not None:
            return result
        return ParseResult(ambiguous=AmbiguousQuery(query=query, options=[]),
                           confidence=result.confidence)

    schema = AttributeSchema(
        category=result.spec.category,
        attributes=[AttributeDef(name=c.name, hard=c.hard, match_mode=c.match_mode)
                    for c in result.spec.constraints])
    cache.put(schema)
    return result
```

- [ ] **Step 5: 통과 + 경계** — Run: `cd apps/api && pytest tests/catalog/test_resolve_category.py -v && python -m importlinter.cli lint` → PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/raw_review/catalog apps/api/tests/catalog
git commit -m "feat: add CategoryResolver with alias-aware schema cache"
```

---

### Task 18: OpenAI 어댑터 (spec parser + mention extractor)

**Files:**
- Create: `apps/api/src/raw_review/adapters/openai_spec_parser.py`
- Create: `apps/api/src/raw_review/adapters/openai_mention_extractor.py`
- Test: `apps/api/tests/adapters/test_openai_spec_parser.py`, `test_openai_mention_extractor.py`

**Interfaces:**
- Consumes: shared 모델, `ParseResult`/`ExtractResult`/`ExtractStatus` (Task 15)
- Produces:
  - spec parser: `build_spec_messages(query, known_schema) -> list[dict]`, `spec_response_schema() -> dict`, `parse_spec_response(raw) -> ParseResult`, `OpenAiSpecParser(client, model)`
  - extractor: `build_extract_messages(post, spec) -> list[dict]`, `mention_response_schema() -> dict`, `parse_extract_response(raw) -> list[ProductMention]`, `OpenAiMentionExtractor(client, model)` — 빈 결과 1회 재시도, 실패 시 `status=UNANALYZED`
- 주: 네트워크 호출 단위테스트 안 함. 순수 함수(build/schema/parse)만 테스트. strict json_schema는 상위 spec의 `_strict` 재귀 방식 사용.

- [ ] **Step 1: 실패 테스트 (parser 순수함수)**

`tests/adapters/test_openai_spec_parser.py`:

```python
import pytest
from raw_review.adapters.openai_spec_parser import (
    build_spec_messages, spec_response_schema, parse_spec_response)

def test_build_includes_query():
    msgs = build_spec_messages("울양말", None)
    assert any("울양말" in m["content"] for m in msgs)

def test_schema_is_strict_object():
    s = spec_response_schema()
    assert s["type"] == "object" and s["additionalProperties"] is False

def test_parse_valid():
    raw = '{"spec": {"category": "양말", "constraints": []}, "confidence": 0.9, "ambiguous": null}'
    r = parse_spec_response(raw)
    assert r.spec.category == "양말" and r.confidence == 0.9

def test_parse_invalid_raises():
    with pytest.raises(ValueError):
        parse_spec_response("{bad")
```

`tests/adapters/test_openai_mention_extractor.py`:

```python
import pytest
from raw_review.adapters.openai_mention_extractor import (
    build_extract_messages, mention_response_schema, parse_extract_response)
from datetime import datetime
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec

def _post():
    return RawPost(source="dc", url="u", title="울양말 후기", body="보풀 심함",
                   posted_at=datetime(2026,10,1), likes=0, comments=[])

def test_build_includes_body():
    msgs = build_extract_messages(_post(), SearchSpec(category="양말", constraints=[]))
    assert any("보풀" in m["content"] for m in msgs)

def test_parse_mentions():
    raw = '''{"mentions": [{"raw": "데카트론 울양말 보풀", "role": "USED", "brand": "데카트론",
               "attributes": [{"name":"소재","raw_value":"울","normalized_value":"울"}],
               "opinions": [{"polarity":"con","aspect":"보풀","quote":"보풀 심함","source_part":"post"}]}]}'''
    mentions = parse_extract_response(raw)
    assert mentions[0].brand == "데카트론" and mentions[0].role.value == "USED"

def test_parse_invalid_raises():
    with pytest.raises(ValueError):
        parse_extract_response("{bad")
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/adapters/test_openai_spec_parser.py tests/adapters/test_openai_mention_extractor.py -v` → FAIL

- [ ] **Step 3: 공통 strict 유틸 + spec parser 구현**

`src/raw_review/adapters/openai_spec_parser.py`:

```python
import json
from pydantic import BaseModel, ValidationError
from raw_review.shared.search_spec import SearchSpec, AttributeSchema, AmbiguousQuery
from raw_review.shared.ports import ParseResult

def strict(schema: dict) -> dict:
    if schema.get("type") == "object":
        schema["additionalProperties"] = False
        props = schema.get("properties", {})
        if props:
            schema["required"] = list(props.keys())
        for sub in props.values():
            strict(sub)
    for key in ("items",):
        if isinstance(schema.get(key), dict):
            strict(schema[key])
    for key in ("$defs", "anyOf"):
        node = schema.get(key)
        if isinstance(node, dict):
            for d in node.values():
                strict(d)
        elif isinstance(node, list):
            for d in node:
                strict(d)
    return schema

class _SpecEnvelope(BaseModel):
    spec: SearchSpec | None = None
    ambiguous: AmbiguousQuery | None = None
    confidence: float = 0.0

_SYSTEM = (
    "너는 상품 검색어 해석기다. 질의를 카테고리와 구조화 속성 제약으로 바꾼다. "
    "속성마다 match_mode(EXACT/ENUM/SYNONYM/NUMERIC/RANGE/SUBSTRING/FREETEXT)를 정하고, "
    "제외는 속성 단위로 둔다. 애매하면 ambiguous.options로 선택지를 준다. JSON 스키마로만 답한다."
)

def spec_response_schema() -> dict:
    return strict(_SpecEnvelope.model_json_schema())

def build_spec_messages(query: str, known_schema: AttributeSchema | None) -> list[dict]:
    hint = ""
    if known_schema is not None:
        hint = f"\n[알려진 스키마]\n{known_schema.model_dump_json()}"
    return [{"role": "system", "content": _SYSTEM},
            {"role": "user", "content": f"질의: {query}{hint}"}]

def parse_spec_response(raw: str) -> ParseResult:
    try:
        env = _SpecEnvelope.model_validate_json(raw)
    except (ValidationError, ValueError) as e:
        raise ValueError(f"spec 파싱 실패: {e}") from e
    return ParseResult(spec=env.spec, ambiguous=env.ambiguous, confidence=env.confidence)

class OpenAiSpecParser:
    def __init__(self, client, model: str):
        self._client = client; self._model = model
    async def parse(self, query: str, known_schema: AttributeSchema | None = None) -> ParseResult:
        resp = await self._client.chat.completions.create(
            model=self._model, temperature=0,
            messages=build_spec_messages(query, known_schema),
            response_format={"type": "json_schema", "json_schema": {
                "name": "spec_envelope", "schema": spec_response_schema(), "strict": True}})
        return parse_spec_response(resp.choices[0].message.content)
```

- [ ] **Step 4: mention extractor 구현**

`src/raw_review/adapters/openai_mention_extractor.py`:

```python
import json
from pydantic import BaseModel, ValidationError
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.mention import ProductMention
from raw_review.shared.ports import ExtractResult, ExtractStatus
from raw_review.adapters.openai_spec_parser import strict

class _Envelope(BaseModel):
    mentions: list[ProductMention] = []

_SYSTEM = (
    "너는 후기 글에서 상품 언급을 추출한다. 브랜드마다 언급을 하나씩 만들고, "
    "role(USED/COMPARED/REJECTED_ALT/MENTIONED)을 정한다. 비교 글은 언급을 쪼갠다. "
    "속성값은 normalized_value에 canonical로 정규화해 넣는다. 단점 인용은 원문 그대로. JSON 스키마로만 답한다."
)

def mention_response_schema() -> dict:
    return strict(_Envelope.model_json_schema())

def build_extract_messages(post: RawPost, spec: SearchSpec) -> list[dict]:
    comments = "\n".join(f"- {c.text}" for c in post.comments)
    user = (f"[검색 조건]\n{spec.model_dump_json()}\n\n"
            f"[글]\n제목: {post.title}\n본문: {post.body}\n별점: {post.rating}\n댓글:\n{comments}")
    return [{"role": "system", "content": _SYSTEM}, {"role": "user", "content": user}]

def parse_extract_response(raw: str) -> list[ProductMention]:
    try:
        return _Envelope.model_validate_json(raw).mentions
    except (ValidationError, ValueError) as e:
        raise ValueError(f"추출 파싱 실패: {e}") from e

class OpenAiMentionExtractor:
    def __init__(self, client, model: str):
        self._client = client; self._model = model

    async def _call(self, post: RawPost, spec: SearchSpec) -> list[ProductMention]:
        resp = await self._client.chat.completions.create(
            model=self._model, temperature=0,
            messages=build_extract_messages(post, spec),
            response_format={"type": "json_schema", "json_schema": {
                "name": "mentions", "schema": mention_response_schema(), "strict": True}})
        return parse_extract_response(resp.choices[0].message.content)

    async def extract(self, post: RawPost, spec: SearchSpec) -> ExtractResult:
        for attempt in range(2):  # 1회 재시도
            try:
                mentions = await self._call(post, spec)
            except ValueError:
                if attempt == 1:
                    return ExtractResult(mentions=[], status=ExtractStatus.UNANALYZED)
                continue
            if mentions:
                return ExtractResult(mentions=mentions, status=ExtractStatus.OK)
            if attempt == 1:
                return ExtractResult(mentions=[], status=ExtractStatus.UNANALYZED)
        return ExtractResult(mentions=[], status=ExtractStatus.UNANALYZED)
```

> 주: strict json_schema가 특정 모델에서 거부되면 `json_object` + 수동 검증으로 폴백하고 사유를 커밋에 남긴다(상위 spec과 동일 방침).

- [ ] **Step 5: 통과** — Run: `cd apps/api && pytest tests/adapters -v` → PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/raw_review/adapters/openai_spec_parser.py apps/api/src/raw_review/adapters/openai_mention_extractor.py apps/api/tests/adapters/test_openai_spec_parser.py apps/api/tests/adapters/test_openai_mention_extractor.py
git commit -m "feat: add OpenAI spec parser and mention extractor"
```

---

### Task 19: 평가 지표 + 카테고리 분산 골든셋

**Files:**
- Create: `apps/api/src/raw_review/eval/metrics.py`
- Create: `apps/api/tests/eval/golden_cases.jsonl`
- Create: `apps/api/tests/eval/conftest.py`
- Create: `apps/api/tests/eval/test_matching_eval.py`

**Interfaces:**
- Consumes: `match_mention` (Task 12), shared 모델, `Settings` (Task 16)
- Produces:
  - `precision(tp, fp)`, `recall(tp, fn)`
  - `evaluate(cases) -> EvalReport(mention_precision, mention_recall, excluded_fp, target_fn, n)` — 각 케이스는 라벨된 언급 + 기대 채택 여부
  - `by_category(cases) -> dict[str, EvalReport]`
  - `comparison_only(cases) -> EvalReport`

- [ ] **Step 1: metrics 실패 테스트**

`tests/eval/test_matching_eval.py`:

```python
from raw_review.eval.metrics import precision, recall

def test_precision_recall():
    assert precision(9, 1) == 0.9
    assert recall(8, 2) == 0.8
    assert precision(0, 0) == 1.0
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/eval/test_matching_eval.py -v` → FAIL

- [ ] **Step 3: metrics 구현**

`src/raw_review/eval/metrics.py`:

```python
from pydantic import BaseModel

def precision(tp: int, fp: int) -> float:
    return 1.0 if (tp + fp) == 0 else tp / (tp + fp)

def recall(tp: int, fn: int) -> float:
    return 1.0 if (tp + fn) == 0 else tp / (tp + fn)

class EvalReport(BaseModel):
    mention_precision: float
    mention_recall: float
    excluded_fp: int   # 비대상/제외 언급을 채택한 수
    target_fn: int     # 대상(USED+일치) 언급을 놓친 수
    n: int

def evaluate(cases) -> EvalReport:
    tp = fp = fn = excluded_fp = target_fn = 0
    for c in cases:
        predicted = c["predicted_matched"]
        expected = c["expected_matched"]
        if predicted and expected:
            tp += 1
        elif predicted and not expected:
            fp += 1
            excluded_fp += 1
        elif not predicted and expected:
            fn += 1
            target_fn += 1
    return EvalReport(mention_precision=precision(tp, fp), mention_recall=recall(tp, fn),
                      excluded_fp=excluded_fp, target_fn=target_fn, n=len(cases))

def by_category(cases) -> dict:
    cats: dict[str, list] = {}
    for c in cases:
        cats.setdefault(c["category_group"], []).append(c)
    return {g: evaluate(cs) for g, cs in cats.items()}

def comparison_only(cases) -> EvalReport:
    return evaluate([c for c in cases if c.get("is_comparison")])
```

- [ ] **Step 4: 골든셋 작성**

`tests/eval/golden_cases.jsonl` — 카테고리군(휴대폰/액세서리, 의류/신발, 스포츠용품, 생활가전, 기타) × 상황(정상·유사·제외·비교·애매·광고) 분포로 손 라벨. 각 줄: `category_group`, `is_comparison`, `spec`, `mention`(ProductMention dict), `expected_matched`(bool). 각 카테고리군 최소 4건, 총 20건+. 비교 글 케이스 최소 4건. 예시 3줄:

```json
{"category_group": "의류/신발", "is_comparison": true, "spec": {"category": "양말", "constraints": [{"name": "소재", "hard": true, "match_mode": "ENUM", "wanted": ["울"], "aliases": {"울": ["메리노"]}}]}, "mention": {"raw": "면양말은 보풀 심했다", "role": "COMPARED", "brand": "무지", "attributes": [{"name": "소재", "raw_value": "면", "normalized_value": "면"}], "opinions": []}, "expected_matched": false}
{"category_group": "의류/신발", "is_comparison": true, "spec": {"category": "양말", "constraints": [{"name": "소재", "hard": true, "match_mode": "ENUM", "wanted": ["울"], "aliases": {"울": ["메리노"]}}]}, "mention": {"raw": "울양말은 따뜻했다", "role": "USED", "brand": "데카트론", "attributes": [{"name": "소재", "raw_value": "메리노", "normalized_value": "울"}], "opinions": [{"polarity": "pro", "aspect": "보온", "quote": "따뜻", "source_part": "post"}]}, "expected_matched": true}
{"category_group": "휴대폰/액세서리", "is_comparison": true, "spec": {"category": "휴대폰케이스", "constraints": [{"name": "대상기기", "hard": true, "match_mode": "EXACT", "wanted": ["아이폰17"], "exclusions": ["17 Pro"]}, {"name": "형태", "hard": true, "match_mode": "ENUM", "wanted": ["투명"], "aliases": {"투명": ["클리어"]}}]}, "mention": {"raw": "17 Pro는 안 맞았다", "role": "REJECTED_ALT", "brand": "슈피겐", "attributes": [{"name": "대상기기", "raw_value": "아이폰17 Pro", "normalized_value": "아이폰17 Pro"}, {"name": "형태", "raw_value": "투명", "normalized_value": "투명"}], "opinions": []}, "expected_matched": false}
```

> 주: 20건 이상을 손으로 채운다. 분포 균형(카테고리군·상황)을 지킨다. 2단계 수집 후 실제 라벨로 확장하고 LLM precision/recall 게이트(100건+)를 추가한다.

- [ ] **Step 5: 로더 + 평가 테스트**

`tests/eval/conftest.py`:

```python
import json
from pathlib import Path
import pytest
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.mention import ProductMention

@pytest.fixture
def golden_cases():
    path = Path(__file__).parent / "golden_cases.jsonl"
    rows = [json.loads(l) for l in path.read_text(encoding="utf-8").splitlines() if l.strip()]
    for r in rows:
        r["spec_obj"] = SearchSpec.model_validate(r["spec"])
        r["mention_obj"] = ProductMention.model_validate(r["mention"])
    return rows
```

`tests/eval/test_matching_eval.py`에 추가:

```python
from raw_review.shared.match_result import MatchOutcome
from raw_review.matching.domain.mention_matcher import match_mention
from raw_review.eval.metrics import evaluate, by_category, comparison_only
from raw_review.config import get_settings

def _run(golden_cases):
    rows = []
    for c in golden_cases:
        outcome = match_mention(c["mention_obj"], c["spec_obj"]).outcome
        rows.append({**c, "predicted_matched": outcome == MatchOutcome.MATCHED})
    return rows

def test_dataset_distribution(golden_cases):
    assert len(golden_cases) >= 20
    groups = {c["category_group"] for c in golden_cases}
    assert len(groups) >= 4
    assert sum(1 for c in golden_cases if c.get("is_comparison")) >= 4

def test_mention_precision_meets_floor(golden_cases):
    report = evaluate(_run(golden_cases))
    assert report.mention_precision >= get_settings().mention_precision_floor
    assert report.excluded_fp == 0  # 제외/비대상 언급을 채택하지 않음

def test_comparison_cases_no_leak(golden_cases):
    report = comparison_only(_run(golden_cases))
    assert report.excluded_fp == 0

def test_per_category_available(golden_cases):
    assert len(by_category(_run(golden_cases))) >= 4
```

- [ ] **Step 6: 통과 + 전체 + 경계** — Run: `cd apps/api && pytest -v && python -m importlinter.cli lint` → 전부 PASS

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/raw_review/eval apps/api/tests/eval
git commit -m "feat: add matching eval metrics and category-distributed golden set"
```

---

## Self-Review 결과

**1. Spec coverage**
- 구조화 AttributeSchema/SearchSpec + match_mode validator: Task 4 ✓
- MatchMode 7종 매처(EXACT 경계/ENUM/SYNONYM 방향/NUMERIC·RANGE 단위/SUBSTRING·FREETEXT): Task 7-10 ✓
- ProductMention(role, ExtractedAttribute): Task 5 ✓
- CoarseGate: Task 11 ✓
- MentionMatcher(role 필터, USED 제외, 하드 누락 UNCERTAIN): Task 12 ✓
- Opinion 변환 계약: Task 14 ✓
- CategoryResolver(캐시 읽기, 별칭): Task 17 ✓
- 2-tier OpenAI 어댑터(json_schema, 빈추출 재시도→UNANALYZED): Task 18 ✓
- eval 지표 함수(precision/recall/excluded_fp/target_fn/카테고리별/비교전용) + 분산 골든셋: Task 19 ✓
- 공유커널 경계: Task 1, 재확인 15/17/19 ✓

**2. 리뷰 10개 지적 반영**
1. ProductMention 숫자/범위 → `ExtractedAttribute`(numeric/lower/upper/unit): Task 5 ✓
2. FREETEXT 충돌 → LLM canonical 변환, 없으면 UNCERTAIN, soft 전용: Task 4(validator), 10 ✓
3. 추출 누락 → recall 우선 + 빈배열 1회 재시도 + UNANALYZED: Task 18 ✓
4. 제외·부정·비교 → role 필터, 제외는 USED subject에만: Task 12 ✓
5. SYNONYM 방향 → canonical 동등, 추출이 방향 처리: Task 8 ✓
6. SUBSTRING hard 금지 → validator: Task 4 ✓
7. Opinion 변환 계약 → `to_opinions`, brand None 버킷, 출처: Task 14 ✓
8. 평가 지표 함수 → Task 19 ✓
9. 하드 누락 REJECT/UNCERTAIN → UNCERTAIN 확정: Task 12 ✓

**3. Placeholder scan**: 모든 코드 스텝에 실제 코드. golden_cases.jsonl은 포맷 예시 3줄 + 20건 분산 지침. to_opinions의 url은 1b 종속으로 명시 ✓

**4. Type consistency**: `ExtractedAttribute`, `ProductMention`, `MentionMatch`, `MatchOutcome`, `AttrMatch`, `SearchSpec`, `AttributeConstraint`, `ParseResult`, `ExtractResult`가 정의(shared/ports)와 소비(matching/catalog/adapters/eval) 전반 동일 ✓
