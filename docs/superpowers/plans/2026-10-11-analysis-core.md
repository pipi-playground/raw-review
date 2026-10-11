# 분석 코어 Implementation Plan

> **⚠️ SUPERSEDED (2026-10-11):** 이 플랜의 catalog/analysis 설계(문자열 딕셔너리 SearchSpec, 문서 전체 문자열 RuleFilter)는 리뷰에서 상품 일반화·언급 단위 매칭을 표현하지 못하는 것으로 판명되어 폐기되었다. 구조화 속성 모델과 언급 단위 매칭은 `docs/superpowers/specs/2026-10-11-matching-engine-design.md`로 재설계되었다. **이 플랜대로 구현하지 말 것.** 매칭 엔진 spec 승인 후 새 플랜으로 대체한다.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 질의를 SearchSpec으로 해석하고, 수집된 글을 룰 + LLM으로 판별해 브랜드별 장단점 리포트를 만드는 순수 분석 코어를 만든다. 수집·API·DB 없이, 고정 입력과 FakeClassifier로 완결된다.

**Architecture:** 경계를 넘는 값 객체와 포트는 `shared` 공유 커널에 둔다. catalog(질의 해석), analysis(판별·추출), review(집계)는 서로를 import하지 않고 `shared`만 공유한다. LLM은 `LlmClassifier` 포트 뒤에 두고, 테스트는 `FakeClassifier`, 실사용은 `OpenAiClassifier`를 쓴다. 도메인은 순수 함수 위주라 DB·네트워크 없이 단위테스트가 된다.

**Tech Stack:** Python 3.12, Pydantic v2, pytest, OpenAI SDK, import-linter. (DB·FastAPI·워커는 3단계 플랜)

**Spec:** `docs/superpowers/specs/2026-10-11-raw-review-design.md`

## Global Constraints

- Python 3.12, Pydantic v2.
- **공유 커널 규칙**: 경계를 넘는 계약(`RawPost`, `SearchSpec`, `PostVerdict`, `AnalyzedPost`, 포트, 텍스트 정규화)은 `shared/`에 둔다. catalog·analysis·review는 서로를 import하지 않고 `shared`만 import한다. 각 `domain/`은 `adapters`·`openai`를 import하지 않는다. import-linter로 강제한다.
- 단점 인용(`quote`)은 원문 표현을 보존하되 최대 120자로 자른다.
- 작성자 식별정보는 입력 모델에 담지 않는다(수집 단계에서 제거된 것으로 간주).
- LLM 어댑터는 **JSON Schema 구조화 출력**(`response_format.type = "json_schema"`, `strict: true`)을 쓴다. schema 검증 실패 시 1회 재시도 후 해당 글을 `UNANALYZED` 처리.
- 광고 판정은 **룰 신호 점수와 LLM 판정을 OR로 합친다.** 둘 중 하나라도 광고로 보면 광고로 처리한다.
- 모든 임계값은 설정값(`config.py`)으로 둔다. 하드코딩 금지. 성공 기준: 관련성 precision ≥ 0.95, 광고 recall ≥ 0.9.
- **LLM 품질 게이트(100건 이상 실제 라벨로 precision/recall 측정)는 2단계(수집)에 종속된다.** 1단계에서는 실제 글이 없으므로, 룰 계층(RuleFilter·광고 신호)만 손으로 라벨링한 실제형 케이스로 oracle 없이 검증한다.

## Review Focus

- **제외 변형만 언급한 글**: "17 Pro" 글이 "아이폰17" 결과에 섞이면 안 된다. 정규화 후 부분 문자열(`아이폰17` ⊂ `아이폰17pro`)로 대상 모델을 오인하지 않도록, 제외 변형이 있으면 모델 존재 여부와 무관하게 먼저 REJECT. (Task 5)
- **광고가 룰로만 잡히는 글**: 본문에 "무상 제공", 댓글에 "뒷광고"가 있는데 LLM이 광고로 못 보는 경우, 룰 점수로 걸러야 한다. (Task 8)
- **빈 수집/전부 제외**: 글 0건이거나 모두 REJECT면 brands=[]인 리포트를 반환하고 터지지 않아야 한다. (Task 10)
- **LLM이 스키마 위반 응답**: 재시도 1회 후 예외 대신 `UNANALYZED`로 그 글만 격리. (Task 8)
- **같은 뜻 다른 표현의 단점**: "황변"과 "변색"이 한 aspect로 병합돼야 집계가 맞다. (Task 9)

---

## File Structure

```
apps/api/
  pyproject.toml
  setup.cfg                           # import-linter 계약
  src/raw_review/
    __init__.py
    config.py                         # 설정값(임계값, 모델명, 상한)
    shared/                           # 공유 커널: 경계를 넘는 계약
      __init__.py
      text.py                         # normalize() 텍스트 정규화
      models.py                       # RawPost, RawComment
      search_spec.py                  # SearchSpec, AttributeSchema, Attribute, AmbiguousQuery
      verdict.py                      # Polarity, Opinion, RelevanceVerdict, AdVerdict, PostVerdict
      analyzed.py                     # AnalyzedPost, AnalyzedStatus
      ports.py                        # LlmClassifier, SpecParser, SchemaCache, ParseResult
    catalog/
      __init__.py
      application/
        resolve_category.py           # CategoryResolver 유스케이스
        schema_cache.py               # InMemorySchemaCache 구현
    analysis/
      __init__.py
      domain/
        rule_filter.py                # 제외어·키워드 룰
        ad_signals.py                 # 광고 신호 점수
        aspect_normalizer.py          # aspect 동의어 정규화
      application/
        analyze_post.py               # 글 1건 분석(룰 → 광고점수 → LLM)
    review/
      __init__.py
      domain/
        report.py                     # ProductReport 등 모델
        report_builder.py             # 브랜드별 집계
    adapters/
      __init__.py
      openai_classifier.py            # OpenAiClassifier (json_schema)
      fake_classifier.py              # FakeClassifier
    eval/
      __init__.py
      metrics.py                      # precision/recall
  tests/
    __init__.py
    test_boundaries.py
    test_config.py
    shared/__init__.py
      test_text.py
      test_models.py
      test_search_spec.py
      test_verdict.py
    catalog/__init__.py
      test_resolve_category.py
    analysis/__init__.py
      test_rule_filter.py
      test_ad_signals.py
      test_aspect_normalizer.py
      test_analyze_post.py
    review/__init__.py
      test_report_builder.py
    adapters/__init__.py
      test_openai_classifier.py
    eval/__init__.py
      rule_cases.jsonl                # 손 라벨링 실제형 케이스(룰 계층 검증용)
      conftest.py
      test_rule_eval.py
```

---

### Task 1: 스캐폴드와 경계 계약

**Files:**
- Create: `apps/api/pyproject.toml`, `apps/api/setup.cfg`
- Create: 위 File Structure의 모든 패키지 `__init__.py` (빈 파일)
- Test: `apps/api/tests/test_boundaries.py`

**Interfaces:**
- Consumes: 없음
- Produces: 설치 가능한 `raw_review` 패키지, import-linter 계약

- [ ] **Step 1: pyproject.toml 작성**

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

- [ ] **Step 2: import-linter 계약 작성 (setup.cfg)**

```ini
[importlinter]
root_package = raw_review

[importlinter:contract:context-independence]
name = catalog/analysis/review는 서로 import하지 않는다
type = independence
modules =
    raw_review.catalog
    raw_review.analysis
    raw_review.review

[importlinter:contract:domain-pure]
name = domain은 adapters/openai를 모른다
type = forbidden
source_modules =
    raw_review.analysis.domain
    raw_review.review.domain
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
    raw_review.analysis
    raw_review.review
    raw_review.adapters
    openai
```

- [ ] **Step 3: 빈 패키지 생성**

위 File Structure의 모든 `__init__.py`를 빈 파일로 만든다.

- [ ] **Step 4: 경계 테스트 (실패 확인)**

`tests/test_boundaries.py`:

```python
import subprocess, sys

def test_import_contracts_hold():
    result = subprocess.run(
        [sys.executable, "-m", "importlinter.cli", "lint"],
        cwd="apps/api", capture_output=True, text=True,
    )
    assert result.returncode == 0, result.stdout + result.stderr
```

- [ ] **Step 5: 설치 후 실행 (통과 확인)**

