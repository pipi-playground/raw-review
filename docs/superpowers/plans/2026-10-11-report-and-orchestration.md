# 리포트 + 오케스트레이션 Implementation Plan (1b)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 매칭 엔진(1a)이 낸 글별 `PostAnalysis`를 브랜드별 `ProductReport`로 집계하고, 질의 해석→여러 글 분석→리포트까지 조립하는 `run_search` 오케스트레이션을 만든다. 광고 판별(룰)도 여기서 적용한다.

**Architecture:** `review` context가 리포트 도메인(집계·광고 룰)을 소유한다. 질의 해석(catalog)·글 분석(matching)·집계(review)를 잇는 `run_search`는 어느 context에도 속하지 않는 **합성 루트** `app/`에 둔다. app만 세 context를 import할 수 있고, context끼리는 여전히 서로 import하지 않는다.

**Tech Stack:** Python 3.12, Pydantic v2, pytest, import-linter. 1a 매칭 엔진 산출물에 의존.

**Spec:** `docs/superpowers/specs/2026-10-11-matching-engine-design.md`(§4.6, §4.7), 상위 `2026-10-11-raw-review-design.md`(§4.4 review, §10 로드맵 1b). 1a 플랜: `2026-10-11-matching-engine.md`.

## Global Constraints

- 1a 플랜 완료가 선행(공유 커널, `PostAnalysis`, `analyze_post`, `resolve_category`, `Opinion`, `AspectNormalizer`, `Settings`).
- `review`는 다른 context를 import하지 않는다. 오케스트레이션은 `app/`(합성 루트)만 담당.
- 리포트 인용(`Quote`)은 글 메타(`PostAnalysis.source/url/posted_at/likes`)에서 채운다. `Opinion`에 url을 추가하지 않는다.
- 광고 판별은 룰 신호 점수(설정 임계값). 광고 글은 리포트 집계에서 제외하되 브랜드별 `ad_rejected`로 집계.
- 상태 분리 유지: OK(집계), EMPTY("후기 없음"), UNANALYZED("분석 실패")를 리포트 통계에 따로 노출.
- 리포트 TTL 기본 7일(설정값).

## Review Focus

- **전부 광고/빈 입력**: 모든 글이 광고거나 0건이면 brands=[]로 터지지 않아야 한다. (Task 4)
- **브랜드 None**: 브랜드 미상 의견이 "브랜드 미상" 버킷으로 모여야 한다. (Task 4)
- **인용 정렬**: 같은 aspect 인용이 최근→추천수 순이어야 한다. (Task 4)
- **애매 질의**: run_search가 AmbiguousQuery면 리포트 대신 선택지를 돌려줘야 한다. (Task 5)
- **부분 실패**: 일부 글 UNANALYZED여도 나머지로 리포트가 나와야 하고 실패 수가 보여야 한다. (Task 5)

---

## File Structure

```
apps/api/src/raw_review/
  review/
    __init__.py
    domain/
      __init__.py
      report.py            # Quote, ConSummary, ProTag, SourceStatus, BrandSummary, ProductReport
      ad_signals.py        # score_ad_signals (룰)
      report_builder.py    # build_report
  app/
    __init__.py
    run_search.py          # run_search 합성 루트 + SearchResult
tests/
  review/__init__.py  test_ad_signals.py test_report_builder.py
  app/__init__.py  test_run_search.py
```

---

### Task 1: 경계 계약 확장 (review 독립 + app 합성 루트)

**Files:**
- Modify: `apps/api/setup.cfg`
- Create: `apps/api/src/raw_review/review/__init__.py`, `review/domain/__init__.py`, `app/__init__.py`
- Create: `apps/api/tests/review/__init__.py`, `tests/app/__init__.py`

**Interfaces:**
- Consumes: 1a의 import-linter 계약
- Produces: review를 independence에 추가. app은 제약 집합에 넣지 않음(세 context import 허용).

- [ ] **Step 1: setup.cfg 수정** — `context-independence`에 `raw_review.review` 추가, `domain-pure`에 `raw_review.review.domain` 추가.

```ini
[importlinter:contract:context-independence]
name = catalog/matching/review는 서로 import하지 않는다
type = independence
modules =
    raw_review.catalog
    raw_review.matching
    raw_review.review

[importlinter:contract:domain-pure]
name = domain은 adapters/openai를 모른다
type = forbidden
source_modules =
    raw_review.matching.domain
    raw_review.review.domain
forbidden_modules =
    raw_review.adapters
    openai
```