Run: `cd apps/api && pip install -e ".[dev]" && pytest tests/test_boundaries.py -v`
Expected: PASS (빈 패키지라 위반 없음)

- [ ] **Step 6: Commit**

```bash
git add apps/api
git commit -m "chore: scaffold api package with shared-kernel boundaries"
```

---

### Task 2: 텍스트 정규화 (shared/text.py)

catalog·analysis 모두 쓰므로 공유 커널에 둔다.

**Files:**
- Create: `apps/api/src/raw_review/shared/text.py`
- Test: `apps/api/tests/shared/test_text.py`

**Interfaces:**
- Consumes: 없음
- Produces: `normalize(text: str) -> str` — 공백 제거 + 소문자화

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_text.py`:

```python
from raw_review.shared.text import normalize

def test_normalize_collapses_space_and_case():
    assert normalize("아이폰 17") == normalize("아이폰17")
    assert normalize("iPhone 17") == normalize("iphone17")
    assert normalize("  슈피겐  ") == "슈피겐"
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_text.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/shared/text.py`:

```python
import re

_SPACE = re.compile(r"\s+")

def normalize(text: str) -> str:
    return _SPACE.sub("", text.strip()).lower()
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/shared/test_text.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/shared/text.py apps/api/tests/shared/test_text.py
git commit -m "feat: add text normalization in shared kernel"
```

---

### Task 3: 입력 계약 모델 (shared/models.py)

**Files:**
- Create: `apps/api/src/raw_review/shared/models.py`
- Test: `apps/api/tests/shared/test_models.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `RawComment(text: str, likes: int = 0)`
  - `RawPost(source: str, url: str, title: str, body: str, posted_at: datetime | None, rating: int | None, likes: int, comments: list[RawComment])`

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_models.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost, RawComment

def test_rawpost_minimal_fields():
    post = RawPost(
        source="dcinside", url="https://gall.dcinside.com/iphone/123",
        title="아이폰17 투명케이스 후기", body="2주 쓰니까 누렇게 떴음",
        posted_at=datetime(2026, 10, 1), rating=None, likes=5,
        comments=[RawComment(text="그거 변색 심하던데", likes=3)],
    )
    assert post.comments[0].text == "그거 변색 심하던데"
    assert post.rating is None
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
    rating: int | None = None  # 쇼핑 리뷰 별점 1~5, 커뮤니티는 None
    likes: int = 0
    comments: list[RawComment] = []
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/shared/test_models.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/shared/models.py apps/api/tests/shared/test_models.py
git commit -m "feat: add RawPost/RawComment input contract"
```

---

### Task 4: SearchSpec (shared/search_spec.py)

경계를 넘는 계약이라 공유 커널에 둔다.

**Files:**
- Create: `apps/api/src/raw_review/shared/search_spec.py`
- Test: `apps/api/tests/shared/test_search_spec.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `Attribute(name: str, value: str = "", hard: bool)`
  - `AttributeSchema(category: str, attributes: list[Attribute])`
  - `SearchSpec(category, hard_attributes: dict[str,str], soft_attributes: dict[str,str], exclusion_variants: list[str], aliases: dict[str,list[str]])`
  - `SearchSpec.all_keywords() -> set[str]`
  - `AmbiguousQuery(query: str, options: list[str])`

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_search_spec.py`:

```python
from raw_review.shared.search_spec import SearchSpec

def test_all_keywords_lowercases_values_and_aliases():
    spec = SearchSpec(
        category="휴대폰케이스",
        hard_attributes={"대상기기": "아이폰17", "형태": "투명"},
        exclusion_variants=["17 Pro", "가죽"],
        aliases={"아이폰17": ["아이폰 17", "iPhone17"]},
    )
    kw = spec.all_keywords()
    assert "아이폰17" in kw
    assert "iphone17" in kw
    assert "투명" in kw
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_search_spec.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/shared/search_spec.py`:

```python
from pydantic import BaseModel

class Attribute(BaseModel):
    name: str
    value: str = ""
    hard: bool

class AttributeSchema(BaseModel):
    category: str
    attributes: list[Attribute]

class SearchSpec(BaseModel):
    category: str
    hard_attributes: dict[str, str]
    soft_attributes: dict[str, str] = {}
    exclusion_variants: list[str] = []
    aliases: dict[str, list[str]] = {}

    def all_keywords(self) -> set[str]:
        kw: set[str] = set()
        for value in self.hard_attributes.values():
            kw.add(value.lower())
        for variants in self.aliases.values():
            for v in variants:
                kw.add(v.lower())
        return kw

class AmbiguousQuery(BaseModel):
    query: str
    options: list[str]
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/shared/test_search_spec.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/shared/search_spec.py apps/api/tests/shared/test_search_spec.py
git commit -m "feat: add SearchSpec in shared kernel"
```

---

### Task 5: RuleFilter (제외어 우선 판정)

**Fix #2**: 제외 변형이 본문에 있으면 모델 존재 여부와 무관하게 먼저 REJECT. 부분 문자열 오인(`아이폰17` ⊂ `아이폰17pro`)을 막는다.

**Files:**
- Create: `apps/api/src/raw_review/analysis/domain/rule_filter.py`
- Test: `apps/api/tests/analysis/test_rule_filter.py`

**Interfaces:**
- Consumes: `RawPost` (Task 3), `SearchSpec` (Task 4), `normalize` (Task 2)
- Produces:
  - `FilterOutcome` enum: `PASS`, `REJECT`, `UNCERTAIN`
  - `FilterResult(outcome, reason: str | None)`
  - `apply_rule_filter(post: RawPost, spec: SearchSpec) -> FilterResult`

- [ ] **Step 1: 실패 테스트**

`tests/analysis/test_rule_filter.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.analysis.domain.rule_filter import apply_rule_filter, FilterOutcome

def _spec():
    return SearchSpec(
        category="휴대폰케이스",
        hard_attributes={"대상기기": "아이폰17", "형태": "투명"},
        exclusion_variants=["17 Pro", "17 Air"],
        aliases={"아이폰17": ["아이폰 17", "iphone17"], "투명": ["클리어"]})

def _post(title, body):
    return RawPost(source="dc", url="u", title=title, body=body,
                   posted_at=datetime(2026,10,1), likes=0, comments=[])

def test_reject_when_exclusion_variant_present_even_if_model_substring_matches():
    # '아이폰17'이 '아이폰 17 Pro' 정규화값의 부분문자열이어도 제외돼야 함
    post = _post("아이폰 17 Pro 투명케이스 후기", "프로 모델 클리어 케이스 씀")
    assert apply_rule_filter(post, _spec()).outcome == FilterOutcome.REJECT

def test_reject_when_no_hard_keyword():
    post = _post("갤럭시 범퍼 케이스", "그냥 잡담")
    assert apply_rule_filter(post, _spec()).outcome == FilterOutcome.REJECT

def test_pass_when_model_and_form_present():
    post = _post("아이폰17 투명케이스 한달 후기", "클리어 케이스 변색됨")
    assert apply_rule_filter(post, _spec()).outcome == FilterOutcome.PASS

def test_uncertain_when_model_present_but_form_missing():
    post = _post("아이폰17 케이스 추천", "케이스 뭐 쓰세요")
    assert apply_rule_filter(post, _spec()).outcome == FilterOutcome.UNCERTAIN
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/analysis/test_rule_filter.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/analysis/domain/rule_filter.py`:

```python
from enum import Enum
from pydantic import BaseModel
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.text import normalize

class FilterOutcome(str, Enum):
    PASS = "PASS"
    REJECT = "REJECT"
    UNCERTAIN = "UNCERTAIN"

class FilterResult(BaseModel):
    outcome: FilterOutcome
    reason: str | None = None

def _contains_any(haystack: str, needles: list[str]) -> bool:
    return any(normalize(n) in haystack for n in needles if n)