> 주: `app`은 어떤 계약에도 넣지 않는다 → catalog/matching/review를 자유롭게 import하는 합성 루트. `shared-pure` 계약은 그대로.

- [ ] **Step 2: 빈 패키지 생성** (위 __init__.py들)

- [ ] **Step 3: 경계 통과 확인** — Run: `cd apps/api && python -m importlinter.cli lint` → PASS (빈 패키지)

- [ ] **Step 4: Commit**

```bash
git add apps/api/setup.cfg apps/api/src/raw_review/review apps/api/src/raw_review/app apps/api/tests/review apps/api/tests/app
git commit -m "chore: add review context and app composition root to boundaries"
```

---

### Task 2: 리포트 모델

**Files:**
- Create: `apps/api/src/raw_review/review/domain/report.py`
- Test: `apps/api/tests/review/test_report_model.py`

**Interfaces:**
- Consumes: 없음
- Produces:
  - `Quote(text, source, url, posted_at, likes)`
  - `ConSummary(aspect, count, quotes: list[Quote])`
  - `ProTag(aspect, count)`
  - `SourceStatus(source, status, collected, reason)`
  - `BrandSummary(brand: str | None, cons, pros, analyzed, ad_rejected)`
  - `ProductReport(category, query_label, brands, total_analyzed, total_ad_rejected, total_empty, total_unanalyzed, source_statuses, generated_at, ttl_days)`

- [ ] **Step 1: 실패 테스트**

`tests/review/test_report_model.py`:

```python
from datetime import datetime
from raw_review.review.domain.report import ProductReport, BrandSummary

def test_report_minimal():
    r = ProductReport(category="양말", query_label="울 양말", brands=[],
                      total_analyzed=0, total_ad_rejected=0, total_empty=0,
                      total_unanalyzed=0, source_statuses=[],
                      generated_at=datetime(2026,10,11), ttl_days=7)
    assert r.brands == [] and r.ttl_days == 7
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/review/test_report_model.py -v` → FAIL

- [ ] **Step 3: 구현**

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
    brand: str | None      # None = 브랜드 미상
    cons: list[ConSummary]
    pros: list[ProTag]
    analyzed: int
    ad_rejected: int

class ProductReport(BaseModel):
    category: str
    query_label: str
    brands: list[BrandSummary]
    total_analyzed: int
    total_ad_rejected: int
    total_empty: int
    total_unanalyzed: int
    source_statuses: list[SourceStatus]
    generated_at: datetime
    ttl_days: int
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/review/test_report_model.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/review/domain/report.py apps/api/tests/review/test_report_model.py
git commit -m "feat: add report models"
```

---

### Task 3: 광고 신호 룰

**Files:**
- Create: `apps/api/src/raw_review/review/domain/ad_signals.py`
- Test: `apps/api/tests/review/test_ad_signals.py`

**Interfaces:**
- Consumes: `RawPost` (shared)
- Produces:
  - `AdSignalScore(score: float, hits: list[str])`
  - `score_ad_signals(post: RawPost) -> AdSignalScore`
  - `is_ad(post, *, threshold) -> bool`

- [ ] **Step 1: 실패 테스트**

`tests/review/test_ad_signals.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost, RawComment
from raw_review.review.domain.ad_signals import score_ad_signals, is_ad

def _post(body, comments=None):
    return RawPost(source="dc", url="u", title="t", body=body,
                   posted_at=datetime(2026,10,1), likes=0, comments=comments or [])

def test_sponsorship_raises_score():
    clean = score_ad_signals(_post("그냥 변색 후기"))
    ad = score_ad_signals(_post("업체로부터 무상 제공받아 작성한 후기"))
    assert ad.score > clean.score and any("제공" in h for h in ad.hits)

def test_comment_callout():
    r = score_ad_signals(_post("좋아요 최고", [RawComment(text="뒷광고네")]))
    assert r.score > 0

def test_is_ad_threshold():
    assert is_ad(_post("무상 제공받아 작성, 소정의 원고료"), threshold=0.5) is True
    assert is_ad(_post("그냥 후기"), threshold=0.5) is False
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/review/test_ad_signals.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/review/domain/ad_signals.py`:

```python
from pydantic import BaseModel
from raw_review.shared.models import RawPost

class AdSignalScore(BaseModel):
    score: float
    hits: list[str]