def apply_rule_filter(post: RawPost, spec: SearchSpec) -> FilterResult:
    text = normalize(post.title + " " + post.body)

    # Fix #2: 제외 변형이 있으면 모델 존재 여부와 무관하게 먼저 REJECT.
    # '17 Pro'→'17pro'가 Pro 글에 잡히고, 정상 '아이폰17 투명케이스'엔 없음.
    if _contains_any(text, spec.exclusion_variants):
        return FilterResult(outcome=FilterOutcome.REJECT, reason="제외 변형 포함")

    # hard 속성별로 '값 + 별칭' 중 하나라도 본문에 있는지
    present = {
        name: _contains_any(text, [value, *spec.aliases.get(value, [])])
        for name, value in spec.hard_attributes.items()
    }

    if not any(present.values()):
        return FilterResult(outcome=FilterOutcome.REJECT, reason="hard 키워드 없음")
    if all(present.values()):
        return FilterResult(outcome=FilterOutcome.PASS)
    return FilterResult(outcome=FilterOutcome.UNCERTAIN, reason="일부 hard 속성만 일치")
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/analysis/test_rule_filter.py -v` → PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/analysis/domain/rule_filter.py apps/api/tests/analysis/test_rule_filter.py
git commit -m "feat: add rule filter with exclusion-first rejection"
```

---

### Task 6: 광고 신호 점수

**Files:**
- Create: `apps/api/src/raw_review/analysis/domain/ad_signals.py`
- Test: `apps/api/tests/analysis/test_ad_signals.py`

**Interfaces:**
- Consumes: `RawPost` (Task 3)
- Produces:
  - `AdSignalScore(score: float, hits: list[str])` — score 0.0~1.0
  - `score_ad_signals(post: RawPost) -> AdSignalScore`

- [ ] **Step 1: 실패 테스트**

`tests/analysis/test_ad_signals.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost, RawComment
from raw_review.analysis.domain.ad_signals import score_ad_signals

def _post(body, comments=None):
    return RawPost(source="dc", url="u", title="t", body=body,
                   posted_at=datetime(2026,10,1), likes=0, comments=comments or [])

def test_sponsorship_phrase_raises_score():
    clean = score_ad_signals(_post("그냥 변색 후기임"))
    ad = score_ad_signals(_post("업체로부터 제품을 무상 제공받아 작성한 후기입니다"))
    assert ad.score > clean.score
    assert any("제공" in h for h in ad.hits)

def test_comment_calling_out_ad_counts():
    result = score_ad_signals(_post("좋아요 최고예요", [RawComment(text="이거 뒷광고네")]))
    assert result.score > 0
    assert any("광고" in h for h in result.hits)
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/analysis/test_ad_signals.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/analysis/domain/ad_signals.py`:

```python
from pydantic import BaseModel
from raw_review.shared.models import RawPost

class AdSignalScore(BaseModel):
    score: float
    hits: list[str]

_BODY_SIGNALS: list[tuple[str, float]] = [
    ("무상 제공", 0.5), ("제공받아", 0.5), ("협찬", 0.5), ("체험단", 0.5),
    ("원고료", 0.5), ("소정의", 0.3), ("쿠폰코드", 0.3), ("할인코드", 0.3),
    ("제휴", 0.2), ("bit.ly", 0.2), ("coupa.ng", 0.4),
]
_COMMENT_SIGNALS: list[tuple[str, float]] = [
    ("뒷광고", 0.4), ("광고", 0.3), ("협찬", 0.3),
]

def score_ad_signals(post: RawPost) -> AdSignalScore:
    score = 0.0
    hits: list[str] = []
    text = post.title + " " + post.body
    for phrase, weight in _BODY_SIGNALS:
        if phrase in text:
            score += weight
            hits.append(phrase)
    for comment in post.comments:
        for phrase, weight in _COMMENT_SIGNALS:
            if phrase in comment.text:
                score += weight
                hits.append(f"댓글:{phrase}")
    return AdSignalScore(score=min(score, 1.0), hits=hits)
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/analysis/test_ad_signals.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/analysis/domain/ad_signals.py apps/api/tests/analysis/test_ad_signals.py
git commit -m "feat: add ad signal scoring"
```

---

### Task 7: 판별 모델·AnalyzedPost·포트 (shared) + FakeClassifier

**Fix #6**: `AnalyzedPost`에 `source`/`posted_at`/`likes`를 실어 리포트 정렬에 쓴다. 상태에 `AD_REJECTED` 추가.

**Files:**
- Create: `apps/api/src/raw_review/shared/verdict.py`
- Create: `apps/api/src/raw_review/shared/analyzed.py`
- Create: `apps/api/src/raw_review/shared/ports.py`
- Create: `apps/api/src/raw_review/adapters/fake_classifier.py`
- Test: `apps/api/tests/shared/test_verdict.py`

**Interfaces:**
- Consumes: `RawPost` (Task 3), `SearchSpec` (Task 4)
- Produces:
  - `Polarity` enum: `PRO`, `CON`
  - `Opinion(brand, product_line: str | None, polarity, aspect, quote, source_part)` — `quote`는 120자로 자른다.
  - `RelevanceVerdict(matches: bool, matched_attributes: list[str], reason: str)`
  - `AdVerdict(is_ad: bool, confidence: float, evidence: list[str])`
  - `PostVerdict(relevance, ad, opinions: list[Opinion])`
  - `AnalyzedStatus` enum: `ANALYZED`, `REJECTED`, `AD_REJECTED`, `UNANALYZED`
  - `AnalyzedPost(url, source, posted_at: datetime | None, likes: int, status: AnalyzedStatus, verdict: PostVerdict | None, reason: str | None)`
  - `LlmClassifier` 포트(Protocol, async): `judge(post, spec) -> PostVerdict`
  - `FakeClassifier(by_url: dict[str, PostVerdict])`

- [ ] **Step 1: 실패 테스트**

`tests/shared/test_verdict.py`:

```python
from raw_review.shared.verdict import Opinion, Polarity

def test_quote_truncated_to_120_chars():
    op = Opinion(brand="다이소", product_line=None, polarity=Polarity.CON,
                 aspect="변색", quote="변색"*100, source_part="post")
    assert len(op.quote) == 120
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/shared/test_verdict.py -v` → FAIL

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
    brand: str
    product_line: str | None = None
    polarity: Polarity
    aspect: str
    quote: str
    source_part: str  # "post" | "comment"

    @field_validator("quote")
    @classmethod
    def _truncate(cls, v: str) -> str:
        return v[:QUOTE_MAX]

class RelevanceVerdict(BaseModel):
    matches: bool
    matched_attributes: list[str] = []
    reason: str = ""

class AdVerdict(BaseModel):
    is_ad: bool
    confidence: float = 0.0
    evidence: list[str] = []

class PostVerdict(BaseModel):
    relevance: RelevanceVerdict
    ad: AdVerdict
    opinions: list[Opinion] = []
```

- [ ] **Step 4: analyzed.py 구현**

`src/raw_review/shared/analyzed.py`:

```python
from datetime import datetime
from enum import Enum
from pydantic import BaseModel
from raw_review.shared.verdict import PostVerdict

class AnalyzedStatus(str, Enum):
    ANALYZED = "ANALYZED"
    REJECTED = "REJECTED"
    AD_REJECTED = "AD_REJECTED"
    UNANALYZED = "UNANALYZED"

class AnalyzedPost(BaseModel):
    url: str
    source: str
    posted_at: datetime | None = None
    likes: int = 0
    status: AnalyzedStatus
    verdict: PostVerdict | None = None
    reason: str | None = None
```

- [ ] **Step 5: ports.py 구현**

`src/raw_review/shared/ports.py`:

```python
from typing import Protocol
from pydantic import BaseModel
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec, AttributeSchema, AmbiguousQuery
from raw_review.shared.verdict import PostVerdict

class LlmClassifier(Protocol):
    async def judge(self, post: RawPost, spec: SearchSpec) -> PostVerdict: ...

class ParseResult(BaseModel):
    spec: SearchSpec | None = None
    ambiguous: AmbiguousQuery | None = None
    confidence: float = 0.0

class SpecParser(Protocol):
    async def parse(self, query: str,
                    known_schema: AttributeSchema | None = None) -> ParseResult: ...

class SchemaCache(Protocol):
    def get(self, category: str) -> AttributeSchema | None: ...
    def put(self, schema: AttributeSchema) -> None: ...
    def classify_category(self, query: str) -> str | None: ...
```

> 주: `SpecParser.parse`는 캐시된 스키마를 힌트로 받을 수 있다(Fix #5). `SchemaCache.classify_category`는 질의에서 카테고리만 가볍게 추정한다(미스면 None).

- [ ] **Step 6: fake_classifier.py 구현**

`src/raw_review/adapters/fake_classifier.py`:

```python
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.verdict import PostVerdict

class FakeClassifier:
    def __init__(self, by_url: dict[str, PostVerdict]):
        self._by_url = by_url

    async def judge(self, post: RawPost, spec: SearchSpec) -> PostVerdict:
        return self._by_url[post.url]
```

- [ ] **Step 7: 통과 + 경계 확인**

Run: `cd apps/api && pytest tests/shared/test_verdict.py -v && python -m importlinter.cli lint`
Expected: PASS (shared는 context를 import하지 않으므로 계약 통과)

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/raw_review/shared apps/api/src/raw_review/adapters/fake_classifier.py apps/api/tests/shared/test_verdict.py
git commit -m "feat: add verdict/analyzed/ports in shared kernel"
```

---

### Task 8: analyze_post 유스케이스 (룰 → 광고점수 → LLM)

**Fix #3**: 광고 신호 점수를 실제 판정에 쓴다. 룰 점수 ≥ 임계값이거나 LLM `is_ad`면 `AD_REJECTED`(브랜드 집계용으로 verdict 보존).

**Files:**
- Create: `apps/api/src/raw_review/analysis/application/analyze_post.py` (+ `application/__init__.py`)
- Test: `apps/api/tests/analysis/test_analyze_post.py`

**Interfaces:**
- Consumes: `apply_rule_filter` (Task 5), `score_ad_signals` (Task 6), `AnalyzedPost`/`AnalyzedStatus`/포트 (Task 7)
- Produces:
  - `async analyze_post(post, spec, classifier, *, ad_score_threshold=0.5, max_retries=1) -> AnalyzedPost`
  - 흐름: RuleFilter REJECT → `REJECTED`(LLM 미호출). PASS/UNCERTAIN → 광고점수 계산 → LLM 호출(예외 시 재시도 후 `UNANALYZED`). `relevance.matches=False` → `REJECTED`. `ad.is_ad or score≥threshold` → `AD_REJECTED`. 그 외 `ANALYZED`. 모든 결과에 `source`/`posted_at`/`likes` 복사.

- [ ] **Step 1: 실패 테스트**

`tests/analysis/test_analyze_post.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost, RawComment
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.verdict import (
    PostVerdict, RelevanceVerdict, AdVerdict, Opinion, Polarity)
from raw_review.shared.analyzed import AnalyzedStatus
from raw_review.adapters.fake_classifier import FakeClassifier
from raw_review.analysis.application.analyze_post import analyze_post

def _spec():
    return SearchSpec(category="휴대폰케이스",
        hard_attributes={"대상기기": "아이폰17", "형태": "투명"},
        exclusion_variants=["17 Pro"],
        aliases={"아이폰17": ["iphone17"], "투명": ["클리어"]})

def _post(url, title, body, comments=None):
    return RawPost(source="dc", url=url, title=title, body=body,
                   posted_at=datetime(2026,10,1), likes=3, comments=comments or [])

def _ok_verdict(is_ad=False):
    return PostVerdict(
        relevance=RelevanceVerdict(matches=True, matched_attributes=["대상기기","형태"]),
        ad=AdVerdict(is_ad=is_ad),
        opinions=[Opinion(brand="다이소", polarity=Polarity.CON,
                          aspect="변색", quote="누렇게 뜸", source_part="post")])

async def test_rejected_post_skips_llm():
    post = _post("u1", "아이폰 17 Pro 투명케이스", "프로 클리어 케이스")
    result = await analyze_post(post, _spec(), FakeClassifier({}))
    assert result.status == AnalyzedStatus.REJECTED

async def test_analyzed_carries_post_metadata():
    post = _post("u2", "아이폰17 투명케이스", "클리어 변색")
    result = await analyze_post(post, _spec(), FakeClassifier({"u2": _ok_verdict()}))
    assert result.status == AnalyzedStatus.ANALYZED
    assert result.source == "dc" and result.likes == 3

async def test_ad_by_rule_score_even_if_llm_says_no():
    # 본문/댓글 광고 신호가 강하면 LLM이 is_ad=False여도 AD_REJECTED
    post = _post("u3", "아이폰17 투명케이스", "업체로부터 무상 제공받아 작성",
                 [RawComment(text="뒷광고네")])
    result = await analyze_post(post, _spec(), FakeClassifier({"u3": _ok_verdict(is_ad=False)}),
                                ad_score_threshold=0.5)
    assert result.status == AnalyzedStatus.AD_REJECTED
    assert result.verdict is not None  # 브랜드 집계용 보존

async def test_ad_by_llm_verdict():
    post = _post("u4", "아이폰17 투명케이스", "클리어 변색")
    result = await analyze_post(post, _spec(), FakeClassifier({"u4": _ok_verdict(is_ad=True)}))
    assert result.status == AnalyzedStatus.AD_REJECTED

async def test_llm_failure_marks_unanalyzed():
    post = _post("u5", "아이폰17 투명케이스", "클리어 변색")
    class Boom:
        async def judge(self, post, spec): raise ValueError("bad json")
    result = await analyze_post(post, _spec(), Boom(), max_retries=1)
    assert result.status == AnalyzedStatus.UNANALYZED

async def test_relevance_false_is_rejected():
    post = _post("u6", "아이폰17 투명케이스", "클리어 변색")
    verdict = PostVerdict(relevance=RelevanceVerdict(matches=False, reason="사실 갤럭시"),
                          ad=AdVerdict(is_ad=False), opinions=[])
    result = await analyze_post(post, _spec(), FakeClassifier({"u6": verdict}))
    assert result.status == AnalyzedStatus.REJECTED
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/analysis/test_analyze_post.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/analysis/application/analyze_post.py`:

```python
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.analyzed import AnalyzedPost, AnalyzedStatus
from raw_review.analysis.domain.rule_filter import apply_rule_filter, FilterOutcome
from raw_review.analysis.domain.ad_signals import score_ad_signals

def _base(post: RawPost, status: AnalyzedStatus, **kw) -> AnalyzedPost:
    return AnalyzedPost(url=post.url, source=post.source, posted_at=post.posted_at,
                        likes=post.likes, status=status, **kw)

async def analyze_post(post: RawPost, spec: SearchSpec, classifier,
                       *, ad_score_threshold: float = 0.5,
                       max_retries: int = 1) -> AnalyzedPost:
    rule = apply_rule_filter(post, spec)
    if rule.outcome == FilterOutcome.REJECT:
        return _base(post, AnalyzedStatus.REJECTED, reason=rule.reason)

    ad_score = score_ad_signals(post)

    attempts = max_retries + 1
    verdict = None
    for i in range(attempts):
        try:
            verdict = await classifier.judge(post, spec)
            break
        except Exception:
            if i == attempts - 1:
                return _base(post, AnalyzedStatus.UNANALYZED, reason="LLM 판정 실패")

    if not verdict.relevance.matches:
        return _base(post, AnalyzedStatus.REJECTED, reason=verdict.relevance.reason)

    # Fix #3: 룰 광고 점수와 LLM 판정을 OR
    if verdict.ad.is_ad or ad_score.score >= ad_score_threshold:
        return _base(post, AnalyzedStatus.AD_REJECTED, verdict=verdict,
                     reason=f"광고(점수 {ad_score.score:.2f}, hits={ad_score.hits})")

    return _base(post, AnalyzedStatus.ANALYZED, verdict=verdict)
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/analysis/test_analyze_post.py -v` → PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/analysis/application apps/api/tests/analysis/test_analyze_post.py
git commit -m "feat: add analyze_post combining rule/ad-score/llm"
```

---

### Task 9: AspectNormalizer

**Files:**
- Create: `apps/api/src/raw_review/analysis/domain/aspect_normalizer.py`
- Test: `apps/api/tests/analysis/test_aspect_normalizer.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `AspectNormalizer(synonyms: dict[str, list[str]])`, `.canonical(aspect: str) -> str`, `.default()`

- [ ] **Step 1: 실패 테스트**

`tests/analysis/test_aspect_normalizer.py`:

```python
from raw_review.analysis.domain.aspect_normalizer import AspectNormalizer

def test_synonyms_map_to_canonical():
    n = AspectNormalizer(synonyms={"변색": ["황변", "누래짐", "누렇게"]})
    assert n.canonical("황변") == "변색"
    assert n.canonical("변색") == "변색"

def test_unknown_aspect_passes_through():
    n = AspectNormalizer(synonyms={"변색": ["황변"]})
    assert n.canonical("무게") == "무게"

def test_default_dictionary_available():
    assert AspectNormalizer.default().canonical("황변") == "변색"
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/analysis/test_aspect_normalizer.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/analysis/domain/aspect_normalizer.py`:

```python
from __future__ import annotations
from pydantic import BaseModel

_DEFAULT = {
    "변색": ["황변", "누래짐", "누렇게", "색변함"],
    "그립감": ["손에 쥐는 느낌", "미끄러움", "미끌"],
    "내구성": ["깨짐", "파손", "금감", "크랙"],
    "가격": ["가성비", "비쌈", "저렴"],
    "두께": ["두꺼움", "얇음"],
}

class AspectNormalizer(BaseModel):
    synonyms: dict[str, list[str]]

    def canonical(self, aspect: str) -> str:
        for canonical, syns in self.synonyms.items():
            if aspect == canonical or aspect in syns:
                return canonical
        return aspect

    @classmethod
    def default(cls) -> "AspectNormalizer":
        return cls(synonyms=_DEFAULT)
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/analysis/test_aspect_normalizer.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/analysis/domain/aspect_normalizer.py apps/api/tests/analysis/test_aspect_normalizer.py
git commit -m "feat: add aspect normalizer"
```

---

### Task 10: ReportBuilder (Spec 완전 반영)

**Fix #6**: `generated_at`, `ttl_days`, 소스별 수집 상태, 인용 정렬(언급수→최근성→추천수), 브랜드별 정확한 `analyzed`(그 브랜드 글 수)와 `ad_rejected`(그 브랜드 광고 글 수).

**Files:**
- Create: `apps/api/src/raw_review/review/domain/report.py`
- Create: `apps/api/src/raw_review/review/domain/report_builder.py`
- Test: `apps/api/tests/review/test_report_builder.py`

**Interfaces:**
- Consumes: `AnalyzedPost`/`AnalyzedStatus` (Task 7), `Opinion`/`Polarity` (Task 7), `AspectNormalizer` (Task 9), `SearchSpec` (Task 4)
- Produces:
  - `Quote(text, source, url, posted_at: datetime | None, likes: int)`
  - `ConSummary(aspect, count, quotes: list[Quote])`
  - `ProTag(aspect, count)`
  - `SourceStatus(source: str, status: str, collected: int, reason: str | None)`
  - `BrandSummary(brand, cons, pros, analyzed: int, ad_rejected: int)`
  - `ProductReport(category, query_label, brands, total_analyzed, total_rejected, total_ad_rejected, source_statuses, generated_at, ttl_days)`
  - `build_report(spec, analyzed, normalizer, *, source_statuses=None, ttl_days=7, max_quotes_per_aspect=5, now=None) -> ProductReport`

- [ ] **Step 1: 실패 테스트**

`tests/review/test_report_builder.py`:

```python
from datetime import datetime, timezone
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.verdict import (
    PostVerdict, RelevanceVerdict, AdVerdict, Opinion, Polarity)
from raw_review.shared.analyzed import AnalyzedPost, AnalyzedStatus
from raw_review.analysis.domain.aspect_normalizer import AspectNormalizer
from raw_review.review.domain.report_builder import build_report
from raw_review.review.domain.report import SourceStatus

def _spec():
    return SearchSpec(category="휴대폰케이스",
        hard_attributes={"대상기기": "아이폰17", "형태": "투명"})

def _analyzed(url, opinions, status=AnalyzedStatus.ANALYZED,
              posted_at=None, likes=0, source="dc"):
    return AnalyzedPost(url=url, source=source, posted_at=posted_at, likes=likes,
        status=status,
        verdict=PostVerdict(relevance=RelevanceVerdict(matches=True),
                            ad=AdVerdict(is_ad=status==AnalyzedStatus.AD_REJECTED),
                            opinions=opinions))

def _con(brand, aspect, quote):
    return Opinion(brand=brand, polarity=Polarity.CON, aspect=aspect,
                   quote=quote, source_part="post")

def test_empty_input_yields_empty_report():
    r = build_report(_spec(), [], AspectNormalizer.default())
    assert r.brands == [] and r.total_analyzed == 0
    assert r.generated_at is not None and r.ttl_days == 7

def test_synonym_cons_merge_into_one_aspect():
    posts = [_analyzed("u1", [_con("다이소", "황변", "누렇게 뜸")]),
             _analyzed("u2", [_con("다이소", "변색", "색 변함")])]
    r = build_report(_spec(), posts, AspectNormalizer.default())
    daiso = next(b for b in r.brands if b.brand == "다이소")
    disc = next(c for c in daiso.cons if c.aspect == "변색")
    assert disc.count == 2 and len(disc.quotes) == 2

def test_brand_analyzed_counts_only_that_brands_posts():
    posts = [_analyzed("u1", [_con("다이소", "변색", "q")]),
             _analyzed("u2", [_con("신지모루", "두께", "q")])]
    r = build_report(_spec(), posts, AspectNormalizer.default())
    daiso = next(b for b in r.brands if b.brand == "다이소")
    assert daiso.analyzed == 1  # 전체 2가 아니라 다이소 글 1

def test_ad_rejected_counted_per_brand():
    posts = [_analyzed("u1", [_con("다이소", "변색", "q")]),
             _analyzed("u2", [_con("다이소", "변색", "q")],
                       status=AnalyzedStatus.AD_REJECTED)]
    r = build_report(_spec(), posts, AspectNormalizer.default())
    daiso = next(b for b in r.brands if b.brand == "다이소")
    assert daiso.analyzed == 1 and daiso.ad_rejected == 1
    assert r.total_ad_rejected == 1

def test_quotes_sorted_by_recency_then_likes_and_capped():
    old = datetime(2026,1,1,tzinfo=timezone.utc)
    new = datetime(2026,10,1,tzinfo=timezone.utc)
    posts = [
        _analyzed("u1", [_con("A","변색","old-lowlike")], posted_at=old, likes=1),
        _analyzed("u2", [_con("A","변색","new-highlike")], posted_at=new, likes=9),
        _analyzed("u3", [_con("A","변색","new-lowlike")], posted_at=new, likes=0),
    ]
    r = build_report(_spec(), posts, AspectNormalizer.default(), max_quotes_per_aspect=2)
    con = r.brands[0].cons[0]
    assert con.count == 3 and len(con.quotes) == 2
    assert con.quotes[0].text == "new-highlike"  # 최근 + 추천 많은 것 먼저

def test_source_statuses_passed_through():
    statuses = [SourceStatus(source="dcinside", status="succeeded", collected=10, reason=None),
                SourceStatus(source="fmkorea", status="failed", collected=0, reason="BLOCKED")]
    r = build_report(_spec(), [], AspectNormalizer.default(), source_statuses=statuses)
    assert any(s.status == "failed" for s in r.source_statuses)

def test_pros_counted_as_tags_with_normalization():
    ops = [Opinion(brand="A", polarity=Polarity.PRO, aspect="가격", quote="쌈", source_part="post"),
           Opinion(brand="A", polarity=Polarity.PRO, aspect="가성비", quote="가성비 좋음", source_part="post")]
    r = build_report(_spec(), [_analyzed("u1", ops)], AspectNormalizer.default())
    price = next(p for p in r.brands[0].pros if p.aspect == "가격")
    assert price.count == 2  # 가성비 → 가격 정규화
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/review/test_report_builder.py -v` → FAIL

- [ ] **Step 3: report.py 구현**

`src/raw_review/review/domain/report.py`:

```python
from datetime import datetime
from pydantic import BaseModel

class Quote(BaseModel):
    text: str
    source: str
    url: str
    posted_at: datetime | None = None
    likes: int = 0

class ConSummary(BaseModel):
    aspect: str
    count: int
    quotes: list[Quote]

class ProTag(BaseModel):
    aspect: str
    count: int

class SourceStatus(BaseModel):
    source: str
    status: str            # succeeded | partial | failed
    collected: int = 0
    reason: str | None = None

class BrandSummary(BaseModel):
    brand: str
    cons: list[ConSummary]
    pros: list[ProTag]
    analyzed: int
    ad_rejected: int

class ProductReport(BaseModel):
    category: str
    query_label: str
    brands: list[BrandSummary]
    total_analyzed: int
    total_rejected: int
    total_ad_rejected: int
    source_statuses: list[SourceStatus]
    generated_at: datetime
    ttl_days: int
```