_BODY = [("무상 제공", 0.5), ("제공받아", 0.5), ("협찬", 0.5), ("체험단", 0.5),
         ("원고료", 0.5), ("소정의", 0.3), ("쿠폰코드", 0.3), ("할인코드", 0.3),
         ("제휴", 0.2), ("bit.ly", 0.2), ("coupa.ng", 0.4)]
_COMMENT = [("뒷광고", 0.4), ("광고", 0.3), ("협찬", 0.3)]

def score_ad_signals(post: RawPost) -> AdSignalScore:
    score = 0.0
    hits: list[str] = []
    text = post.title + " " + post.body
    for phrase, w in _BODY:
        if phrase in text:
            score += w; hits.append(phrase)
    for c in post.comments:
        for phrase, w in _COMMENT:
            if phrase in c.text:
                score += w; hits.append(f"댓글:{phrase}")
    return AdSignalScore(score=min(score, 1.0), hits=hits)

def is_ad(post: RawPost, *, threshold: float) -> bool:
    return score_ad_signals(post).score >= threshold
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/review/test_ad_signals.py -v` → PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/review/domain/ad_signals.py apps/api/tests/review/test_ad_signals.py
git commit -m "feat: add ad signal rule"
```

---

### Task 4: ReportBuilder

**Files:**
- Create: `apps/api/src/raw_review/review/domain/report_builder.py`
- Test: `apps/api/tests/review/test_report_builder.py`

**Interfaces:**
- Consumes: `PostAnalysis` (shared, 1a Task 20), `ExtractStatus` (shared), `Opinion`/`Polarity` (shared), `SearchSpec` (shared), 리포트 모델 (Task 2)
- Produces:
  - `build_report(spec, analyses: list[PostAnalysis], *, ad_urls: set[str], source_statuses=None, ttl_days=7, max_quotes_per_aspect=5, now=None) -> ProductReport`
  - 규칙: `ad_urls`에 든 글은 집계 제외하고 브랜드별 `ad_rejected`로 카운트. 상태 OK가 아니면 EMPTY/UNANALYZED 카운트. 브랜드별 단점 aspect 묶음(count+대표 인용 최근→추천수 정렬, cap), 장점 태그 count. 브랜드 None은 그대로 보존. 빈 입력이면 brands=[].

- [ ] **Step 1: 실패 테스트**

`tests/review/test_report_builder.py`:

```python
from datetime import datetime, timezone
from raw_review.shared.search_spec import SearchSpec, AttributeConstraint, MatchMode
from raw_review.shared.verdict import Opinion, Polarity
from raw_review.shared.analysis import PostAnalysis
from raw_review.shared.ports import ExtractStatus
from raw_review.review.domain.report_builder import build_report

def _spec():
    return SearchSpec(category="휴대폰케이스", constraints=[
        AttributeConstraint(name="대상기기", hard=True, match_mode=MatchMode.EXACT,
                            wanted=["아이폰17"])])

def _pa(url, opinions, status=ExtractStatus.OK, posted_at=None, likes=0):
    return PostAnalysis(url=url, source="dc", posted_at=posted_at, likes=likes,
                        status=status, opinions=opinions)

def _con(brand, aspect, quote):
    return Opinion(brand=brand, polarity=Polarity.CON, aspect=aspect,
                   quote=quote, source_part="post")

def test_empty_yields_empty_report():
    r = build_report(_spec(), [], ad_urls=set())
    assert r.brands == [] and r.total_analyzed == 0
    assert r.generated_at is not None and r.ttl_days == 7

def test_cons_merge_and_count():
    analyses = [_pa("u1", [_con("다이소","변색","q1")]),
                _pa("u2", [_con("다이소","변색","q2")])]
    r = build_report(_spec(), analyses, ad_urls=set())
    daiso = next(b for b in r.brands if b.brand == "다이소")
    con = next(c for c in daiso.cons if c.aspect == "변색")
    assert con.count == 2 and daiso.analyzed == 2

def test_ad_excluded_and_counted():
    analyses = [_pa("u1", [_con("다이소","변색","q")]),
                _pa("u2", [_con("다이소","변색","q")])]
    r = build_report(_spec(), analyses, ad_urls={"u2"})
    daiso = next(b for b in r.brands if b.brand == "다이소")
    assert daiso.analyzed == 1 and daiso.ad_rejected == 1
    assert r.total_ad_rejected == 1

def test_quotes_sorted_recency_then_likes_capped():
    old = datetime(2026,1,1,tzinfo=timezone.utc)
    new = datetime(2026,10,1,tzinfo=timezone.utc)
    analyses = [_pa("u1", [_con("A","변색","old")], posted_at=old, likes=1),
                _pa("u2", [_con("A","변색","new-hi")], posted_at=new, likes=9),
                _pa("u3", [_con("A","변색","new-lo")], posted_at=new, likes=0)]
    r = build_report(_spec(), analyses, ad_urls=set(), max_quotes_per_aspect=2)
    con = r.brands[0].cons[0]
    assert con.count == 3 and len(con.quotes) == 2
    assert con.quotes[0].text == "new-hi"

def test_status_counts():
    analyses = [_pa("u1", [_con("A","변색","q")]),
                _pa("u2", [], status=ExtractStatus.EMPTY),
                _pa("u3", [], status=ExtractStatus.UNANALYZED)]
    r = build_report(_spec(), analyses, ad_urls=set())
    assert r.total_empty == 1 and r.total_unanalyzed == 1 and r.total_analyzed == 1

def test_brand_none_bucket():
    r = build_report(_spec(), [_pa("u1", [_con(None,"변색","q")])], ad_urls=set())
    assert any(b.brand is None for b in r.brands)
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/review/test_report_builder.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/review/domain/report_builder.py`:

```python
from collections import defaultdict
from datetime import datetime, timezone
from raw_review.shared.search_spec import SearchSpec
from raw_review.shared.verdict import Polarity
from raw_review.shared.analysis import PostAnalysis
from raw_review.shared.ports import ExtractStatus
from raw_review.review.domain.report import (
    ProductReport, BrandSummary, ConSummary, ProTag, Quote, SourceStatus)

_EPOCH = datetime.min.replace(tzinfo=timezone.utc)

def build_report(spec: SearchSpec, analyses: list[PostAnalysis], *,
                 ad_urls: set[str], source_statuses: list[SourceStatus] | None = None,
                 ttl_days: int = 7, max_quotes_per_aspect: int = 5,
                 now: datetime | None = None) -> ProductReport:
    now = now or datetime.now(timezone.utc)
    ok = [a for a in analyses if a.status == ExtractStatus.OK]
    kept = [a for a in ok if a.url not in ad_urls]
    ad = [a for a in ok if a.url in ad_urls]
    total_empty = sum(1 for a in analyses if a.status == ExtractStatus.EMPTY)
    total_unanalyzed = sum(1 for a in analyses if a.status == ExtractStatus.UNANALYZED)

    cons: dict[str | None, dict[str, list[Quote]]] = defaultdict(lambda: defaultdict(list))
    pros: dict[str | None, dict[str, int]] = defaultdict(lambda: defaultdict(int))
    brand_posts: dict[str | None, set[str]] = defaultdict(set)
    ad_rejected: dict[str | None, int] = defaultdict(int)

    for a in kept:
        for op in a.opinions:
            brand_posts[op.brand].add(a.url)
            if op.polarity == Polarity.CON:
                cons[op.brand][op.aspect].append(Quote(
                    text=op.quote, source=a.source, url=a.url,
                    posted_at=a.posted_at, likes=a.likes))
            else:
                pros[op.brand][op.aspect] += 1
    for a in ad:
        for brand in {op.brand for op in a.opinions}:
            ad_rejected[brand] += 1

    brands: list[BrandSummary] = []
    # None(브랜드 미상)이 섞여도 정렬 가능하도록 키 보정
    for brand in sorted(set(cons) | set(pros) | set(ad_rejected),
                        key=lambda b: (b is None, b or "")):
        con_list = []
        for aspect, quotes in cons.get(brand, {}).items():
            quotes.sort(key=lambda q: (q.posted_at or _EPOCH, q.likes), reverse=True)
            con_list.append(ConSummary(aspect=aspect, count=len(quotes),
                                        quotes=quotes[:max_quotes_per_aspect]))
        con_list.sort(key=lambda c: c.count, reverse=True)
        pro_list = sorted((ProTag(aspect=a_, count=c) for a_, c in pros.get(brand, {}).items()),
                          key=lambda p: p.count, reverse=True)
        brands.append(BrandSummary(brand=brand, cons=con_list, pros=pro_list,
                                   analyzed=len(brand_posts.get(brand, set())),
                                   ad_rejected=ad_rejected.get(brand, 0)))

    return ProductReport(
        category=spec.category,
        query_label=" ".join(c.wanted[0] for c in spec.constraints if c.wanted),
        brands=brands,
        total_analyzed=len(kept),
        total_ad_rejected=len(ad),
        total_empty=total_empty,
        total_unanalyzed=total_unanalyzed,
        source_statuses=source_statuses or [],
        generated_at=now, ttl_days=ttl_days)
```

- [ ] **Step 4: 통과** — Run: `cd apps/api && pytest tests/review/test_report_builder.py -v` → PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/raw_review/review/domain/report_builder.py apps/api/tests/review/test_report_builder.py
git commit -m "feat: add report builder with ad exclusion and status counts"
```

---

### Task 5: run_search 오케스트레이션 (합성 루트)

**Files:**
- Create: `apps/api/src/raw_review/app/run_search.py`
- Test: `apps/api/tests/app/test_run_search.py`

**Interfaces:**
- Consumes: `resolve_category` (catalog), `analyze_post` (matching), `build_report`/`is_ad` (review), `AspectNormalizer`, `AmbiguousQuery`, `PostAnalysis`, `Settings`
- Produces:
  - `SearchResult(report: ProductReport | None, ambiguous: AmbiguousQuery | None)`
  - `async run_search(query, posts, *, parser, cache, extractor, normalizer, source_statuses=None, now=None, settings=None) -> SearchResult`
  - 흐름: `resolve_category` → ambiguous면 그대로 반환. 아니면 각 글 `analyze_post` + `is_ad`(ad_urls 수집) → `build_report`.

- [ ] **Step 1: 실패 테스트**

`tests/app/test_run_search.py`:

```python
from datetime import datetime
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import SearchSpec, AttributeConstraint, MatchMode, AmbiguousQuery
from raw_review.shared.mention import ProductMention, MentionRole, ExtractedAttribute, MentionOpinion
from raw_review.shared.verdict import Polarity
from raw_review.shared.ports import ParseResult, ExtractResult, ExtractStatus
from raw_review.adapters.fakes import FakeSpecParser, FakeMentionExtractor
from raw_review.catalog.application.schema_cache import InMemorySchemaCache
from raw_review.matching.domain.aspect_normalizer import AspectNormalizer
from raw_review.config import Settings
from raw_review.app.run_search import run_search

def _spec():
    return SearchSpec(category="양말", constraints=[
        AttributeConstraint(name="소재", hard=True, match_mode=MatchMode.ENUM,
                            wanted=["울"], aliases={"울": ["메리노"]})])

def _post(url, body):
    return RawPost(source="dc", url=url, title="양말 후기", body=body,
                   posted_at=datetime(2026,10,1), likes=1, comments=[])

def _mention():
    return ProductMention(raw="데카트론 울양말 보풀 심함", role=MentionRole.USED, brand="데카트론",
        attributes=[ExtractedAttribute(name="소재", raw_value="메리노", normalized_value="울")],
        opinions=[MentionOpinion(polarity=Polarity.CON, aspect="보풀",
                                 quote="보풀 심함", source_part="post")])

def _deps(extract_by_url):
    return dict(parser=FakeSpecParser(ParseResult(spec=_spec(), confidence=0.9)),
                cache=InMemorySchemaCache(),
                extractor=FakeMentionExtractor(extract_by_url),
                normalizer=AspectNormalizer.default(),
                settings=Settings(ad_score_threshold=0.5) if hasattr(Settings(), "ad_score_threshold") else Settings())

async def test_ambiguous_returns_options():
    parser = FakeSpecParser(ParseResult(spec=None,
        ambiguous=AmbiguousQuery(query="장갑", options=["자전거용","작업용"]), confidence=0.2))
    res = await run_search("장갑", [], parser=parser, cache=InMemorySchemaCache(),
                           extractor=FakeMentionExtractor({}),
                           normalizer=AspectNormalizer.default())
    assert res.report is None and res.ambiguous is not None

async def test_builds_report_from_posts():
    posts = [_post("u1", "데카트론 울양말 보풀 심함")]
    ext = {"u1": ExtractResult(mentions=[_mention()], status=ExtractStatus.OK)}
    res = await run_search("울양말", posts, **_deps(ext))
    assert res.ambiguous is None and res.report is not None
    daiso = res.report.brands[0]
    assert daiso.brand == "데카트론" and daiso.cons[0].aspect == "보풀"

async def test_ad_post_excluded():
    posts = [_post("u1", "데카트론 울양말 보풀 심함, 업체로부터 무상 제공받아 작성")]
    ext = {"u1": ExtractResult(mentions=[_mention()], status=ExtractStatus.OK)}
    res = await run_search("울양말", posts, **_deps(ext))
    assert res.report.total_ad_rejected == 1
    assert res.report.total_analyzed == 0