- [ ] **Step 4: report_builder.py 구현**

`src/raw_review/review/domain/report_builder.py`:

```python
from collections import defaultdict
from datetime import datetime, timezone
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.verdict import Polarity
from raw_review.shared.analyzed import AnalyzedStatus
from raw_review.review.domain.report import (
    ProductReport, BrandSummary, ConSummary, ProTag, Quote, SourceStatus)

_EPOCH = datetime.min.replace(tzinfo=timezone.utc)

def build_report(spec, analyzed, normalizer, *, source_statuses=None,
                 ttl_days: int = 7, max_quotes_per_aspect: int = 5,
                 now: datetime | None = None) -> ProductReport:
    now = now or datetime.now(timezone.utc)
    kept = [a for a in analyzed if a.status == AnalyzedStatus.ANALYZED and a.verdict]
    ad_posts = [a for a in analyzed if a.status == AnalyzedStatus.AD_REJECTED and a.verdict]
    rejected = [a for a in analyzed if a.status == AnalyzedStatus.REJECTED]

    cons: dict[str, dict[str, list[Quote]]] = defaultdict(lambda: defaultdict(list))
    pros: dict[str, dict[str, int]] = defaultdict(lambda: defaultdict(int))
    brand_posts: dict[str, set[str]] = defaultdict(set)
    ad_rejected: dict[str, int] = defaultdict(int)

    for post in kept:
        for op in post.verdict.opinions:
            brand_posts[op.brand].add(post.url)
            aspect = normalizer.canonical(op.aspect)
            if op.polarity == Polarity.CON:
                cons[op.brand][aspect].append(Quote(
                    text=op.quote, source=post.source, url=post.url,
                    posted_at=post.posted_at, likes=post.likes))
            else:
                pros[op.brand][aspect] += 1

    for post in ad_posts:
        for brand in {op.brand for op in post.verdict.opinions}:
            ad_rejected[brand] += 1

    brands: list[BrandSummary] = []
    for brand in sorted(set(cons) | set(pros) | set(ad_rejected)):
        con_list = []
        for aspect, quotes in cons.get(brand, {}).items():
            quotes.sort(key=lambda q: (q.posted_at or _EPOCH, q.likes), reverse=True)
            con_list.append(ConSummary(aspect=aspect, count=len(quotes),
                                       quotes=quotes[:max_quotes_per_aspect]))
        con_list.sort(key=lambda c: c.count, reverse=True)
        pro_list = sorted(
            (ProTag(aspect=a, count=c) for a, c in pros.get(brand, {}).items()),
            key=lambda p: p.count, reverse=True)
        brands.append(BrandSummary(
            brand=brand, cons=con_list, pros=pro_list,
            analyzed=len(brand_posts.get(brand, set())),
            ad_rejected=ad_rejected.get(brand, 0)))

    return ProductReport(
        category=spec.category,
        query_label=" ".join(spec.hard_attributes.values()),
        brands=brands,
        total_analyzed=len(kept),
        total_rejected=len(rejected),
        total_ad_rejected=len(ad_posts),
        source_statuses=source_statuses or [],
        generated_at=now,
        ttl_days=ttl_days)
```

- [ ] **Step 5: 통과 확인** — Run: `cd apps/api && pytest tests/review/test_report_builder.py -v` → PASS (7 tests)

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/raw_review/review apps/api/tests/review
git commit -m "feat: add report builder with full spec fields"
```

---

### Task 11: 설정 모듈

**Files:**
- Create: `apps/api/src/raw_review/config.py`
- Test: `apps/api/tests/test_config.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `Settings` (BaseSettings, env_prefix `RR_`): `openai_api_key`, `openai_model="gpt-4o-mini"`, `relevance_precision_floor=0.95`, `ad_recall_floor=0.9`, `ad_score_threshold=0.5`, `max_posts_per_job=200`, `quote_max=120`, `report_ttl_days=7`
  - `get_settings() -> Settings`

- [ ] **Step 1: 실패 테스트**

`tests/test_config.py`:

```python
from raw_review.config import Settings

def test_settings_read_from_env(monkeypatch):
    monkeypatch.setenv("RR_OPENAI_API_KEY", "sk-test")
    monkeypatch.setenv("RR_OPENAI_MODEL", "gpt-4o")
    s = Settings()
    assert s.openai_api_key == "sk-test"
    assert s.openai_model == "gpt-4o"
    assert s.relevance_precision_floor == 0.95
    assert s.ad_score_threshold == 0.5
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
    openai_model: str = "gpt-4o-mini"
    relevance_precision_floor: float = 0.95
    ad_recall_floor: float = 0.9
    ad_score_threshold: float = 0.5
    max_posts_per_job: int = 200
    quote_max: int = 120
    report_ttl_days: int = 7

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/test_config.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/config.py apps/api/tests/test_config.py
git commit -m "feat: add settings module"
```

---

### Task 12: OpenAiClassifier (JSON Schema 구조화 출력)

**Fix #4**: `response_format.type = "json_schema"` + `PostVerdict.model_json_schema()`, `strict: true`.

**Files:**
- Create: `apps/api/src/raw_review/adapters/openai_classifier.py`
- Test: `apps/api/tests/adapters/test_openai_classifier.py`

**Interfaces:**
- Consumes: `RawPost`/`SearchSpec`/`PostVerdict` (shared)
- Produces:
  - `build_messages(post, spec) -> list[dict]` (순수)
  - `response_schema() -> dict` — PostVerdict의 JSON schema에 `additionalProperties:false`와 `strict` 적용 (순수)
  - `parse_response(raw_json: str) -> PostVerdict` (순수, 실패 시 `ValueError`)
  - `OpenAiClassifier(client, model)` — `judge`에서 `response_format={"type":"json_schema","json_schema":{"name":"post_verdict","schema":response_schema(),"strict":True}}` 사용
- 주: 네트워크 호출은 통합/골든셋에서만. 단위테스트는 순수 함수 3개만 검증.

- [ ] **Step 1: 실패 테스트**

`tests/adapters/test_openai_classifier.py`:

```python
import pytest
from datetime import datetime
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.adapters.openai_classifier import (
    build_messages, parse_response, response_schema)

def _spec():
    return SearchSpec(category="휴대폰케이스",
        hard_attributes={"대상기기": "아이폰17", "형태": "투명"})

def _post():
    return RawPost(source="dc", url="u", title="아이폰17 투명케이스",
                   body="변색됨", posted_at=datetime(2026,10,1), likes=0, comments=[])

def test_build_messages_includes_query_and_body():
    joined = " ".join(m["content"] for m in build_messages(_post(), _spec()))
    assert "아이폰17" in joined and "변색됨" in joined

def test_response_schema_is_strict_object():
    schema = response_schema()
    assert schema["type"] == "object"
    assert schema["additionalProperties"] is False
    assert "relevance" in schema["properties"]

def test_parse_valid_response():
    raw = '''{"relevance": {"matches": true, "matched_attributes": ["대상기기"], "reason": ""},
              "ad": {"is_ad": false, "confidence": 0.1, "evidence": []},
              "opinions": [{"brand": "다이소", "product_line": null, "polarity": "con",
                            "aspect": "변색", "quote": "누렇게 뜸", "source_part": "post"}]}'''
    v = parse_response(raw)
    assert v.relevance.matches is True and v.opinions[0].aspect == "변색"

def test_parse_invalid_json_raises():
    with pytest.raises(ValueError):
        parse_response("{not json")
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/adapters/test_openai_classifier.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/adapters/openai_classifier.py`:

```python
import json
from pydantic import ValidationError
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.verdict import PostVerdict

_SYSTEM = (
    "너는 제품 후기 분석기다. 주어진 글이 요청한 모델/형태에 해당하는지 판단하고, "
    "광고/협찬 여부를 판단하고, 브랜드별 장단점을 추출한다. "
    "단점 인용은 원문 표현 그대로 쓴다. 반드시 지정한 JSON 스키마로만 답한다."
)

def _strict(schema: dict) -> dict:
    # OpenAI strict json_schema: 모든 object에 additionalProperties:false,
    # 모든 property를 required로.
    if schema.get("type") == "object":
        schema["additionalProperties"] = False
        props = schema.get("properties", {})
        if props:
            schema["required"] = list(props.keys())
        for sub in props.values():
            _strict(sub)
    for key in ("items", "$defs", "anyOf"):
        node = schema.get(key)
        if isinstance(node, dict):
            if key == "$defs":
                for d in node.values():
                    _strict(d)
            else:
                _strict(node)
        elif isinstance(node, list):
            for d in node:
                _strict(d)
    return schema

def response_schema() -> dict:
    return _strict(PostVerdict.model_json_schema())

def build_messages(post: RawPost, spec: SearchSpec) -> list[dict]:
    comments = "\n".join(f"- {c.text}" for c in post.comments)
    user = (
        f"[검색 조건]\n카테고리: {spec.category}\n"
        f"필수속성: {json.dumps(spec.hard_attributes, ensure_ascii=False)}\n"
        f"제외: {spec.exclusion_variants}\n\n"
        f"[글]\n제목: {post.title}\n본문: {post.body}\n별점: {post.rating}\n"
        f"댓글:\n{comments}"
    )
    return [{"role": "system", "content": _SYSTEM},
            {"role": "user", "content": user}]

def parse_response(raw_json: str) -> PostVerdict:
    try:
        return PostVerdict.model_validate(json.loads(raw_json))
    except (json.JSONDecodeError, ValidationError) as e:
        raise ValueError(f"응답 파싱 실패: {e}") from e

class OpenAiClassifier:
    def __init__(self, client, model: str):
        self._client = client
        self._model = model

    async def judge(self, post: RawPost, spec: SearchSpec) -> PostVerdict:
        resp = await self._client.chat.completions.create(
            model=self._model,
            messages=build_messages(post, spec),
            response_format={"type": "json_schema", "json_schema": {
                "name": "post_verdict", "schema": response_schema(), "strict": True}},
            temperature=0,
        )
        return parse_response(resp.choices[0].message.content)
```

> 주: `PostVerdict`에 `Enum`(Polarity)이 있으면 schema에 `$defs`로 들어간다. `_strict`가 `$defs`도 재귀 처리한다. strict 모드가 특정 모델에서 거부되면 Task 실행 시 `json_object` + 수동 검증으로 폴백하고 그 사유를 커밋 메시지에 남긴다.

- [ ] **Step 4: 통과 확인** — Run: `cd apps/api && pytest tests/adapters/test_openai_classifier.py -v` → PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/adapters/openai_classifier.py apps/api/tests/adapters
git commit -m "feat: add OpenAI classifier with json_schema output"
```

---

### Task 13: CategoryResolver (캐시 읽기 포함)

**Fix #5**: 캐시를 읽는다. 카테고리 추정 → `cache.get` 히트면 스키마를 파서에 힌트로 넘겨 일관된 SearchSpec 생성, 미스면 전체 파싱 후 스키마 캐시.

**Files:**
- Create: `apps/api/src/raw_review/catalog/application/resolve_category.py` (+ `application/__init__.py`)
- Create: `apps/api/src/raw_review/catalog/application/schema_cache.py`
- Test: `apps/api/tests/catalog/test_resolve_category.py`

**Interfaces:**
- Consumes: `SearchSpec`/`AttributeSchema`/`Attribute`/`AmbiguousQuery` (Task 4), `ParseResult`/`SpecParser`/`SchemaCache` 포트 (Task 7)
- Produces:
  - `InMemorySchemaCache` 구현 (`get`/`put`/`classify_category`)
  - `async resolve_category(query, parser, cache, *, confidence_floor=0.6) -> ParseResult`
  - 동작: `cache.classify_category(query)` → 히트면 `cache.get`한 스키마를 `parser.parse(query, known_schema=...)`에 넘김. confidence < floor 또는 spec None이면 ambiguous 반환. 성공 시 스키마를 캐시에 저장(put).

- [ ] **Step 1: 실패 테스트**

`tests/catalog/test_resolve_category.py`:

```python
from raw_review.shared.search_spec import (
    SearchSpec, AttributeSchema, Attribute, AmbiguousQuery)
from raw_review.shared.ports import ParseResult
from raw_review.catalog.application.resolve_category import resolve_category
from raw_review.catalog.application.schema_cache import InMemorySchemaCache

class RecordingParser:
    def __init__(self, result):
        self._result = result
        self.seen_schema = "unset"
    async def parse(self, query, known_schema=None):
        self.seen_schema = known_schema
        return self._result

def _spec():
    return SearchSpec(category="휴대폰케이스",
        hard_attributes={"대상기기": "아이폰17", "형태": "투명"})

async def test_high_confidence_returns_spec_and_caches_schema():
    cache = InMemorySchemaCache()
    parser = RecordingParser(ParseResult(spec=_spec(), confidence=0.9))
    result = await resolve_category("아이폰17 투명케이스", parser, cache)
    assert result.spec.category == "휴대폰케이스"
    assert cache.get("휴대폰케이스") is not None

async def test_cache_hit_passes_known_schema_to_parser():
    cache = InMemorySchemaCache()
    schema = AttributeSchema(category="휴대폰케이스",
        attributes=[Attribute(name="대상기기", value="", hard=True)])
    cache.put(schema)
    # classify_category가 캐시된 카테고리를 찾도록 질의에 카테고리 힌트 포함
    parser = RecordingParser(ParseResult(spec=_spec(), confidence=0.9))
    await resolve_category("휴대폰케이스 아이폰17 투명", parser, cache)
    assert parser.seen_schema is not None
    assert parser.seen_schema.category == "휴대폰케이스"

async def test_low_confidence_returns_ambiguous():
    amb = AmbiguousQuery(query="키보드", options=["기계식", "멤브레인", "태블릿용"])
    parser = RecordingParser(ParseResult(spec=None, ambiguous=amb, confidence=0.3))
    result = await resolve_category("키보드", parser, InMemorySchemaCache())
    assert result.ambiguous is not None and "기계식" in result.ambiguous.options
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
        # 가벼운 추정: 캐시된 카테고리명이 질의에 들어있으면 그것.
        q = normalize(query)
        for category in self._by_category:
            if normalize(category) in q:
                return category
        return None
```

- [ ] **Step 4: resolve_category.py 구현**

`src/raw_review/catalog/application/resolve_category.py`:

```python
from raw_review.shared.search_spec import AttributeSchema, Attribute, AmbiguousQuery
from raw_review.shared.ports import ParseResult

async def resolve_category(query, parser, cache, *, confidence_floor: float = 0.6) -> ParseResult:
    # Fix #5: 캐시 읽기 — 카테고리를 추정해 스키마가 있으면 파서에 힌트로 전달
    known_schema = None
    category = cache.classify_category(query)
    if category is not None:
        known_schema = cache.get(category)

    result = await parser.parse(query, known_schema=known_schema)

    if result.spec is None or result.confidence < confidence_floor:
        if result.ambiguous is not None:
            return result
        return ParseResult(
            ambiguous=AmbiguousQuery(query=query, options=[]),
            confidence=result.confidence)

    schema = AttributeSchema(
        category=result.spec.category,
        attributes=[Attribute(name=n, value=v, hard=True)
                    for n, v in result.spec.hard_attributes.items()]
        + [Attribute(name=n, value=v, hard=False)
           for n, v in result.spec.soft_attributes.items()])
    cache.put(schema)
    return result
```

- [ ] **Step 5: 통과 확인** — Run: `cd apps/api && pytest tests/catalog/test_resolve_category.py -v` → PASS (3 tests)

- [ ] **Step 6: 전체 테스트 + 경계 계약**

Run: `cd apps/api && pytest -v && python -m importlinter.cli lint`
Expected: 전부 PASS

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/raw_review/catalog apps/api/tests/catalog
git commit -m "feat: add category resolver with schema cache read"
```

---

### Task 14: 룰 계층 평가 하네스 (실제형 라벨, oracle 없음)

**Fix #7**: 1단계는 룰 계층(RuleFilter·광고 신호)을 손 라벨링 케이스로 **실제 로직 그대로** 검증한다. LLM precision/recall 100건 게이트는 2단계(수집)에 종속.

**Files:**
- Create: `apps/api/src/raw_review/eval/metrics.py`
- Create: `apps/api/tests/eval/rule_cases.jsonl`
- Create: `apps/api/tests/eval/conftest.py`
- Create: `apps/api/tests/eval/test_rule_eval.py`