```

- [ ] **Step 2: 실패 확인** — Run: `cd apps/api && pytest tests/app/test_run_search.py -v` → FAIL

- [ ] **Step 3: 구현**

`src/raw_review/app/run_search.py`:

```python
from datetime import datetime
from pydantic import BaseModel
from raw_review.shared.models import RawPost
from raw_review.shared.search_spec import AmbiguousQuery
from raw_review.config import Settings, get_settings
from raw_review.catalog.application.resolve_category import resolve_category
from raw_review.matching.application.analyze_post import analyze_post
from raw_review.review.domain.ad_signals import is_ad
from raw_review.review.domain.report import ProductReport, SourceStatus
from raw_review.review.domain.report_builder import build_report

class SearchResult(BaseModel):
    report: ProductReport | None = None
    ambiguous: AmbiguousQuery | None = None

async def run_search(query: str, posts: list[RawPost], *, parser, cache, extractor,
                     normalizer, source_statuses: list[SourceStatus] | None = None,
                     now: datetime | None = None, settings: Settings | None = None) -> SearchResult:
    settings = settings or get_settings()
    parsed = await resolve_category(query, parser, cache,
                                    confidence_floor=settings.resolve_confidence_floor)
    if parsed.spec is None:
        return SearchResult(ambiguous=parsed.ambiguous
                            or AmbiguousQuery(query=query, options=[]))

    spec = parsed.spec
    analyses = []
    ad_urls: set[str] = set()
    for post in posts:
        analyses.append(await analyze_post(post, spec, extractor, normalizer))
        if is_ad(post, threshold=settings.ad_score_threshold):
            ad_urls.add(post.url)

    report = build_report(spec, analyses, ad_urls=ad_urls,
                          source_statuses=source_statuses,
                          ttl_days=settings.report_ttl_days, now=now)
    return SearchResult(report=report)
```

> 주: `run_search`가 세 context를 잇는 유일한 지점(합성 루트). 워커(3단계)는 수집한 `posts`와 함께 이 함수를 호출한다. `Settings`에 `ad_score_threshold`, `report_ttl_days` 필드가 필요하다(아래 Step 4).

- [ ] **Step 4: Settings에 필드 추가**

`config.py`의 `Settings`에 추가(1a Task 16에서 만든 파일): `ad_score_threshold: float = 0.5`, `report_ttl_days: int = 7`. 기존 테스트는 영향 없음.

- [ ] **Step 5: 통과 + 경계** — Run: `cd apps/api && pytest tests/app -v && python -m importlinter.cli lint` → PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/raw_review/app apps/api/src/raw_review/config.py apps/api/tests/app/test_run_search.py
git commit -m "feat: add run_search composition root"
```

---

## Self-Review 결과

**1. Spec coverage**
- review 리포트 모델(생성시각/ttl/소스상태/상태카운트): Task 2 ✓
- 광고 판별 룰: Task 3 ✓
- ReportBuilder(브랜드별 단점 인용 정렬/장점 태그/ad 제외·카운트/브랜드 None): Task 4 ✓
- orchestration run_search(질의 해석→글 분석→리포트, ambiguous 반환): Task 5 ✓ (리뷰 1번 해결)
- 경계: review 독립 + app 합성 루트: Task 1 ✓

**2. 리뷰 2차 지적 중 1b 관련**
- 1번 orchestration 끊김 → run_search(합성 루트)로 연결, CategoryResolver→analyze_post→build_report 조립 ✓
- EMPTY/UNANALYZED 분리 집계가 리포트까지 노출(total_empty/total_unanalyzed) ✓

**3. Placeholder scan**: 모든 코드 스텝에 실제 코드. 광고 사전은 1a와 동일 패턴 ✓

**4. Type consistency**: `PostAnalysis`/`ExtractStatus`/`Opinion`/`SearchSpec`/`AmbiguousQuery`가 1a 공유 커널과 동일. `ProductReport`/`SourceStatus`가 Task 2 정의와 소비(Task 4/5) 일치. `run_search`가 반환하는 `SearchResult`는 워커(3단계)가 소비 ✓

**5. 열린 질문**: `Opinion`에 url 미포함(글 메타에서 Quote 생성). 쇼핑 리뷰 별점 기반 저별점 우선은 2단계 수집에서 `PostAnalysis`/정렬에 반영 검토.