**Interfaces:**
- Consumes: `apply_rule_filter` (Task 5), `score_ad_signals` (Task 6), `RawPost`/`SearchSpec` (shared)
- Produces:
  - `precision(tp, fp) -> float`, `recall(tp, fn) -> float`

- [ ] **Step 1: metrics 실패 테스트**

`tests/eval/test_rule_eval.py` (metrics 부분):

```python
from raw_review.eval.metrics import precision, recall

def test_precision_recall_basic():
    assert precision(tp=9, fp=1) == 0.9
    assert recall(tp=8, fn=2) == 0.8
    assert precision(tp=0, fp=0) == 1.0  # 분모 0이면 1.0
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/eval/test_rule_eval.py -v` → FAIL

- [ ] **Step 3: metrics 구현**

`src/raw_review/eval/metrics.py`:

```python
def precision(tp: int, fp: int) -> float:
    return 1.0 if (tp + fp) == 0 else tp / (tp + fp)

def recall(tp: int, fn: int) -> float:
    return 1.0 if (tp + fn) == 0 else tp / (tp + fn)
```

- [ ] **Step 4: 라벨 케이스 작성**

`tests/eval/rule_cases.jsonl` — 손으로 작성한 실제형 케이스 20건 이상. 각 줄: `spec`, `post`, `rule_relevant`(제외어/키워드 룰로 통과해야 하나), `rule_ad`(광고 신호로 걸러야 하나). 다양한 함정 포함: 17 Pro/Air 제외, 가죽케이스 제외, 무상제공/체험단, 댓글 뒷광고, 모델만 있고 형태 없음(UNCERTAIN은 relevant로 간주). 예시 3줄:

```json
{"spec": {"category": "휴대폰케이스", "hard_attributes": {"대상기기": "아이폰17", "형태": "투명"}, "exclusion_variants": ["17 Pro", "17 Air", "가죽"], "aliases": {"아이폰17": ["iphone17"], "투명": ["클리어"]}}, "post": {"source": "dcinside", "url": "c1", "title": "아이폰17 투명케이스 한달", "body": "다이소 클리어 2주만에 누렇게 변색", "likes": 12, "comments": []}, "rule_relevant": true, "rule_ad": false}
{"spec": {"category": "휴대폰케이스", "hard_attributes": {"대상기기": "아이폰17", "형태": "투명"}, "exclusion_variants": ["17 Pro", "17 Air", "가죽"], "aliases": {"아이폰17": ["iphone17"], "투명": ["클리어"]}}, "post": {"source": "clien", "url": "c2", "title": "아이폰17 Pro 투명케이스 후기", "body": "프로 모델 클리어 케이스", "likes": 3, "comments": []}, "rule_relevant": false, "rule_ad": false}
{"spec": {"category": "휴대폰케이스", "hard_attributes": {"대상기기": "아이폰17", "형태": "투명"}, "exclusion_variants": ["17 Pro", "17 Air", "가죽"], "aliases": {"아이폰17": ["iphone17"], "투명": ["클리어"]}}, "post": {"source": "blog", "url": "c3", "title": "아이폰17 투명케이스 강추", "body": "업체로부터 무상 제공받아 작성한 후기 정말 최고", "likes": 1, "comments": [{"text": "뒷광고네"}]}, "rule_relevant": true, "rule_ad": true}
```

> 주: 20건을 전부 손으로 채운다. 2단계 수집이 들어오면 실제 긁은 글로 교체·확장하고, 여기에 LLM precision≥0.95 / recall≥0.9 게이트(100건)를 추가한다.

- [ ] **Step 5: 로더 + 룰 평가 테스트**

`tests/eval/conftest.py`:

```python
import json
from pathlib import Path
import pytest
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec

@pytest.fixture
def rule_cases():
    path = Path(__file__).parent / "rule_cases.jsonl"
    rows = [json.loads(l) for l in path.read_text(encoding="utf-8").splitlines() if l.strip()]
    return [{"post": RawPost.model_validate(r["post"]),
             "spec": SearchSpec.model_validate(r["spec"]),
             "rule_relevant": r["rule_relevant"], "rule_ad": r["rule_ad"]} for r in rows]
```

`tests/eval/test_rule_eval.py`에 추가:

```python
from raw_review.analysis.domain.rule_filter import apply_rule_filter, FilterOutcome
from raw_review.analysis.domain.ad_signals import score_ad_signals
from raw_review.eval.metrics import precision, recall
from raw_review.config import get_settings

def test_dataset_has_enough_cases(rule_cases):
    assert len(rule_cases) >= 20

def test_rule_filter_precision_on_labeled_cases(rule_cases):
    # 룰이 통과(PASS/UNCERTAIN)시킨 것 중 실제 관련 글 비율
    tp = fp = 0
    for c in rule_cases:
        passed = apply_rule_filter(c["post"], c["spec"]).outcome != FilterOutcome.REJECT
        if passed and c["rule_relevant"]:
            tp += 1
        elif passed and not c["rule_relevant"]:
            fp += 1
    assert precision(tp, fp) >= get_settings().relevance_precision_floor

def test_ad_signal_recall_on_labeled_cases(rule_cases):
    s = get_settings()
    tp = fn = 0
    for c in rule_cases:
        if c["rule_ad"]:
            flagged = score_ad_signals(c["post"]).score >= s.ad_score_threshold
            tp += 1 if flagged else 0
            fn += 0 if flagged else 1
    assert recall(tp, fn) >= s.ad_recall_floor
```

> 주: 이 테스트는 oracle이 아니라 **실제 룰 구현을 라벨에 대고 돌린다.** 임계값을 못 넘기면 룰 사전(제외어/신호어/별칭)을 보강해 통과시킨다 — 그게 1단계 룰 계층 튜닝이다.

- [ ] **Step 6: 통과 확인** — Run: `cd apps/api && pytest tests/eval -v` → PASS (라벨 데이터로 룰 통과)

- [ ] **Step 7: 전체 + 경계 최종 확인**

Run: `cd apps/api && pytest -v && python -m importlinter.cli lint`
Expected: 전부 PASS

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/raw_review/eval apps/api/tests/eval
git commit -m "feat: add rule-layer eval harness with labeled cases"
```

---

## Self-Review 결과

**1. Spec coverage**
- catalog(SearchSpec, CategoryResolver+캐시 읽기, 별칭): Task 4, 13 ✓
- analysis(RuleFilter, 광고 신호, analyze_post에서 광고 OR 결합, AspectNormalizer): Task 5, 6, 8, 9 ✓
- review(ReportBuilder: generated_at/ttl/source_statuses/정렬/브랜드별 analyzed·ad_rejected): Task 10 ✓
- LLM 포트 + Fake + OpenAI(json_schema): Task 7, 12 ✓
- 공유 커널 경계 + import-linter: Task 1, 재확인 13/14 ✓
- 설정값: Task 11 ✓
- 룰 계층 실제 평가, LLM 게이트는 2단계 종속 명시: Task 14 ✓

**2. 검토 지적 반영**
1. 경계 충돌 → shared 커널 + normalize 이동, independence/forbidden 계약 ✓
2. 17 Pro 부분 문자열 → 제외어 우선 REJECT, 전용 테스트 ✓
3. 광고 점수 미사용 → analyze_post에서 OR 결합, AD_REJECTED, 전용 테스트 ✓
4. json_object → json_schema + strict + response_schema 테스트 ✓
5. SchemaCache 읽기 → classify_category + known_schema 전달, 캐시 히트 테스트 ✓
6. 리포트 축소 → 필드 전부 추가, 브랜드별 analyzed/ad_rejected 정정, 정렬 테스트 ✓
7. oracle 골든셋 → 룰 계층 실제 라벨 평가(20건+), LLM 100건 게이트 2단계 종속 명시 ✓

**3. Placeholder scan**: 모든 코드 스텝에 실제 코드. "적절히 처리" 류 없음. rule_cases.jsonl은 포맷 예시 3줄 + 20건 작성 지침 ✓

**4. Type consistency**: `AnalyzedPost`(source/posted_at/likes/status), `AnalyzedStatus`, `PostVerdict`, `ParseResult`, `AttributeSchema`가 정의(shared)와 소비(analysis/review/catalog) 전반에서 동일 ✓
